---
title: GCC降低版本的一个实践方案
date: 2021-05-17 14:33:34
tags:
description: LINUX下GCC从高版本（9.3.0）向低版本（4.0.0）安装的一个可行实践方案
---
### 阅读指导

​	本文档将提供LINUX下GCC从高版本（9.3.0）向低版本（4.0.0）安装的一个可行实践方案，本方案使用的技术细节、操作细节、指令细节有一部分为经验操作，有文档、理论支撑的部分我将尽量给出文档来源和理论来源。

​	由于gcc，glibc，binutils和当前host系统 四者之间存在紧密联系，请读者在阅读实践中务必注意四者之间的版本。

​	在默认读者能够使用Ubuntu等Linux常见发行版及常用工具的前提下，读者需要能够：

- 理解gcc，glibc，binutils和系统四者之间的依赖概念和联系；
- 了解基本的编译知识（以理解系统架构、依赖库、链接器的概念和作用）
- 了解make的基本概念，最好能够阅读Makefile（以更好的修改编译过程）
- 了解Linux中环境变量概念
- 了解Linux中库、头文件等“文件系统”的结构与作用

## 零、初始环境与注意事项

- 系统镜像：ubuntu-20.04.2-live-server-amd64
- 虚拟机：VMware Workstation 16 Player
- 编译环境：APT 安装gcc，g++，gcc-multilib，g++-multilib，make，zlib1g-dev
  - gcc 9.3.0
  - g++ 9.3.0
  - make 4.2.1

   - 本方案采用的编译思路是逐大版本向下逐步降低，由于版本相差过大可能存在源码语法改变导致编译失败的情况，因此笔者采用保守的编译思路，来减少这种情况的发生。

 - 读者在编译过程中务必在源码文件夹下新建一个文件夹用于存储编译产生的文件（笔者采用“build”作为文件夹名），例如：

```
gcc-8.1.0
├── ABOUT-NLS
├── build
├── ……
```

 - 在make阶段发生错误或者中止编译，请使用指令`make distclean` 来清理build文件夹，或者全部删除掉，重新执行configure阶段，以减少意料之外的错误发生。
 - 编译安装任何软件，请在configure阶段指定`--prefix`字段来指定安装路径，请勿轻易使用默认的安装路径。笔者采用`/home/shao/build/<$SOFRWARE-NAME>`作为安装路径。
 - 编译时使用`make -j<x>`启用多线程编译可以有效加快速度，此处x最好为编译机器的CPU数量+1，来达到CPU最大负载的目的，如果编译中出错，请关闭该选项重新编译，来定位错误位置。
 - 编译完成后为了启用该版本GCC，请使用任意方式修改PATH环境变量并激活，将`<$PREFIX>/bin`文件夹路径排到PATH首位（不清楚排末尾是否有效），完成后执行`gcc -v`检查是否成功修改。
 - 下文中的make阶段和make install步骤如无必要不再列出，如果读者不理解未列出的两个步骤，请参阅https://gcc.gnu.org/install/

## 一、9.3.0->8.1.0

1. 进入源码文件夹`gcc-8.1.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。

2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

   ```
   ../configure --prefix=/home/shao/build/gcc-8.1.0 \
          --enable-threads=posix --disable-nls \
          --enable-multilib \
          --disable-bootstrap \
          --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
   ```

   - `--prefix`作用见上文
   - `--enable-threads=posix`指定编译使用POSIX线程标准
   - `--disable-nls`关闭对Native Language Support的支持，能够有效减少编译时间（文档来源：https://wiki.osdev.org/Building_GCC）
   - `--enable-multilib`启用multilib库
   - `--disable-bootstrap`关闭bootstrap，也就是不采用自举方式编译，在版本差别小的情况下能够有效减少编译时间，如果出现编译错误请尝试开启该选项（文档来源：https://wiki.osdev.org/Building_GCC）
   - `--with-system-zlib`指定开启zlib库的使用
   - `--enable-languages=c,c++`指定GCC对C和C++的支持，不编译JAVA，FORTRAN等语言模块，能够有效减少编译时间。
   - `--enable-long-long`支持64位长整数
   - `--disable-libsanitizer`关闭libsanitizer的编译，笔者的环境无法编译该库

## 二、8.1.0->7.1.0

1. 进入源码文件夹`gcc-7.1.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。

2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

   ```
   ../configure --prefix=/home/shao/build/gcc-7.1.0 \
          --enable-threads=posix --disable-nls \
          --enable-multilib \
          --disable-bootstrap \
          --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
   ```

3. make过程中出现错误：

   ```
   In file included from ../../../../libgcc/unwind-dw2.c:403:0:
   ./md-unwind-support.h: In function 'x86_fallback_frame_state':
   ./md-unwind-support.h:141:18: error: field 'uc' has incomplete type
     struct ucontext uc;
                     ^~
   make[4]: *** [../../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
   make[4]: Leaving directory '/home/shao/gcc-7.1.0/build/x86_64-pc-linux-gnu/32/libgcc'
   ```

   根据上下文修改该文件：`/home/shao/gcc-7.1.0/build/x86_64-pc-linux-gnu/32/libgcc/./md-unwind-support.h`第141行为`struct ucontext_t uc;`

4. make过程中出现错误：

   ```
   In file included from ../../../libgcc/unwind-dw2.c:403:0:
   ./md-unwind-support.h: In function 'x86_64_fallback_frame_state':
   ./md-unwind-support.h:65:47: error: dereferencing pointer to incomplete type 'struct ucontext'
          sc = (struct sigcontext *) (void *) &uc_->uc_mcontext;
                                                  ^~
   make[2]: *** [../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
   make[2]: Leaving directory '/home/shao/gcc-7.1.0/build/x86_64-pc-linux-gnu/libgcc'
   ```

   根据上下文修改该文件：`/home/shao/gcc-7.1.0/build/x86_64-pc-linux-gnu/libgcc/./md-unwind-support.h`第61行为`struct ucontext_t *uc_ = context->cfa;`

## 三、7.1.0->6.5.0

1. 进入源码文件夹`gcc-6.5.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。

2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

   ```
   ../configure --prefix=/home/shao/build/gcc-6.5.0 \
          --enable-threads=posix --disable-nls \
          --enable-multilib \
          --disable-bootstrap \
          --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
   ```

## 四、6.5.0->6.1.0

1. 进入源码文件夹`gcc-6.1.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。
2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gcc-6.1.0 \
       --enable-threads=posix --disable-nls \
       --enable-multilib \
       --disable-bootstrap \
       --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
```

3. make过程中出现错误：

   ```
   In file included from ../../../../libgcc/unwind-dw2.c:401:0:
   ./md-unwind-support.h: In function 'x86_fallback_frame_state':
   ./md-unwind-support.h:141:18: error: field 'uc' has incomplete type
     struct ucontext uc;
                     ^~
   make[4]: *** [../../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
   make[4]: Leaving directory '/home/shao/gcc-6.1.0/build/x86_64-pc-linux-gnu/32/libgcc'
   ```

   根据上下文修改该文件：`/home/shao/gcc-6.1.0/build/x86_64-pc-linux-gnu/32/libgcc/./md-unwind-support.h`第141行为`struct ucontext_t uc;`

4. make过程中出现错误：

   ```
   In file included from ../../../libgcc/unwind-dw2.c:401:0:
   ./md-unwind-support.h: In function 'x86_64_fallback_frame_state':
   ./md-unwind-support.h:65:47: error: dereferencing pointer to incomplete type 'struct ucontext'
          sc = (struct sigcontext *) (void *) &uc_->uc_mcontext;
                                                  ^~
   make[2]: *** [../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
   make[2]: Leaving directory '/home/shao/gcc-6.1.0/build/x86_64-pc-linux-gnu/libgcc'
   ```

   根据上下文修改该文件：`/home/shao/gcc-6.1.0/build/x86_64-pc-linux-gnu/32/libgcc/./md-unwind-support.h`第61行为`struct ucontext_t *uc_ = context->cfa;`

## 五、6.1.0->5.5.0

1. 进入源码文件夹`gcc-5.5.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。
2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gcc-5.5.0 \
       --enable-threads=posix --disable-nls \
       --enable-multilib \
       --disable-bootstrap \
       --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
```

## 六、5.5.0->5.1.0

1. 进入源码文件夹`gcc-5.1.0`，执行指令`./contrib/download_prerequisites `，使用提供的工具下载依赖软件的源码包。
2. 下载完成后创建build文件夹，进入build文件夹下，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gcc-5.1.0 \
       --enable-threads=posix --disable-nls \
       --enable-multilib \
       --disable-bootstrap \
       --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
```

3. make阶段出现错误：

```
In file included from ../../../../libgcc/unwind-dw2.c:401:0:
./md-unwind-support.h: In function 'x86_fallback_frame_state':
./md-unwind-support.h:141:18: error: field 'uc' has incomplete type
  struct ucontext uc;
                  ^
make[4]: *** [../../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
make[4]: Leaving directory '/home/shao/gcc-5.1.0/build/x86_64-unknown-linux-gnu/32/libgcc'
```

​	根据上下文修改该文件：`/home/shao/gcc-5.1.0/build/x86_64-unknown-linux-gnu/32/libgcc/./md-unwind-support.h`第141行为`struct ucontext_t uc;`

4. make阶段出现错误：

```
In file included from ../../../libgcc/unwind-dw2.c:401:0:
./md-unwind-support.h: In function 'x86_64_fallback_frame_state':
./md-unwind-support.h:65:47: error: dereferencing pointer to incomplete type 'struct ucontext'
       sc = (struct sigcontext *) (void *) &uc_->uc_mcontext;
                                               ^
make[2]: *** [../../../libgcc/shared-object.mk:14: unwind-dw2.o] Error 1
make[2]: Leaving directory '/home/shao/gcc-5.1.0/build/x86_64-unknown-linux-gnu/libgcc'
```

​	根据上下文修改该文件：`/home/shao/gcc-5.1.0/build/x86_64-unknown-linux-gnu/32/libgcc/./md-unwind-support.h`第61行为`struct ucontext_t *uc_ = context->cfa;`

## 七、5.1.0->4.3.0

1. apt安装m4
2. 下载并解压gmp-5.0.2源码，进入源码文件夹，创建build子文件夹并进入，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gmp
```

3. make安装gmp
4. 下载并解压mpfr-3.0.1源码，进入源码文件夹，创建build子文件夹并进入，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/mpfr --with-gmp=/home/shao/build/gmp
```

5. make安装mpfr

6. 在gcc源码文件夹中创建build文件夹，进入build文件夹下，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gcc-4.3.0 \
	   --with-gmp=/home/shao/build/gmp --with-mpfr=/home/shao/build/mpfr \
       --enable-threads=posix --disable-nls \
       --enable-multilib \
       --disable-bootstrap \
       --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
```

​	`--with-gmp`字段指定gmp安装位置，`--with-mpfr`字段指定mpfr安装位置。接下来不急于执行make。

7. make阶段之前修改Makefile文件，然后make

   ```
   CC = gcc
   CXX = g++
   
   改为
   
   CC = gcc -fgnu89-inline
   CXX = g++ -fgnu89-inline
   ```

8. 设置环境变量`export LD_LIBRARY_PATH=/home/shao/build/mpfr/lib`来解决找不到mpfr动态链接库的问题

9. make阶段出现错误：

   ```
   In file included from ../../../../libgcc/../gcc/unwind-dw2.c:338:
   ../../../../libgcc/../gcc/config/i386/linux-unwind.h: In function 'x86_fallback_frame_state':
   ../../../../libgcc/../gcc/config/i386/linux-unwind.h:142: error: field 'info' has incomplete type
   ../../../../libgcc/../gcc/config/i386/linux-unwind.h:143: error: field 'uc' has incomplete type
   make[4]: *** [../../../../libgcc/shared-object.mk:12: unwind-dw2.o] Error 1
   make[4]: Leaving directory '/home/shao/gcc-4.3.0/build/x86_64-unknown-linux-gnu/32/libgcc'
   ```

   根据上下文修改文件`/home/shao/gcc-4.3.0/build/x86_64-unknown-linux-gnu/32/libgcc/../../../../libgcc/../gcc/config/i386/linux-unwind.h`第142行143行为：

   ```
   siginfo_t info;
   struct ucontext_t uc;
   ```

10. make阶段出现错误：

    ```
    In file included from ../../../libgcc/../gcc/unwind-dw2.c:338:
    ../../../libgcc/../gcc/config/i386/linux-unwind.h: In function 'x86_64_fallback_frame_state':
    ../../../libgcc/../gcc/config/i386/linux-unwind.h:58: error: dereferencing pointer to incomplete type
    make[2]: *** [../../../libgcc/shared-object.mk:12: unwind-dw2.o] Error 1
    make[2]: Leaving directory '/home/shao/gcc-4.3.0/build/x86_64-unknown-linux-gnu/libgcc'
    ```

    根据上下文修改文件`/home/shao/gcc-4.3.0/build/x86_64-unknown-linux-gnu/32/libgcc/../../../../libgcc/../gcc/config/i386/linux-unwind.h`第54行为：

    ```
    struct ucontext_t *uc_ = context->cfa;
    ```

11. make阶段出现错误：

    ```
    /usr/bin/ld: cannot find crti.o: No such file or directory
    collect2: ld returned 1 exit status
    ```

    进入/usr/lib文件夹，执行指令：

    ```
    sudo ln -s /usr/lib/x86_64-linux-gnu/crt1.o crt1.o
    sudo ln -s /usr/lib/x86_64-linux-gnu/crti.o crti.o
    sudo ln -s /usr/lib/x86_64-linux-gnu/crtn.o crtn.o
    ```

    来在ld的默认搜索路径中创建crti.o等3个文件的符号链接，避免设置LD_LIBRARY_PATH的冲突问题。

## 八、4.3.0->4.0.0

1. 在gcc源码文件夹中创建build文件夹，进入build文件夹下，执行configure阶段指令：

```
../configure --prefix=/home/shao/build/gcc-4.0.0 \
	   --with-gmp=/home/shao/build/gmp --with-mpfr=/home/shao/build/mpfr \
       --enable-threads=posix --disable-nls \
       --disable-multilib \
       --disable-bootstrap \
       --with-system-zlib --enable-languages=c,c++ --enable-long-long --disable-libsanitizer
```

​	注意此处修改了`--disable-multilib`字段

2. 修改Makefile文件，`LD = ld`修改为`LD = ld -m elf_i386`

3. make阶段出现错误：

```
In file included from ../../gcc/unwind-dw2.c:257:
../../gcc/config/i386/linux-unwind.h: In function 'x86_64_fallback_frame_state':
../../gcc/config/i386/linux-unwind.h:55: error: dereferencing pointer to incomplete type
make[2]: *** [libgcc.mk:519: libgcc/./unwind-dw2.o] Error 1
make[2]: Leaving directory '/home/shao/gcc-4.0.0/build/gcc'
```

​	根据上下文修改文件`/home/shao/gcc-4.0.0/build/gcc/../../gcc/config/i386/linux-unwind.h`第54行为`struct ucontext_t *uc_ = context->cfa;`

4. make阶段出现错误：

```
In file included from ../../gcc/unwind-dw2.c:257:
../../gcc/config/i386/linux-unwind.h: In function 'x86_fallback_frame_state':
../../gcc/config/i386/linux-unwind.h:138: error: field 'info' has incomplete type
../../gcc/config/i386/linux-unwind.h:139: error: field 'uc' has incomplete type
make[2]: *** [libgcc.mk:1126: libgcc/32/unwind-dw2.o] Error 1
make[2]: Leaving directory '/home/shao/gcc-4.0.0/build/gcc'
```

​	根据上下文修改文件`/home/shao/gcc-4.0.0/build/gcc/../../gcc/config/i386/linux-unwind.h`第138行139行为：

```
siginfo_t info;
struct ucontext_t uc;
```

## 九、未完成的工作：4.0.0->?

在4.0.0向下降低的过程中，笔者发现上文中自gcc-7.X经常出现的`error: dereferencing pointer to incomplete type`问题进一步扩大，以至于凭笔者的能力无法解决该问题，考虑到该问题出现的位置位于gcc的异常处理模块，现对该问题的解决方案提供一种推测方案：GCC在编译过程中可能会通过自主产生和复制的方式，从glibc库中获得一些文件，组成编译用的库，该问题的产生可能由于高版本glibc库源码和低版本gcc编译器之间的语法不兼容导致，因此笔者推测降低glibc库是该问题的一种解决方式。由于时间原因和glibc库安装中对环境的潜在污染问题，笔者并未在该方案上做太多尝试，如果读者有能够完美解决glibc库兼容问题的方案，可以求证glibc库版本是否是导致该问题的直接原因。

