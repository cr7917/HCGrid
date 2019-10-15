# HCGrid
- *Version:* 1.0.0
- *Authors:*

## Introduction
Gridding refers to map the non-uniform sampled data to a uniform grid, which is one of the key steps in the data processing pipeline of the radio telescope. The computation performance is the main bottleneck of gridding, which affects the whole performance of the radio data reduction pipeline and even determines the release speed of available astronomical scientific data. For the single-dish radio telescope, the representative solution for this problem is implemented on the multi-CPU platform and that mainly based on the convolution gridding algorithm. Such methods can achieve a good performance through parallel threads on multi-CPU architecture, however, its performance is still limited to a certain extent by the homogeneous multi-core hardware architecture.
HCGrid is a convolution-based gridding framework for radio astronomy on CPU/GPU heterogeneous platform, which can efficiently resample raw non-uniform sample data into a uniform grid of arbitrary resolution, generate DataCube and save the processing result as FITS file. 

## implementation
<P align="center"><img src=pic/HCGrid.png width="50%"></img></p>
The main work of HCGrid include three-part:

- **Initialization module:** This module mainly initializes some parameters involved in the calculation process, such as setting the size of the sampling space, output resolution and other parameters.
- **Gridding module:** The core functional modules of the HCGrid. The key to improving gridding performance is to increase the speed of convolution calculations. First, in order to reduce the search space of the original sampling points, we use a parallel ordering algorithm to preorder the sampling points based on HEALPix on the CPU platform and propose an efficient two-level lookup table to speed up the acquisition of sampling points. Then, accelerating convolution by utilizing the high parallelism of GPU and through related performance optimization strategies based on CUDA architecture to further improve the gridding performance.
- **Results Process module:** Responsible for verifying the accuracy of gridding results; Save processing results as FITS files and visualize the computing results.

## Features
- Supports WCS projection system as target.
- High performance.
- Scales well on CPU/GPU heterogeneous platforms.

## Build
- The Makefile is very simple, you can easily adapt it to any Unix-like OS.
- Change the path of CUDA to match your server, in general, the default installation path for CUDA is /usr/local/cuda-xx.
- If using profiling, it is necessary to use the compile option --ptxas-options=-v to view all static resources identified by the kernel during the compilation phase, such as register resources, shared memory resources, etc.

## Dependencies
We kept the dependencies as minimal as possible. The following packages are required:
- cfitsio-3.47
- wcslib-5.16

 All of these packages can be found in "Dependencies" directory (order versions of these libraries may work, but we didn't test these!).

## Usage
### Minimal example
Using HCGrid is extremely simple. Just define a FITS header(with valid WCS), define gridding kernel, pre-sorting sample points and run the gridding function.
1. Define a FITS header and define gridding kernel:
``` python
/*Read input points*/
lon,lat,data = read_input_map(...);

/*define target FITS/WCS header*/
header = {
	'NAXIS': 3,
	'NAXIS1': dnaxis1,
	'NAXIS2': dnaxis2,
	'NAXIS3': 1,  
	'CTYPE1': 'RA---SIN',
	'CTYPE2': 'DEC--SIN',
	'CUNIT1': 'deg',
	'CUNIT2': 'deg',
	'CDELT1': -pixsize,
	'CDELT2': pixsize,
	'CRPIX1': dnaxis1 / 2.,
	'CRPIX2': dnaxis2 / 2.,
	'CRVAL1': mapcenter[0],
	'CRVAL2': mapcenter[1],
	}

/*Set kernel*/
kernel_type = GAUSS1D;
kernelsize_fwhm = 300./3600.;
kernelsize_sigma = 0.2;
kernel_params[0] = kernelsize_sigma;
sphere_radius = 3.*kernelsize_sigma;
hpx_max_resolution=kernelsize_sigma/2;
_prepare_grid_kernel(
	kernel_type, 
	kernel_params, 
	sphere_radius, 
	hpx_max_resolution
	);
```
2. Select a suitable pre-sorting interface to pre-sort sampling points. Our program provides a variety of pre-sorting interfaces to preorder the sampleing points. shch as "BLOCK_INDIRECT_SORT", "PARALLEL_STABLE_SORT", etc. Through a series of experiments, we demonstrated that the BlockIndirectSort based on CPU multi-thread could achieve the best performance when dealing with large-scale data. So, we set BlockIndirectSort as the default pre-sorting interface in our program.
``` C++
/* 
 * func: Sort input points with CPU
 * sort_param:
 * - BLOCK_INDIRECT_SORT
 * - PARALLEL_STABLE_SORT
 * - STL_SORT
 *   */
void init_input_with_cpu(const int &sort_param) {
    double iTime1 = cpuSecond();
    uint32_t data_shape = h_GMaps.data_shape;
    std::vector<HPX_IDX> V(data_shape);
    V.reserve(data_shape);

    // Sort input points by param
    double iTime2 = cpuSecond();
    if (sort_param == BLOCK_INDIRECT_SORT) {
        boost::sort::block_indirect_sort(V.begin(), V.end());
    } else if (sort_param == PARALLEL_STABLE_SORT) {
        boost::sort::parallel_stable_sort(V.begin(), V.end());
    } else if (sort_param == STL_SORT) {
        std::sort(V.begin(), V.end());
    }
}

/* 
 * func: Sort input points with Thrust
 * sort_param:
 * - THRUST
 * */
void init_input_with_thrust(const int &sort_param) {
    double iTime1 = cpuSecond();
    uint32_t data_shape = h_GMaps.data_shape;

    // Sort input points by param
    double iTime2 = cpuSecond();
    if (sort_param == THRUST) {
        thrust::sort_by_key(h_hpx_idx, h_hpx_idx + data_shape, in_inx);
    }
}
```
3. Do the gridding.
``` C++
/* Gridding process. */
void solve_gridding(const char *infile, const char *tarfile, const char *outfile, const char *sortfile, 
                    const int& param, const int &bDim) {

    // Read input points.
    read_input_map(infile);

    // Read output map.
    read_output_map(tarfile);

    // Set wcs for output pixels.
    set_WCS();

    // Initialize output spectrals and weights.
    init_output();

    // Block Indirect Sort input points by their healpix indexes.
    if (param == THRUST) {
        init_input_with_thrust(param);
    } else {
        init_input_with_cpu(param);
    }

    // Alloc data for GPU.
    data_alloc();

    // Send data from CPU to GPU.
    data_h2d();
    printf("h_zyx[1]=%d, h_zyx[2]=%d, ", h_zyx[1], h_zyx[2]);

    // Set block and thread.
    dim3 block(bDim);
    dim3 grid((h_GMaps.block_warp_num * h_zyx[1] - 1) / (block.x / 32) + 1);
    printf("grid.x=%d, block.x=%d, ", grid.x, block.x);

    // Call device kernel.
    hcgrid<<< grid, block >>>(d_lons, d_lats, d_data, d_weights, d_xwcs, d_ywcs, d_datacube, d_weightscube, d_hpx_idx);

    // Send data from GPU to CPU
    data_d2h();

    // Write output FITS file
    write_output_map(outfile);

    // Write sorted input FITS file
    if (sortfile) {
        write_ordered_map(infile, sortfile);
    }

    // Release data
    data_free();
}
```