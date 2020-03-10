# 使用cmake编译ceph模块

> ceph大概从jewel版开始使用cmake进行ceph项目的编译管理。
> 但是基本网上没有关于ceph cmake的使用介绍，对于小白我来说，因为对cmake还有以前的autoconfig不了解，甚至对Makefile的编写规则都不太懂，只懂得用简单的gcc/g++命令进行编译。但是在修改/调试ceph代码时，由于src代码目录结构众多，库文件的引用关联太多，使用gcc基本上很难编译成功。
> 经过一段时间的Makefile和cmkae命令的学习摸索，基本上了解如何用cmake编译ceph了。谨写此文备忘，并分享给其它小白。


# Makefile
> 这里有篇很详尽的Makefile规则教程：[Linux Makefile详细教程](https://blog.csdn.net/liang13664759/article/details/1771246)
> 关于Makefile的编写规则不在本文章介绍范围内，Makefile的介绍网上一搜一大堆，请自行百度/google。

在刚开始学习C/C++的时候，还记得使用的是visual-6.0。确实很方便，写完程序后点击一下运行就能自动跑起来。虽然使用IDE工具可以让新手暂时省略各种编译链接过程，将注意力集中在程序语言本身。但是长久以往，就会离不开IDE。一旦没有了IDE环境，就感觉不会写代码一般，特别是从Windows平台切换到Linux平台时，感受特别深刻。
在Linux中，简单的程序可以使用gcc编译器直接进行编译，但是对于大项目来说，Makefile就是主要的编译管理工具。Makefile的主要作用是告诉make命令如何对项目程序进行编译链接，并生成可执行文件的过程。
原则上，编写完Makefile后，在shell上执行`make`命令后，就会自动在本目录下查找Makefile文件并进行编译链接。

# cmake
> 这里有篇很简单的cmake入门博客：[如何编写CMakeList.txt](https://www.cnblogs.com/cv-pr/p/6206921.html)
> cmake同样地网上介绍一大堆，希望了解的自行百度/google

cmake是跨平台的编译管理工具。主要作用其实就是根据规则自动生成Makefile，然后使用make命令进行编译链接。所以使用cmake需要如下步骤：
1. 编写CMakeList.txt文件
2. 生成Makefile，`mkdir build && cd build && cmake ../CMakeList.txt`。因为cmake生成的目录文件很多，一般情况下都会创建一个build目录进行放置。
3. 执行`make` 命令进行项目编译链接。

cmake具体上比automake、autoconfig还有其它编译管理工具在具体上有什么优势，因为我之前没使用过autoconfig或者automake，所以不太懂。只知道cmake在跨平台、大工程上的管理优势明显。但是cmake定义为高级编译配置工具，功能丰富、兼容性好、操作简单，知道这些感觉就够了。同时通过ceph编译的使用体验上来说，cmake还是比autoconfig简单多多了。

cmake生成的目录结果大致如下所示：
```txt
├── build
│   ├── CMakeCache.txt
│   ├── CMakeFiles
│   │   ├── 3.2.2
│   │   │   ├── CMakeCCompiler.cmake
│   │   │   ├── CMakeCXXCompiler.cmake
│   │   │   ├── CMakeDetermineCompilerABI_C.bin
│   │   │   ├── CMakeDetermineCompilerABI_CXX.bin
│   │   │   ├── CMakeSystem.cmake
│   │   │   ├── CompilerIdC
│   │   │   │   ├── a.out
│   │   │   │   └── CMakeCCompilerId.c
│   │   │   └── CompilerIdCXX
│   │   │       ├── a.out
│   │   │       └── CMakeCXXCompilerId.cpp
│   │   ├── cmake.check_cache
│   │   ├── CMakeDirectoryInformation.cmake
│   │   ├── CMakeOutput.log
│   │   ├── CMakeTmp
│   │   ├── feature_tests.bin
│   │   ├── feature_tests.c
│   │   ├── feature_tests.cxx
│   │   ├── Makefile2
│   │   ├── Makefile.cmake
│   │   ├── progress.marks
│   │   ├── TargetDirectories.txt
│   │   └── test_sqrt.dir
│   │       ├── build.make
│   │       ├── C.includecache
│   │       ├── cmake_clean.cmake
│   │       ├── DependInfo.cmake
│   │       ├── depend.internal
│   │       ├── depend.make
│   │       ├── flags.make
│   │       ├── link.txt
│   │       ├── progress.make
│   │       └── src
│   │           ├── b.c.o
│   │           └── main.c.o
│   ├── cmake_install.cmake
│   ├── Makefile
│   └── test_sqrt
├── CMakeLists.txt
├── include
│   └── b.h
└── src
    ├── b.c
    └── main.c
```

# 使用cmake编译ceph
终于到正题了。之前学习ceph的时候，想修改代码，比如增加些打印语句等等，这还好办，直接从新make对应的模块就好了。比如修改了rbd模块相关文件，那就执行`make rbd`就行。但是如果想添加个文件，模块或者工具tool时，就不太好办了。因为Makefile中并没有相关文件的信息，也就是说即使执行`make`，新加的模块也不会进行编译。
但是自从有了cmake后就好办多了。可以直接在CMakeList.txt中把新写的模块文件添加进去，然后make就好。因为是基于ceph整个工程的，所以头文件include路径依照ceph/src为工程根目录导入即可。同时，也不太需要关心global链接依赖的问题，直接模仿一下ceph的写法就好。

先说明一下ceph CMakeList.txt的结构，如下所示。在工程根目录下有一个总的CMakeList.txt文件，然后使用`add_subdirectory`语句层层递进，从而找到所有的CMakeList的配置。因此，我们在更改ceph源码时，只需要修改该模块下的CMakeList.txt即可。
``` txt
├── ceph-root
│   ├── build
│   │   ├── ...
│   ├── CMakeLists.txt
│   ├── src
│   │   ├── CMakeLists.txt
│   │   ├── tools
│   │   │   ├── CMakeLists.txt
│   │   ├── osd
│   │   │   ├── CMakeLists.txt
│   │   ├── mon
│   │   │   ├── CMakeLists.txt
│   │   │...
```

比如，我们想增加一个命令行工具，在 src/tools/ 目录下，新建了一个目录，嗯，叫demo。这个工程什么都不做，就使用了boost-pragram_option进行命令行的解析，然后打印参数值。其程序如下：
```cpp
#include <boost/program_options.hpp>
#include <iostream>

using namespace std;
namespace po = boost::program_options;

int main(int args, char** argv){

  try {

    po::options_description generic("Generic Options: ");
    generic.add_options()
      ("help", "help message")
      ("version,v", "print procedure version");

    int opt = 5;
    po::options_description ha("haha Options: ");
    ha.add_options()
      ("opt", po::value<int>(&opt)->default_value(10), "optimize value");

    po::options_description cmdline_options;
    cmdline_options.add(generic).add(ha);

    po::variables_map vm;
    po::store(po::parse_command_line(args, argv, cmdline_options), vm);
    po::notify(vm);

    if (vm.count("help")){
      cout << cmdline_options << endl;
      return 1;
    }
    cout << "vm opt = " << vm["opt"].as<int>() << endl;
    cout << "no opt = " << opt << endl;
  }
  catch(exception& e) {
  }
  catch(...) {
  }
  cout << "haha!" << endl;
  return 0;
```

然后再demo/ 目录下，新建一个CMakeList.txt文件，只需要在文件上写上这么几句话：
``` cmake
set(demo test.cc)
add_executable(demo ${demo})
target_link_libraries(demo global
  ${BLKID_LIBRARIES} ${CMAKE_DL_LIBS})
```

然后在上层目录的CMakeList.txt中标识demo这个文件。
```cmake
...
add_subdirectory(demo)
...
```

然后，返回root/build目录，执行`cmake ..`，会重新生成一个Makefile，然后执行`make demo`，就能够找到 build/bin/demo 这个可执行文件，执行就会输出：
```txt
@ ./bin/demo --help
Generic Options: :
  --help                 help message
  -v [ --version ]       print procedure version

haha Options: :
  --opt arg (=10)        optimize value
```

按照这个例子，可以引用librados、librbd等库或者函数，然后就可以依样画葫芦，编译出自己的tool了。