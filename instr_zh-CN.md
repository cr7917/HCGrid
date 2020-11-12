# HCGRID 服务器部署说明书

[toc]

***注：若有sudo安装权限，可以根据默认路径将cfitsio和wcslib库安装在默认路径下，步骤一和步骤二针对的是没有sudo权限或因为其它原因需要将库文件安装在自己账号下的用户***



#### 步骤一：安装cfitsio库文件

###### 1.1解压缩cfitsio库文件的压缩包

```sh
tar -xzvf 压缩包名称
压缩包可以使用我们提供的文件，在Dependence目录下
或前往官网下载 https://heasarc.gsfc.nasa.gov/fitsio/
```

###### 1.2给解压缩后的cfitsio文件夹加权限

```sh
chmod 777 解压缩后的cfitsio文件夹名称
```

###### 1.3安装cfitsio库，分为以下四小步

```sh
cd 解压缩后的文件夹名称
```

```sh
./configure --prefix = /home/software/cfistio
这样运行之后，cfitsio库将安装在/home/software/cfitsio文件夹中，您可以根据需要变更安装路径
```

```sh
make
```

```sh
make install
```

执行以上四条命令后，安装成功！

###### 1.4如果您需要验证cfitsio库是否安装成功，可以使用如下安装命令

```sh
make testprog
生成一个名为testprog的可执行文件
```

```sh
./testprog > testprog.lis
```

```sh
diff testprog.lis testprog.out
如果cfitsio正确安装，在命令行输入此命令后将没有返回结果
```

```sh
cmp testprog.fit testprog.std
如果cfitsio正确安装，在命令行输入此命令后将没有返回结果
```



#### 步骤二：安装wcslib库文件

###### 2.1解压缩wcslib库文件的压缩包

```sh
tar -xzvf 压缩包名称
压缩包可以使用我们提供的文件，在Dependence目录下
或前往官网下载 https://www.atnf.csiro.au/people/Mark.Calabretta/WCS/
```

###### 2.2给解压缩后的wcslib文件夹加权限

```sh
chmod 777 解压缩后的wcslib文件夹名称
```

###### 2.3安装wcslib库，分为以下三小步

```sh
cd 解压缩之后的文件夹名称
```

```sh
./configure --prefix = /home/software/wcslib
这样运行之后，wcslib库将安装在/home/software/wcslib文件夹中，您可以根据需要变更安装路径
```

```sh
gmake
```

```sh
gmake install
```

执行以上四条命令后，安装成功！

###### 2.4如果您需要验证wcslib库是否安装成功，可以使用如下安装命令

```
gmake check
此命令来源于wcslib解压缩的文件夹目录下的INSTALL文件，原文如下：
To build and exercise the test suite use
  gmake check
如果执行此命令报错，但是您在执行以上步骤时命令行正常输出，忽略步骤2.4即可
```



#### 步骤三：配置环境变量

这部分需要根据之前wcslib和cfitsio库文件的安装路径进行配置，如果您安装本说明的安装路径进行安装，使用以下语句即可，如果您变更了安装路径，相应的更改即可

需要执行以下三条命令

```sh
export LD_LIBRARY_PATH=/home/software/wcslib/lib:$LD_LIBRARY_PATH
```

```sh
export LD_LIBRARY_PATH=/home/software/cfitsio/lib:$LD_LIBRARY_PATH
```

```sh
export LD_LIBRARY_PATH=/home/software/cfitsio/include:$LD_LIBRARY_PATH
```

###### 注意：

您可以直接在命令行执行此语句，tab键可以补全文件名。但是在命令行执行此语句是暂时性的，重启之后需要重新配置。

如果想要永久添加环境变量，需要更改主目录下的.bashrc文件，手动添加以上三条语句。

```sh
vim .bashrc
更改文件，或使用其它编辑器进行更改操作
在.bashrc文件中添加如下三行并退出保存，即可永久配置环境变量
export LD_LIBRARY_PATH=/home/software/wcslib/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/home/software/cfitsio/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/home/software/cfitsio/include:$LD_LIBRARY_PATH
```



#### 步骤四：更改HCGrid目录下的Makefile文件

###### 4.1首先需要更改INCLUDE，添加HCGRID、WCSLIB、CFITSIO库的路径

在原有记录中添加如下三条

```sh
INCLUDE :=-I/home/HCGRID-master/HCGrid\     
          -I/home/software/cfitsio/include\
          -I/home/software/wcslib/include/wcslib\
```

注意：除以上三条记录需要根据你的路径进行更改外，我们提供的makefile中的INCLUDE也包含cuda和c++安装路径，也需要您根据自己的路径进行更改



###### 4.2其次需要更改LIBRARIES，需要添加如下两条

```sh
LIBRARIES := -L/home/software/cfitsio/lib\
			 -L/home/software/wcslib/lib
```

注意：与上一条相同，我们提供的makefile中的LIBRARIES中包含cuda的路径，需要您根据自己的情况进行更改版本号或路径



###### 4.3其它注意事项

在完成以上操作之后，编译HCGrid仍可能报错，可能仍需做出如下修改：

```sh
CXX := clang++
位于Makefile第二行，需要根据你安装的版本更改
如改成
CXX := clang++-5.0
```

```sh
$(NVCC) -O3 -ccbin gcc-5 $(NVCC_FLAGS) $(CXX_FLAGS) $(INCLUDE) -o $@ -c $<
在Makefile的第30行左右，编译gridding.cu的部分的语句，需要根据你的gcc版本进行更改，如改为
$(NVCC) -O3 -ccbin gcc $(NVCC_FLAGS) $(CXX_FLAGS) $(INCLUDE) -o $@ -c $<
```

同时你需要前往wcslib/include/wcslib-版本号 路径下查找是否拥有路径为wcslib/wcslib.h的文件，如果没有则需要更改我们提供的gmap.cpp文件（在HCGrid目录下），将

```c++
#include <wcslib/wcslib.h>
```

改为

```c++
#include <wcslib.h>
```



#### 步骤五：尝试编译与运行HCGrid

###### 5.1 确认以上操作无误后，您可以前往HCGrid路径下，尝试编译与运行HCGrid程序

```sh
make HCGrid
若报错HCGrid is up to date则将原先的HCGrid可执行文件删除即可
rm HCGrid
```

如果编译成功，则可以尝试运行！若报错，则需要检查以上安装路径是否正确书写！



###### 5.2 编译成功后，可以尝试运行HCGrid

```sh
./HCGrid -h
查看HCGrid程序的参数描述
```

或运行如下语句

```sh
./HCGrid --fits_path /home/HCGrid-master/testdata/ --input_file input --target_file target --output_file output --fits_id 1 --beam_size 300 --order_arg 1 --block_num 64
```

这里的fits_path指的是fits文件的路径，input、target是输入文件名的前缀，并已经存在于该路径下，而output是输出文件的前缀，您可以自己定义，运行HCGrid后也会将输出文件保存在fits_path路径下。
fits_id是fits文件的序号与后缀，beam_size是波束宽度，二者在生成target文件时定义。或者您可以运行Create_target_file.py文件，来更改文件路径、文件序号与波束宽度等参数，命令如下：

```sh
python Creat_target_file.py -p /home/这里的路径您可以自己定义/testdata/ -t target -n 1 -b 300
-n定义序号，-b定义波束宽度，-t定义target map的文件名


注意：确保您的python环境安装了astropy库，若没有则可以使用pip命令安装
pip install astropy
```

运行HCGrid可执行文件后，若命令行没有返回错误信息，则表示您已完成HCGrid环境的配置!





