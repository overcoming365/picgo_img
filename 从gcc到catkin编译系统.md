# 从gcc到catkin编译系统

## 概述

​	在ROS开发使用过程中，调试代码往往占用的时间长度要远大于编写代码的时间长度。掌握编程语言的基础知识和编译原理，对于学习和使用ROS往往能达到事倍功半的效果。Linux系统十分依赖文件的内容，若无目的的修改文件，往往会导致各种问题，很难解决。

​	编译系统是ROS的一个关键组成部分，从ROS1初期使用的rosbuild，到ROS1的groovy版本之后的catkin，虽然编译系统并不是ROS框架中的核心部分，但却是开发者最常接触的一个重要功能，了解其基本使用方法，常常可以发挥事半功倍的效果。

​	ROS是一种**分布式**的框架，软件以**相互独立的软件包package**的形式存在，用起来确实方便。但是对于编译系统来讲，编译的难度很大，它不仅需要正确、快速地完成**自动化构建**过程，还需要解决每个软件包之间的**相互依赖关系**，即依赖包又依赖包的版本，这同时也是Ubuntu等类Unix系统中经常要处理的问题。虽然ROS中的编译系统一直采用标准的构建工具，比如CMake、Python setuptools，但是这些工具并没有办法完全满足ROS的需求，都需要添加一些额外的功能。

## 编译系统演进全景图

```
编译系统发展历程：从手动到自动化，从单一到分布式

┌─────────────────────────────────────────────────────────────────────────────┐
│                          编译复杂度与项目规模                                    │
│                                 ↑                                          │
│                                 │                                          │
│    复杂                         │                                          │
│                                 │                                          │
│         ┌─────────────────────┐ │  ROS分布式系统                             │
│         │     Catkin          │ │  • 多包依赖管理                             │
│         │   (ROS专用构建)      │ │  • 消息/服务生成                           │
│         │                     │ │  • 工作空间统一编译                         │
│         └─────────────────────┘ │                                          │
│                ↑                │                                          │
│         ┌─────────────────────┐ │  跨平台大型项目                             │
│         │      CMake          │ │  • IDE集成                                │
│         │   (跨平台构建)       │ │  • 依赖自动查找                            │
│         │                     │ │  • 多平台支持                              │
│         └─────────────────────┘ │                                          │
│                ↑                │                                          │
│         ┌─────────────────────┐ │  中型项目                                  │
│         │     Makefile        │ │  • 增量编译                               │
│         │   (构建自动化)       │ │  • 依赖管理                               │
│         │                     │ │  • 脚本化构建                              │
│         └─────────────────────┘ │                                          │
│                ↑                │                                          │
│         ┌─────────────────────┐ │  小型项目                                  │
│    简单   │       g++           │ │  • 直接编译                               │
│         │    (手动编译)        │ │  • 完全控制                               │
│         │                     │ │  • 学习验证                               │
│         └─────────────────────┘ │                                          │
│                                │────────────────────────────────────────→   │
│                                      项目规模（文件数量）                     │
│                              1-5个    10-50个   100+个    分布式系统         │
└─────────────────────────────────────────────────────────────────────────────┘

时间轴：1970s────────1980s────────2000s────────2010s────────现在
        g++       Makefile      CMake        Catkin       现代构建

每个阶段解决的核心问题：

g++阶段：
• 问题：重复输入编译命令
• 解决：直接控制编译过程
• 适用：单文件或简单项目

Makefile阶段：
• 问题：大量文件的重复编译
• 解决：增量编译和依赖管理
• 适用：传统C/C++项目

CMake阶段：
• 问题：跨平台编译复杂
• 解决：生成平台特定构建文件
• 适用：现代C++项目

Catkin阶段：
• 问题：ROS生态系统集成复杂
• 解决：ROS专用构建和包管理
• 适用：机器人系统开发
```

## 编译执行与解释执行

- **编译执行(C/C++）**

  ​	编译执行就是使用编译器将代码翻译成计算机可直接运行的二进制文件，执行的时候就不需要再次翻译，直接运行。C/C++的代码需要编译器进行编译。

  ​	编译型语言，**执行速度快、效率高；依赖编译器、跨平台性差。**

- **解释执行（Python、Java）**

  ​	Python是典型的解释型语言，在运行前不需要编译器编译，不需要预处理，运行时由解释器一句句解释运行即可。但是在ROS2里，Python编写的文件要进行预处理，放置到运行路径下面。因此，也需要使用编译指令colconbuild。但本质上，这个操作并不是编译。

  ​	解释型语言，**执行速度慢、效率低；依赖解释器、跨平台性好。**

## gcc与g++编译器

> ​	所谓**编译器**，可以简单地将其理解为“**翻译器**”。要知道，计算机只认识**二进制指令**（仅有0和1组成的指令），我们日常编写的C语言代码、C++代码、Go代码等，计算机根本无法识别，只有将程序中的每条语句翻译成对应的二进制指令，计算机才能执行。	

​	早期gcc的全拼为**GNU C Compiler**，即GUN计划诞生的C语言编译器，显然最初gcc的定位确实只用于编译C语言。但经过这些年不断的迭代，gcc的功能得到了很大的扩展，它不仅可以用来编译C语言程序，还可以处理C++、Go、Objective-C等多种编译语言编写的程序。
与此同时，由于之前的GNU C Compiler已经无法完美诠释gcc的含义，所以其英文全称被重新定义为**GNU Compiler Collection**，即GNU编译器套件，都是GCC但是含义不一样了。

​	**g++和gcc**都是由GNU编译器集合（GNU-CompilerCollection，简称GCC）提供编译器，分别用于编译和链接C++和C代码。g++在链接时除了关注C语言标准库外，还会链接C++标准库，如libstdc++。g++命令可以用于将C++源文件编译成可执行文件，或者生成对象文件和库文件。了解g++编译工具，对于理解CMake的编译过程十分重要。

**gcc和g++区别**：

- gcc与g++都可以编译c代码与c++代码。但是，后缀为.c的，gcc把它当做C程序，而g++当做是C++程序；后缀为.cpp的，两者都会认为是C++程序。
- 编译阶段，g++会调用gcc，对于c++代码，两者是等价的，但是因为gcc命令不能自动和C++程序使用的库链接，所以通常用g++来完成链接。
- 编译可以用gcc/g++，而链接可以用g++或者gcc -lstdc++。因为gcc命令不能自动和C++程序使用的库链接，所以通常使用g++来完成链接。但在编译阶段，g++会自动调用gcc，二者等价。

## 代码编译过程

C/C++的编译过程包含了四个步骤：
1.预处理(Preprocessing)
2.编译(Compilation)
3.汇编(Assemble)
4..链接(Linking)

​	我们首先创建show.cpp用于实现简单打印输出的功能，show.h头文件进行外部调用声明，main.cpp去调用我们定义show.h中的函数：

**main.cpp：**

```
#include "show.h"
#include <iostream>    
#include <string>     

int main(int argc, char* argv[]) {

    std::string exe_path = argv[0];  
    show();
    std::cout << "result from: " << exe_path << std::endl;

}
```

**show.h：**

```
#ifndef __SHOW_H_
#define __SHOW_H_

void show();

#endif
```

**show.cpp：**

```
#include "show.h"
#include <iostream>

void show()
{
    std::cout<<"hello micu"<<std::endl;
}
```

### 预处理

​	**预处理阶段**主要处理一些**预处理指令**，比如文件包括、宏定义、条件编译。文件包括就是将所有的**#include头文件**替换成真正的内容。经过预处理，会产生一个没有宏定义，没有条件编译指令，没有特殊符号的输出文件，这个文件的含义同原本的文件无异，只是内容上有所不同。

​	读取C/C++源程序，对其中的伪指令（以#开头的指令）进行处理，内容如下：

1. 将所有的"#define"删除，并且展开所有的宏定义；

2. 处理所有的条件编译指令，如："#if"、"#ifdef"、"#elif"、"#else"、"#endif"等。这些伪指令的引入使得程序员可以通过定义不同的宏来决定编译程序对哪些代码进行处理。预编译程序将根据有关的文件，将那些不必要的代码过滤掉；

3. 处理"#include"预编译指令，将被包含的文件插入到该预编译指令的位置（注意：这个过程可能是递归进行的，也就是说被包含的文件可能还包含其他文件）；

4. 删除所有的注释；

5. 添加行号和文件名标识，以便于编译时编译器产生调试用的行号信息及用于编译时产生的编译错误或警告时能够显示行号；

6. 保留所有的#pragma编译器指令。

**终端命令实现：**

```bash
g++ -E 源文件名称.cpp -o 源文件名称.ii
```

要打印预处理阶段的结果，请向g++传递-E选项。

### 编译

​	**编译阶段**进行**语法分析、词法分析和语义分析**，并且将代码优化后产生相应的**汇编代码文件（ASCII文件）**，即.s文件。这个过程是整个程序构建的核心部分，也是最复杂的部分之一。在这个阶段，预处理过的代码被翻译成目标处理器架构特有的汇编指令。这些形成了一种中间的人类可读语言。

**终端命令实现：**

```bash
g++ -S 源文件名称.ii -o 源文件名称.s
```

要保存编译阶段的结果，可以向g++传递-S选项。

### 汇编

​	**汇编**即通过不同平台（Windows、Linux）的汇编器将**编译完的汇编代码文件**翻译成**机器码**，即生成**二进制可重定向文件**（.o）。每一个汇编语句几乎都对应一条机器指令。所以汇编器的汇编过程相对于编译器来讲比较简单，它没有复杂的语法，也没有语义，也不需要做指令优化，只是根据汇编指令和机器指令的对照表一一**翻译**即可。任何一个源文件在进行编译阶段的时候会去产生**符号表**，符号表中存放的就是程序所产生的符号（例如：函数名，变量名等），我们的编译阶段是不会去给符号分配正确的地址。这些符号都没有被分配地址，因此.o文件没有经过链接是**无法执行**的。

**终端命令实现：**

```bash
g++ -c 源文件名称.s -o 源文件名称.o
```

要保存汇编阶段的结果，请向gcc传递-c选项。这个.o文件的内容是二进制格式，可以用运行命令**hexdump**或**od -c**来检查。

### 链接

​	**链接**是通过链接器将一个个目标文件或库文件链接在一起生成一个**完整的可执行程序**。由汇编程序生成的目标文件并不能立即就被执行，其中可能还有许多没有解决的问题。

​	例如，某个源文件中的函数可能引用了另一个源文件中定义的某个符号（如变量或者函数调用等）；在程序中可能调用了某个库文件中的函数，等等。所有的这些问题，都需要经链接程序的处理方能得以解决。链接程序的主要工作就是将**有关的目标文件彼此相连接**，也就是将在一个文件中引用的符号同该符号在另外一个文件中的定义连接起来，使得所有的这些目标文件成为一个能够被操作系统装入执行的**统一整体**。

**程序的链接阶段可分为两个步骤：**

1. 由于每个.o文件都有都有自己的代码段、bss段、堆、栈等，所以链接器首先将多个.o文件相应的段进行合并，建立映射关系并且去**合并符号表**。进行符号解析，符号解析完成后就是给符号**分配虚拟地址**。
2. 将分配好的虚拟地址与符号表中的定义的符号一一对应起来，使其成为正确的地址，使代码段的指令可以根据符号的地址执行相应的操作，最后由链接器生成可执行文件。

**终端命令实现：**

```bash
g++ 源文件名称.o -o 可执行文件名称
```

> **注意：**以上四步就是代码编译的过程，这些可以合并成一个命令来完成编译，生成可执行文件
>
> ```bash
> g++ 源文件名称.cpp -o 可执行文件名称
> ```
>
> 最后在终端输入：
>
> ```bash
> ./可执行文件名称
> ```
>
> 就可以执行代码。



## 库

​	库是写好的，现有的，成熟的，可以复用的代码。现实中每个程序都要依赖很多基础的底层库，不可能每个人的代码都从零开始，因此库的存在意义非同寻常。本质上来说，库是一种可执行代码的二进制形式，可以被操作系统载入内存执行。库有两种：静态库（.a、.lib）和动态库（.so、.dll）。

### 静态库

#### 特点

​	之所以称为【静态库】，是因为在链接阶段，会将汇编生成的目标文件.o与引用到的库一起链接打包到可执行文件中。因此对应的链接方式称为**静态链接**。
​	试想一下，静态库与汇编生成的目标文件一起链接为可执行文件，那么静态库必定跟.o文件格式相似。其实一个静态库可以简单看成是一组目标文件（.o/.obj文件）的集合，即很多目标文件经过压缩打包后形成的一个文件。静态库特点总结如下：

- 静态库对函数库的链接是放在**编译时期**完成的。
- 程序在**运行时与函数库无关**，移植方便。
- **浪费空间和资源**，因为所有相关的目标文件与牵涉到的函数库被链接合成一个可执行文件。

#### 创建与使用

首先编译成可重定位目标文件（.o)，但未进行链接。

```
g++ -c 源文件名称.cpp -o 源文件名称.o
```

然后将目标文件（.o）打包成静态库文件（.a)

```
ar -crv lib静态库名称.a 文件名称.o
```

最后是**链接我们创建的静态库**，顺序不能变

```
g++ 源文件名称.cpp -L指定路径 -l 静态库的名称 -o 可执行文件名称
```

```
 g++ main.cpp -L . -lmyshow -o end1
```

- **`-L.`**：指定静态库的搜索目录为**当前目录**（`.`）。 作用：让链接器在当前目录下寻找静态库`libmyshow.a`。
- **`-lmyshow`**：链接**静态库`libmyshow.a`**（`-l`后面跟库名，省略`lib`前缀和`.a`后缀）。 作用：告诉链接器“需要从`libmyshow.a`中获取`show()`函数的实现”。
- **`-o output1`**：指定输出的可执行文件名为`output1`。

### 动态库

#### 特点

为什么还需要动态库？其实也就是静态库的缺点导致。

​	**空间浪费**是静态库的一个问题。另一个问题是静态库对程序的更新、部署和发布也会带来麻烦。如果静态库libxx.lib更新了，所有使用它的应用程序都需要重新编译、发布给用户（对于用户来说，只是一个很小的改动，却导致整个程序重新下载，全量更新）。

​	**动态库**在程序编译时并不会被连接到目标代码中，而是在程序运行时才被载入，因此所占用内存相对静态库来说小了很多。不同的应用程序如果调用相同的库，那么在内存里只需要有一份该共享库的实例，规避了空间浪费问题。动态库在程序运行时才被载入，也解决了静态库对程序的更新、部署和发布的麻烦。用户只需要更新动态库即可，增量更新。

#### 创建与使用

首先需要**编译源文件为 PIC 目标文件**，即**位置无关代码（Position-Independent Code, PIC）**：使库能在内存中任意位置加载，供多个进程共享（必须）。

```
g++ -fPIC -c 源文件名称.cpp -o 目标文件名称.o
```

- 说明：
  - `-fPIC`：生成位置无关代码（**必须**，否则链接动态库时会报错）；
  - `-c`：只编译，不链接，输出目标文件（`.o`）；
  - `-o mylib.o`：指定输出的目标文件名（默认是 `a.o`，建议显式指定）。

 然后需要**链接目标文件为动态库**：用 `-shared` 选项将目标文件链接为动态库，并指定输出文件名（需遵循 `libxxx.so` 命名规范）：

```
g++ -shared  -o lib动态库名称.so 目标文件名称.o
```

- 说明：
  - `-shared`：生成共享库（动态库）；
  - `-o libmylib.so`：指定输出的动态库文件名（**必须以 `lib` 开头，`.so` 结尾**，否则后续无法用 `-l` 选项链接）。

制作完动态链接库后，，我们需要手动链接去使用他：

```
g++  源文件名称.cpp   -L . -l 动态库名称  -o 可执行文件名称
```

运行可执行文件时，会出现报错，是因为系统**默认的usr/lib目录下没有动态库文件**，可以通过手动转移或者**修改环境变量**
例如：在终端**输入 pwd** 查询**当前动态链接库所在工作目录的绝对路径**，并输入

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:绝对路径 
或
export LD_LIBRARY_PATH='绝对路径‘
```

$():用于引I入括号里的变量，使用export指定动态库所在的路径，再进行链接生成可执行文件。

可以使用**ldd命令**，显示程序**直接依赖的动态库**进行验证：

```
ldd main2
```

![image-20250729000644959](从gcc到catkin编译系统.assets/image-20250729000644959.png)

## Makefile

​	在前面的示例中，我们使用了三个文件：`main.cpp`、`show.cpp`和`show.h`。当项目只有少量文件时，使用g++命令直接编译是可行的。但随着项目规模的增长，手动管理编译过程会变得越来越困难。

让我们通过实际例子来说明这个问题，对于我们的示例项目，手动编译过程如下：

```bash
# 方法1：一步编译
g++ main.cpp show.cpp -o myapp

# 方法2：分步编译
g++ -c show.cpp -o show.o
g++ -c main.cpp -o main.o
g++ show.o main.o -o myapp
```

假设我们的项目逐渐扩展，增加了更多功能模块：

```
project/
├── main.cpp
├── show.cpp
├── show.h
├── utils.cpp         # 新增：工具函数
├── utils.h
├── calculator.cpp    # 新增：计算模块
├── calculator.h
├── logger.cpp        # 新增：日志模块
├── logger.h
└── config.cpp        # 新增：配置模块
    config.h
```

此时手动编译命令变成：

```bash
g++ main.cpp show.cpp utils.cpp calculator.cpp logger.cpp config.cpp -o myapp
```

**问题显现：**

1. **命令过长**：随着源文件增加，命令行变得冗长且易错
2. **重复编译**：即使只修改一个文件，也需要重新编译所有文件
3. **依赖管理**：无法自动处理文件间的依赖关系
4. **编译时间**：大项目编译时间会变得很长

​	Makefile 可以简单的认为是**一个工程文件的编译规则**，描述了整个工程的编译和链接等规则。其中包含了那些文件需要编译，那些文件不需要编译，那些文件需要先编译，那些文件需要后编译，那些文件需要重建等等。编译整个工程需要涉及到的，在 Makefile 中都可以进行描述。换句话说，Makefile 可以使得我们的项目工程的编译变得自动化，不需要每次都手动输入一堆源文件和参数

让我们为show项目创建一个简单的Makefile：

```makefile
#定义变量：target（目标文件）
target=output3
#定义变量：obj（源文件列表），wildcard是Makefile的内置函数，用于获得指定目录下指定类型的文件名
obj=$(wildcard *.cpp)
#定义"如何生成目标文件"
$(target):$(obj)
	g++ $(obj) -o $(target)
```

 **Makefile使用示例**：

> ```bash
> # 编译项目
> make
> # 清理生成的文件
> make clean
> # 重新编译
> make rebuild
> ```
>

```
编译流程图：

源文件变更检测
       ↓
   [main.cpp] ──┐
                ├── 检查时间戳 ──→ 需要编译？
   [show.cpp] ──┘                     ↓
       ↓                            是/否
   编译.o文件                         ↓
       ↓                      ┌─────────┐
   [main.o]  ──┐              │ 编译.o  │
               ├── 链接 ──→    │ 文件    │
   [show.o]  ──┘              └─────────┘
       ↓                           ↓
   生成可执行文件                链接生成
       ↓                         可执行文件
   [myapp]                        ↓
                               [myapp]
```

##  CMake



​	虽然Makefile解决了手动编译的问题，但随着项目规模的进一步扩大和跨平台需求的增加，Makefile的局限性变得明显：

1. **平台依赖**：不同操作系统的Makefile语法可能不同
2. **复杂语法**：学习曲线较陡峭
3. **维护困难**：大型项目的Makefile变得复杂且难以维护
4. **缺乏自动化**：需要手动管理依赖关系和编译选项

这些限制促使了更高级构建工具的发展，如CMake。

​	CMake是一种**过程式语言**，是一个**跨平台**的安装（编译）工具，采用了"**生成器**"的概念：可以用简单的语句来描述所有平台的安装（编译）过程。它能够产生标准的makefile或者project文件，CMake的组态档（即configuration)取名为CMakeLists.txt.

​	Cmake的所有语句都写在一个CMakeLists.txt的文件中，CMakeLists.txt文件确定后，直接使用Cmake命令进行运行，但是这个命令要指向CMakeLists.txt所在的目录，Cmake之后就会产生我们想要的makefile文件，然后再直接make就可以编译出我们需要的结果了。

更简单的解释就是Cmake是为了生成Makefile而存在，这样我们就不需要再去写Makefile了，只需要写简单的CMakeLists.txt即可。



![image-20250724225026606](从gcc到catkin编译系统.assets/image-20250724225026606.png)



让我们为同样的项目创建一个简单的CMakeLists.txt：

```cmake
# 指定CMake最低版本
cmake_minimum_required(VERSION 3.10)

# 项目名称
project(Test)

include_directories(include)

aux_source_directory(./src  SRC_LIST)

# 创建可执行文件
add_executable(output4 ${SRC_LIST})


```

 **CMake编译流程**：

> ```bash
> # 创建构建目录（推荐的out-of-source构建）
> mkdir build
> cd build
> 
> # 生成构建文件
> cmake ..
> 
> # 编译项目
> make     
> ```
>

| 特性 | Makefile | CMake |
|------|----------|-------|
| **跨平台支持** | ❌ 需要多个文件 | ✅ 单一配置文件 |
| **依赖查找** | ❌ 手动指定路径 | ✅ 自动查找系统库 |
| **语法复杂度** | ⚠️ 中等 | ✅ 相对简单 |
| **构建类型** | ⚠️ 手动管理 | ✅ 自动管理Debug/Release |
| **IDE集成** | ❌ 有限支持 | ✅ 广泛支持 |
| **学习成本** | ⚠️ 中等 | ⚠️ 中等 |
| **生态系统** | ⚠️ 传统工具 | ✅ 现代C++标准 |

这种演进展示了构建系统的发展趋势：从手动控制到自动化管理，从平台特定到跨平台通用。

## Catkin编译系统

​	对于源代码包，我们只有编译才能在系统上运行。而Linux下的编译器有gcc、g++，随着源文件的增加，直接用gcc/g++命令的方式显得效率低下，人们开始用Makefile来进行编译。然而随着工程体量的增大，Makefile也不能满足需求，于是便出现了Cmake工具。CMake是对make工具的生成器，是更高层的工具，它简化了编译构建过程，能够管理大型项目，具有良好的扩展性。对于ROS这样大体量的平台来说，就采用的是CMake，并且ROS对CMake进行了**扩展**，于是便有了**Catkin编译系统**。虽然CMake已经很强大，但ROS作为机器人操作系统，有其特殊的需求。让我们通过将show项目转换为ROS包来说明这个过程。

###  ROS包的特殊需求

假设我们要将show项目变成一个ROS节点：

**传统的ROS节点（使用标准CMake）：**

```cpp
// main.cpp - 改造为ROS节点
#include <ros/ros.h>
#include <std_msgs/String.h>
#include "show.h"

int main(int argc, char** argv)
{
    ros::init(argc, argv, "show_node");
    ros::NodeHandle nh;
    
    ros::Publisher pub = nh.advertise<std_msgs::String>("show_topic", 1000);
    
    ros::Rate loop_rate(10);
    while (ros::ok())
    {
        std_msgs::String msg;
        msg.data = "hello micu ";
        
        show();  // 调用我们的函数
        pub.publish(msg);
        
        ros::spinOnce();
        loop_rate.sleep();
    }
    
    return 0;
}
```

**标准CMake的问题：**

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyShowApp)

# 手动查找ROS相关包
find_package(catkin REQUIRED COMPONENTS
    roscpp
    std_msgs
)

# 手动设置包含路径
include_directories(${catkin_INCLUDE_DIRS})

# 手动管理ROS消息依赖
# 这里变得复杂...

add_executable(show_node main.cpp show.cpp)
target_link_libraries(show_node ${catkin_LIBRARIES})
```

**面临的问题：**

1. **包依赖管理复杂**：需要手动管理ROS包之间的依赖关系
2. **消息和服务生成**：ROS特有的.msg和.srv文件需要特殊处理
3. **工作空间管理**：多个ROS包需要统一编译和管理
4. **环境设置**：ROS特有的环境变量和路径设置



###  Catkin风格的CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(show_ros_package)

## 编译标准
add_compile_options(-std=c++11)

## 查找catkin宏和库
find_package(catkin REQUIRED COMPONENTS
  roscpp
  std_msgs
  message_generation
)

## 声明ROS消息和服务
add_message_files(
  FILES
  ShowMessage.msg
)

add_service_files(
  FILES
  ShowService.srv
)

## 生成消息
generate_messages(
  DEPENDENCIES
  std_msgs
)

## catkin包配置
catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ${PROJECT_NAME}
  CATKIN_DEPENDS roscpp std_msgs message_runtime
)

## 指定包含目录
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
)

## 创建库
add_library(${PROJECT_NAME}
  src/show.cpp
)

## 创建可执行文件
add_executable(show_node src/main.cpp)

## 指定依赖关系（确保消息文件先生成）
add_dependencies(show_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

## 链接库
target_link_libraries(show_node
  ${PROJECT_NAME}
  ${catkin_LIBRARIES}
)

## 安装规则
install(TARGETS ${PROJECT_NAME} show_node
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)

install(DIRECTORY launch/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
)
```

```
完整的构建系统演进过程：

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│   g++    │ → │ Makefile │ → │  CMake   │ → │ Catkin   │
│  手动编译 │   │  构建脚本 │   │ 跨平台构建│   │ROS生态系统│
└──────────┘   └──────────┘   └──────────┘   └──────────┘
      ↓             ↓             ↓             ↓
   
  特点：         特点：         特点：         特点：
• 直接控制      • 增量编译      • 跨平台       • ROS专用
• 简单项目      • 依赖管理      • 自动配置     • 消息/服务
• 学习用途      • 单一平台      • IDE集成      • 包管理
                                             • 工作空间

  命令：         命令：         命令：         命令：
g++ main.cpp   make           cmake ..       catkin_make
show.cpp       make clean     make           catkin build
-o myapp       make rebuild   cmake --build  rosrun pkg node

  适用规模：     适用规模：     适用规模：     适用规模：
1-5个文件      10-50个文件    100+个文件     ROS项目
                              多平台项目     机器人系
```

### 基本编译命令

```bash
# 创建工作空间
mkdir -p catkin_ws/src
cd catkin_ws/src
catkin_init_workspace

# 编译整个工作空间
cd ..
catkin_make

# 编译特定包
catkin_make -DCATKIN_WHITELIST_PACKAGES=package_name

# 清理编译文件
catkin_make clean
```

**命令说明：**

- `catkin_init_workspace`：初始化catkin工作空间
  - **作用**：在src目录创建CMakeLists.txt链接文件
  - **位置**：必须在src目录下执行

- `catkin_make`：编译工作空间
  - **作用**：自动查找所有包并按依赖顺序编译
  - **并行编译**：默认使用多核并行编译
  - **输出**：生成的文件放在devel目录

- `--DCATKIN_WHITELIST_PACKAGES`：只编译指定包及其依赖
  - **用途**：调试时避免编译整个工作空间
  - **依赖解析**：自动处理包之间的依赖关系

### 环境变量配置

```bash
# 设置环境变量（每次打开终端都需要）
source devel/setup.bash

# 永久设置（添加到~/.bashrc）
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc

# 验证环境设置
echo $ROS_PACKAGE_PATH
rospack find package_name
```

**环境变量说明：**

- `ROS_PACKAGE_PATH`：ROS包搜索路径
- `setup.bash`：自动设置所有必要的环境变量
- **链式设置**：工作空间可以叠加，后source的会覆盖前面的，因此导入多个工作空间的环境变量后面需要加上**--extend**

## ROS中的CMakeLists.txt解析

###  基本结构

​	CMakeLists.txt是**CMake工具的配置文件**，用于描述**项目的构建过程**。在ROS项目中，每个package都需要一个CMakeLists.txt文件来定义编译规则。一个典型的ROS CMakeLists.txt文件包含以下几个部分：

1. **项目基本信息：编译、链接、构建可执行文件所需的最低CMake版本 (cmake_minimum_required)和功能包的名称 (project(package_name))**
2. **管理构建这个功能包需要依赖的其他功能包以及相关系统依赖项 (find_package())；**
3. **启动Python模块支持 (catkin_python_setup())；**
4. **自定义消息/服务/操作(Message/Service/Action)生成器，**
5. **在此处列出的任何添加的消息和服务生成的依赖项(generate_messages())；**
6. **指明构建makefile文件所需的功能包 (catkin_package())；**
7. **用来设置头文件的相对路径 (include_directories())；**
8. **添加要编译的库和可执行文件 (add_library()/add_executable()/target_link_libraries())；**

###  常用指令详解

#### 项目基本信息

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(your_package_name)
```

**指令说明：**
- `cmake_minimum_required(VERSION 3.0.2)`：指定所需的CMake最低版本
  - **作用**：确保使用的CMake功能与版本兼容
  - **参数**：VERSION后跟版本号，ROS Melodic推荐3.0.2+
  - **示例**：`cmake_minimum_required(VERSION 3.5)`

- `project(your_package_name)`：声明项目名称
  - **作用**：定义项目名称，影响生成的目标文件名
  - **参数**：项目名称，通常与package名称保持一致
  - **示例**：`project(my_robot_controller)`

#### 管理所需依赖功能包

```cmake
find_package(catkin REQUIRED COMPONENTS
  roscpp
  rospy
  std_msgs
  geometry_msgs
  tf2
  tf2_ros
)
```

**指令说明：**

- `find_package(catkin REQUIRED)`：这里指明构建当前功能包需要**依赖的package**，我们使用catkin_make的编译方式，至少需要catkin这个功能包。
  - **作用**：定位catkin编译系统及其提供的宏和函数
  - **REQUIRED关键字**：表示该包是必需的，找不到时停止编译
  - **COMPONENTS关键字**：指定需要的具体组件或依赖包

​	如果一个功能包被find_package，那么它就会导致一些**CMake变量**的产生，这些变量后面将在CMake的脚本中用到，这些变量描述了所依赖的包输出的头文件、源文件、库文件在哪里。这些变量的名字依照的惯例是**<package_name>_<property>（<功能包名>_<属性>）**，比如：

- **<NAME>_FOUND：**这个变量说明这个库是否被找到，如果找到就被设置为true，否则设为false；
- **<NAME>_INCLUDE_DIRS** or **<NAME>_INCLUDES**：这个包输出的头文件目录；
- **<NAME>_LIBRARIES** or **<NAME>_LIBS**：这个包输出的库文件（由自定义类和函数编译链接形成的二进制.o可执行文件）。

需要的所有包我们都可用这种方式包含进来，比如我们还需要roscpp，rospy，std_msgs。我们可以写成：

```
find_package(catkin REQUIRED）
find_package(roscpp REQUIRED）  
find_package(rospy REQUIRED)  
find_package(std_msgs REQUIRED)  
```

这样的话，每个依赖的package都会产生几个变量，这样很不方便。所以还有另外一种方式：

```
find_package(catkin REQUIRED COMPONENTS  
               roscpp  
               rospy  
               std_msgs  
)
```

​	这样，它会把所有pacakge里面的头文件和库文件等等目录加到一组变量上，最终就**只产生一组变量**。例如：catkin_INCLUDE_DIRS（代指./devel/include目录）这样，我们就可以用这个变量查找需要的文件了。

#### 查找系统依赖项

​	前面catkin_XXX 作为CMAKE变量为几乎每个程序编译所需要（构建库文件所需功能包），而之后的比如 Boost_XXX, OpenCV_XXX 则是在编译特定文件时用的到（依赖项）。所以我们分成多个find_package，去指明系统依赖项。

```cmake
find_package(Boost REQUIRED COMPONENTS system)
find_package(OpenCV REQUIRED)
```

**指令说明：**
- `find_package(PackageName REQUIRED)`：查找系统级第三方库
  
  - **作用**：定位并配置外部依赖库
  - **Boost库**：C++扩展库，提供线程、网络、文件系统等功能
  - **OpenCV**：计算机视觉库，用于图像处理和分析
  
  这个依赖项是我们在ROS中使用boost库时需要声明的。如果我们使用boost库还需要声明以下内容：

```
// 由于源文件的编写中调用了boost库，因此需要加载系统依赖项——boost库  
find_package(Boost REQUIRED COMPONENTS system)  
// 包含boost导出的.h头文件  
include_directories(  
  include  
  ${catkin_INCLUDE_DIRS}  
  ${Boost_INCLUDE_DIRS}  
)  
// 将二进制可执行文件链接boost库  
target_link_libraries(demo ${catkin_LIBRARIES} ${Boost_LIBRARIES})  
```

#### 自定义消息文件生成

​	当我们需要使用**.msg .srv .action**形式的文件时，我们需要特殊的预处理器把他们转化为**系统可以识别特定编程语言（.h/.cpp**）。 系统会用里面所有的(一些编程语言)生成器（比如 gencpp, genpy, genlisp, etc）生成相应的.cpp .py文件。这就需要三个宏：**add_message_files， add_service_files，add_action_files**来相应的控制.msg .srv .action。这些宏后面必须跟着一个调用**generate_messages()** 用来处理使用三个宏指明的消息文件使之**生成符合C++标准（或者其他编程语言标准）的头文件和源文件**。
```cmake
add_message_files(
  FILES
  CustomMessage.msg
  SensorData.msg
)

add_service_files(
  FILES
  CalculateDistance.srv
  ResetPosition.srv
)

add_action_files(
  FILES
  NavigateToGoal.action
)

generate_messages(
  DEPENDENCIES
  std_msgs
  geometry_msgs
)
```

**指令说明：**

- `add_message_files(FILES ...)`：添加自定义消息文件
  - **作用**：指定需要生成C++/Python代码的.msg文件
  - **位置**：消息文件通常放在msg/目录下
  - **生成结果**：自动生成对应的头文件和Python模块

- `add_service_files(FILES ...)`：添加服务文件
  - **作用**：定义ROS服务的请求和响应格式
  - **位置**：服务文件放在srv/目录下
  - **文件格式**：包含请求（Request）和响应（Response）两部分

- `add_action_files(FILES ...)`：添加动作文件
  - **作用**：定义长时间运行任务的目标、反馈和结果
  - **位置**：动作文件放在action/目录下
  - **应用场景**：导航、机械臂运动等需要反馈的任务

- `generate_messages(DEPENDENCIES ...)`：生成消息代码
  - **作用**：触发消息、服务、动作文件的代码生成
  - **依赖**：指定生成的消息所依赖的其他消息包

####  指明编译构建C++源文件所需的功能包

catkin_package()是catkin提供的CMake宏，用于为catkin提供构建、生成pkg-config和CMake文件所需要的信息。形式如下所示：

```cmake
catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ${PROJECT_NAME}
  CATKIN_DEPENDS roscpp rospy std_msgs geometry_msgs
  DEPENDS system_lib
)
```

**指令说明：**
- `catkin_package(...)`：配置catkin包的导出信息
  - **作用**：声明包对外提供的接口和依赖关系

**参数详解：**
- `INCLUDE_DIRS include`：表明我们这个功能包的.h文件都存放在这个功能包下面的include文件夹下
  
  - **作用**：其他包依赖此包时可以找到头文件
  - **路径**：该功能包中头文件所在的相对路径
  
- `LIBRARIES ${PROJECT_NAME}`：指明需要依赖该功能包的其他功能包
  
  - **作用**：声明包生成的库供其他包使用
  - **变量**：`${PROJECT_NAME}`引用project()中定义的项目名
  
- `CATKIN_DEPENDS`：声明在catkin编译系统中，编译本功能包所需的catkin官方的依赖功能包；
  
  - **作用**：指定运行时需要的其他官方功能包
  - **传递性**：依赖会传递给使用此包的其他包
  
- `DEPENDS`：声明在编译这些源文件时，所需要的非catkin官方的依赖功能包，即系统依赖
  
  - **作用**：指定需要的系统级库（非catkin官方的依赖功能包）
  - **示例**：OpenCV、PCL、Boost等可用于C++编程的库。我们要想实现功能就一定会借助像OpenCV、Boost这样的库，这个依赖项就是用来声明这个功能包所用到的除了catkin官方给出的库以外的库。
  
  此外，DEPENDS还有不同功能包继承依赖的作用，例如：
  
  在功能包A的CMakelist.txt中我们设置了:
  
  ```
  find_package{Boost REQUIRED}  
  ...
  ...
  catkin_package{  
                 ...
                 DEPENDS Boost  
  } 
  ```
  
  
  ​	又由于功能包B依赖于功能包A，因此功能包B无需再用find_package{Boost REQUIRED}来包含boost依赖库了，如果DEPENDS中未包含boost，那么功能包B必须包含加载boost库所需的一切操作，即不能够继承功能包A所带来的便利。

#### 头文件包含路径

```cmake
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS}
)
```

**指令说明：**
- `include_directories(...)`：添加头文件搜索路径。告知catkin编译器，要找头文件一方面去本功能包下的include目录去找，另外也去catkin_INCLUDE_DIRS这个目录下去找。catkin_INCLUDE_DIRS目录下存储着着由catkin功能包编译而成的.h文件，因此以“catkin功能包名称+_INCLUDE_DIRS”组成。
  - **作用**：告诉编译器在哪里查找头文件
  - **include**：本包的头文件目录
  - `${catkin_INCLUDE_DIRS}`：所有catkin依赖包的头文件路径
  - `${PackageName_INCLUDE_DIRS}`：特定第三方库的头文件路径

​	以此类推，要是find_package{Boost REQUIRED}，那boost库中的.h文件存放路径为Boost_INCLUDE_DIRS。当我们调用boost库的.h文件时，只需加载${Boost_INCLUDE_DIRS}即可。

####  编译库文件

​	我们一般编写C++程序时，自定义类类型的定义与自定义类类型的实例化不在一个.cpp源文件中，因此要在XXX.cpp中使用到我们自定义的数据类型就必须要将我们自定义数据类型所在文件（也成为库文件）进行声明。要是不声明，编译器根本找不到自定义数据类型所在的文件。

```cmake
add_library(${PROJECT_NAME}
  src/utility.cpp
  src/algorithm.cpp
  src/data_processor.cpp
)

```

​	我们上述代码就声明了一个库文件，这个库文件整体名称映射为${PROJECT_NAME}，即我们的前面定义的项目名，这个映射名称就代表XXX.h和XXX.cpp组成的库文件。

**指令说明：**

- `add_library(library_name source_files...)`：创建库文件
  - **作用**：将源文件编译成静态或动态库
  - **库名**：通常使用项目名称
  - **源文件**：相对路径，通常在src/目录下

**add_library{...}一般什么时候使用呢？**

C++代码编写风格一般有如下两种：

​	编写C++代码时，我们常常将“**类/结构体的声明、定义和使用“相互剥离**，在ROS项目文件中我们也可以这样做。在ROS项目文件中声明、定义、使用自定义数据类型时，我们可以进行如下两种方式的操作：

① 自定义数据类型的声明+自定义数据类型的定义及使用

![image-20250802225535994](从gcc到catkin编译系统.assets/image-20250802225535994.png)

② 自定义数据类型的声明+自定义数据类型的定义+自定义数据类型的使用

![image-20250802225629500](从gcc到catkin编译系统.assets/image-20250802225629500.png)



​	当**库的定义和调用分开**时（即第二种情况：自定义数据类型的定义（库文件的定义）和调用分开编写在不同文件中），我们才使用add_library{...}去告知catkin编译器（ROS使用的编译器）：有一个自定义数据类型库文件是由XXX.h和XXX.cpp文件构成的，把它加载到库文件目录中以后用的时候调用即可。



#### 指明可执行文件/头文件的依赖项

```
add_dependencies(robot_controller 
  ${${PROJECT_NAME}_EXPORTED_TARGETS}
  ${catkin_EXPORTED_TARGETS}
)
```

- `add_dependencies(target dependencies...)`：添加编译依赖
- - **作用**：确保在编译目标前先完成依赖项
  - `${${PROJECT_NAME}_EXPORTED_TARGETS}`：如果在编译包或者执行文件时，需要用到msg和srv，就要显示调用由message_generation自动生成的用于编译消息文件的依赖项，对应于不同消息_EXPORTED_TARGETS也不同（前面的${${PROJECT_NAME}...}不变）
  - `${catkin_EXPORTED_TARGETS}`：用于为add_excutable()中映射的可执行文件提供catkin官方依赖包。

####  编译生成可执行文件

```cmake
add_executable(robot_controller 
  src/main.cpp
  src/controller.cpp
)
```

**指令说明：**
- `add_executable(executable_name source_files...)`：创建可执行文件，第一个参数为期望生成的可执行文件名称；后面的参数为参与编译的源文件（cpp),如果需要多个代码文件，用空格区分开。
  - **作用**：编译生成可执行程序
  - **命名**：可执行文件名，避免与系统命令冲突
  - **源文件**：包含main函数的源文件及其依赖

#### 指定要链接库或可执行目标的库

```
// 为可执行文件指定链接规则
target_link_libraries(robot_controller
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
)
```

- `target_link_libraries(target_name libraries...)`：指定可执行文件链接的库，这个要用在add_executable()后面

- - **作用**：将目标与依赖库链接
  - **目标**：库或可执行文件名称
  - **库文件**：需要链接的所有库

  ${catkin_LIBRARIES}表示由catkin功能包编译而成的可执行库。

  一般指定链接规则之前，一定要用add_dependancies设置依赖项，例如：

  ```
  // exec_name为add_executable(exec_name ...)中映射的可执行文件名  
  //exec_name所表示的源文件链接的前提是catkin已经编译完毕
  add_dependencies(exec_name ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS}) 
  
  
  // 为exec_name添加依赖项：catkin_LIBRARIES
  // 将已经编译完毕的catkin生成库文件链接exec_name可执行文件
  target_link_libraries(exec_name 
    ${catkin_LIBRARIES}  
  ) 
  ```

  ${...}的存在可以区分谁到底是谁链接谁，即谁的运行以谁的运行完毕为前提。

### Catkin_packages和find_packages的区别

​	catkin_packages和find_packages都是指明所需依赖功能包，他们有何不同呢？我们可以了解一下“静态库和共享库的区别，构建依赖和执行依赖的区别”，前面已经讲过：**静态库是代码库可供我们调用，而共享库是编译了之后的二进制文件供编译使用。**

- **Build Dependencies构建依赖项**其实就是静态库，在构建程序的过程中将.srv、.msg、.action等这些文件转化为符合C++（或其他编程语言）的头文件（我们将自定义消息文件转化为了类类型定义存储在头文件中），“转化”的含义：将静态库中的代码用替换的方式将自定义文件中的对应部分进行替换得到了符合C++标准的头文件。
- **Execution Dependencies执行依赖项**其实就是共享库（也称为动态库，后缀为.so），构建可执行文件的过程也是共享库链接由源文件编译而来的二进制文件的过程。
- **find_packages{}**包含的是静态库文件，主要用途是将自定义消息文件使用ROS提供的库文件转化为符合C++标准的头文件（仅以C++为例）。
- **catkin_packages{}**包含的是共享库文件（动态库文件），主要用途是在编译过程中将二进制共享库文件链接进“由源文件编译而来的XXX.o二进制目标文件”之中。

​	一般来讲，ROS中像roscpp这样的库不仅存在相应的静态库也存在相应的共享库（动态库），但有时也有例外：在消息文件的处理过程中，generation_messages依赖项使用roscpp静态库将自定义消息文件转化为C++头文件并存放在./devel/include目录之下（经此操作自定义文件转化为了C++头文件），而在进行二进制文件的链接操作时用到的功能包是runtime_messages依赖项，该依赖项使用roscpp二进制共享库进行编译时的链接操作（经此操作.o二进制文件转化为了可执行文件）。

