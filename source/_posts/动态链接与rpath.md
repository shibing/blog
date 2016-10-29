---
title: 动态链接与rpath
date: 2016-08-20 12:57:50
tags:
    - gcc
    - linux
---

最近在工作中做`php`环境的绿色安装，所谓绿色安装是指将程序以及它的所有依赖包括一些共享库、配置文件等打包到一起然后进行安装的方式。因为所有的依赖都打包到了一起，那么部署的过程其实就是简单的拷贝过程，非常的方便。但是这之中遇到了一个问题，就是如何让`php`运行的时候加载我们绿色包里面的共享库，而不是加载系统自身的共享库。为此学习了`linux`下运行时链接器器搜索共享库的方式，发现通过指定`RPATH`可以很方便的解决这个问题。

# `RPATH`
linux下在链接共享库的时候可以通过`rpath`选项来指定**运行时**共享库加载路径。 通过这个选项指定的路径会写到ELF文件`dynamic`段的`RPATH`里, 运行时链接器会在此路径下搜索ELF文件所依赖的共享库。

# 示例

假设我们有一个共享库`libarith.so`，提供了常见的算术运算，它由`arith.h`与`arith.c`两个文件编译生成。内容如下：

`arith.h`:
```c
#pragma once
int add(int a, int b);
```

`arith.c`:
```c
#include "arith.h"
int add(int a, int b)
{
    return a + b;
}
```

生成`so`文件
```
$ gcc -fPIC -shared arith.c -o libarith.so
```

然后我们有一个`main.c`文件需要调用前面生成的共享库，内容如下：

```c
#include "arith.h"
int main()
{
    add(1, 2);
}
```

如果我们不指定`rpath`直接编译`main.c`：
```
$ gcc -L. -larith main.c -o main
```
编译后目录内容如下：
```
.
├── arith.c
├── arith.h
├── libarith.so
├── main
└── main.c
```

若此时运行`main`文件会报如下错误：
```
./main: error while loading shared libraries: libarith.so: cannot open shared object file: No such file or directory
```

报错提示找不到`libarith.so`文件。该文件在我们当前目录下，但是当前目录并不在运行时链接器的搜索路径中。一种解决办法是将当前路径添加到`LD_LIBRARY_PATH`中，但是该方法是一种全局配置，总是显得不那么干净。下面介绍第二种方法，就是在链接的时候直接将搜索路径写到RPATH中，按如下方式重新编译：
```
$ gcc -L. -larith main.c -Wl,-rpath='.' -o main
```

`-rpath`是链接器选项，并不是`gcc`的编译选项，所以上面通过`-Wl,`告知编译器将此选项传给下一阶段的链接器。重新编译后，采用`readelf`命令查看`main`文件的`dynamic`节，发现多了一个`RPATH`字段，且值就是我们前面设置的路径。 
```
$ readelf -d main| grep PATH
  0x000000000000000f (RPATH)              Library rpath: [.]
```

再次尝试运行`main`文件会发现一切正常。


# `$ORIGIN`
上面的解决办法还有一些小问题，`RPATH`指定的路径是相当于当前目录的，而不是相对于可执行文件所在的目录，那么当换一个目录再执行上面的程序，就会又报找不到共享库。解决这个问题的办法就是使用`$ORIGIN`变量，在运行的时候，链接器会将该变量的值用可执行文件所在的目录来替换，这样我们就又能相对于可执行文件来指定`RPATH`了。重新编译如下：
```
$ gcc -L. -larith main.c -Wl,-rpath='$ORIGIN/' -o main
```

# `patchelf命令`
很多时候我们拿到的是编译好的二进制文件，这样我们就不能用前面的办法来指定`RPATH`了。幸好有`patchelf`这个小工具，它可以用来修改`elf`文件，用它修改`main`的`RPATH`的方法如下：

```
$ pathelf  main --set-rpath='$ORIGIN/'
```

# 总结
上面介绍了`RPATH`的作用，如何在编译的时候通过相关参数指定`RPATH`，以及使用`patchelf`修改`RPATH`的方法。
