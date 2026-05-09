

# IDE背后的命令 #

## 1. IDE是什么

IDE指集成开发环境(Integrated Development Environment)。
我们开发STM32F103等单片机程序时使用是keil就是一种IDE。
使用IDE，很容易操作，点点鼠标就可完成：
* 添加文件
* 指定文件路径(头文件路径、库文件路径)
* 指定链接库
* 编译、链接
* 下载、调试
## 2. IDE的背后是命令
现场使用keil来演示命令的操作。
* 注意
* 使用GitBash执行命令的话
由于GitBash采用类似Linux的文件路径表示方法(比如 /d/abc，而非 d:\abc)，命令行中windows格式的路径名要加上双引号，比如`".\objects\main.o"`
* 使用dos命令行执行命令的话
不需要加双引号
* 在某个Keil工程所在目录下，打开Git Bash：
`doc_and_source_for_mcu_mpu\STM32MF103\source\02_录制视频时现场编写的源码\01_led_c`
* 编译main.c
执行命令：
```

"C:\Keil_v5\ARM\ARMCC\Bin\ArmCC" --c99 --gnu -c --cpu Cortex-M3 -D__EVAL -g -O0 --apcs=interwork --split_sections -I.\RTE\_led_c -I"C:\Users\thisway_diy\AppData\Local\Arm\Packs\Keil\STM32F1xx_DFP\2.3.0\Device\Include" -I"C:\Keil_v5\ARM\CMSIS\Include" -D__UVISION_VERSION="527" -DSTM32F10X_HD -o ".\objects\main.o" --omf_browse ".\objects\main.crf" --depend ".\objects\main.d" "main.c"

```
* 编译start.S
执行命令：
```

"C:\Keil_v5\ARM\ARMCC\Bin\ArmAsm" --cpu Cortex-M3 --pd "__EVAL SETA 1" -g --apcs=interwork -I.\RTE\_led_c -I"C:\Users\thisway_diy\AppData\Local\Arm\Packs\Keil\STM32F1xx_DFP\2.3.0\Device\Include" -I"C:\Keil_v5\ARM\CMSIS\Include" --pd "__UVISION_VERSION SETA 527" --pd "STM32F10X_HD SETA 1" --list ".\listings\start.lst" --xref -o ".\objects\start.o" --depend ".\objects\start.d" "start.s"

```

* 链接
执行命令：
```

"C:\Keil_v5\ARM\ARMCC\Bin\ArmLink" --cpu Cortex-M3 ".\objects\main.o" ".\objects\start.o" --ro-base 0x08000000 --entry 0x08000000 --rw-base 0x20000000 --entry Reset_Handler --first __Vectors --strict --summary_stderr --info summarysizes --map --load_addr_map_info --xref --callgraph --symbols --info sizes --info totals --info unused --info veneers --list ".\Listings\led_c.map" -o ".\Objects\led_c.axf"

```

## 3. 提出几个问题
* 头文件在哪？

* 库文件在哪？库文件是哪个？

* 源文件有哪些？

* 源文件怎么编译？可以指定编译参数吗？


* 多个源文件怎么链接成一个可执行程序？

* 有a.c, b.c, c.c，我只修改了a.c，就只需要编译a.c，然后在链接：怎么做到的？

## 4. 要解决这些疑问，需要了解命令行
如果你只学习单片机，只想使用keil，当然可以不学习命令行。
但是如果想升级到Linux、各类RTOS，需要掌握命令行。
## 5. 有两套主要的编译器

* armcc
* ARM公司的编译器
* keil使用的就是armcc

* gcc
* GNU工具链
* Linux等开源软件经常使用gcc
后面以GNU工具链为例讲解，所涉及的知识可以平移到armcc上。

# 准备工作
## 1. arm-linux-gcc和gcc是类似的
* arm-linux-gcc
* 给ARM芯片编译程序
* gcc
* 在x86编译程序
* 用法基本一样
* 为方便演示，我们使用gcc
* 为了方便在windows下演示，我们使用`Code::Blocks`
* 它的安装程序自带gcc

## 2. Code::Blocks

它是一款基于GCC的windows IDE，可以用来开发C/C++/Fortran。
官网地址：`http://www.codeblocks.org/`
![[003_download_codeblocks.png]]
  
在我们提供的GIT仓库里也有：`git clone https://e.coding.net/weidongshan/noos/cortexA7_windows_tools.git`

下载GIT后，在apps目录下。

  

  

### 2.1 安装
双击安装。
### 2.2 设置windows环境变量

在Path环境变量中添加：`C:\Program Files\CodeBlocks\MinGW\bin`
### 2.3 命令行示例

启动Git Bash，编译程序hello.c:
```c
#include <stdio.h>

int main(void)
{
printf("hello, world!\n");
return 0;
}

```
编译、运行命令如下：
```
gcc -o hello hello.c
./hello.exe
```

# gcc编译过程详解 #

## 1. 程序编译4步骤

  ![[001_4_steps 1.png]]
  
我们经常使用“编译”泛指上面的4个步骤之一，甚至有时候会囊括这四个步骤。
## 2. gcc的使用方法
 
```
gcc [选项] 文件名
```

  ### 2.1 gcc使用示例
```
gcc hello.c // 输出一个名为a.out的可执行程序，然后可以执行./a.out
gcc -o hello hello.c // 输出名为hello的可执行程序，然后可以执行./hello
gcc -o hello hello.c -static // 静态链接

gcc -c -o hello.o hello.c // 先编译(不链接)
gcc -o hello hello.o // 再链接
```
### 2.2 gcc常用选项

#### 2.2.1 手工控制编译过程

| 选项 | 功能 |
| --- |-------------|
| -v | 查看gcc编译器的版本，显示gcc执行时的详细过程 |
| -o \<file> | 指定输出文件名为file，这个名称不能跟源文件名同名 |
| -E | 只预处理，不会编译、汇编、链接t |
| -S | 只编译，不会汇编、链接 |
| -c | 编译和汇编，不会链接 |
一个c/c++文件要经过预处理、编译、汇编和链接才能变成可执行文件。
（1）预处理

C/C++源文件中，以“#”开头的命令被称为预处理命令，如包含命令“#include”、宏定义命令“#define”、条件编译命令“#if”、“#ifdef”等。预处理就是将要包含(include)的文件插入原文件中、将宏定义展开、根据条件编译命令选择要使用的代码，最后将这些东西输出到一个“.i”文件中等待进一步处理。

（2）编译

编译就是把C/C++代码(比如上述的“.i”文件)“翻译”成汇编代码。

（3）汇编

汇编就是将第二步输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件(OBJ文件)。“反汇编”是指将机器代码转换为汇编代码，这在调试程序时常常用到。

（4）链接

链接就是将上步生成的OBJ文件和系统库的OBJ文件、库文件链接起来，最终生成了可以在特定平台运行的可执行文件。

  

hello.c(预处理)->hello.i(编译)->hello.s(汇编)->hello.o(链接)->hello

详细的每一步命令如下：

```
gcc -E -o hello.i hello.c
gcc -S -o hello.s hello.i
gcc -c -o hello.o hello.s
gcc -o hello hello.o
```

上面一连串命令比较麻烦，gcc会对.c文件默认进行预处理操作，使用-c再来指明了编译、汇编，从而得到.o文件,

再将.o文件进行链接，得到可执行应用程序。简化如下：

```
gcc -c -o hello.o hello.c
gcc -o hello hello.o
```
#### 2.2.2 使用后缀名决定编译过程
 
参考`《嵌入式Linux应用开发完全手册》`：
![[002_gcc_default_operation.png]]
* 总结
* 输入文件的后缀名和选项共同决定gcc到底执行那些操作
* 在编译过程中，最后的步骤都是链接
* 除非使用了-E、-S、-c选项
* 或者编译出错阻止了完整的编译过程
#### 2.2.3 指定头文件目录
头文件在哪里？
* 系统目录
* 系统目录在哪？工具链里的某个include目录
* 怎么确定？
```
echo 'main(){}'| gcc -E -v - // 它会列出头文件目录、库目录(LIBRARY_PATH)
```
* 可以不使用系统include目录吗？可以，编译时指定参数`-nostdinc`
* 可以自己指定头文件目录
```
-I <头文件目录>
```

#### 2.2.4 指定库文件
库文件在哪里？
* 系统目录
* 系统目录在哪？工具链里的某个lib目录
* 怎么确定？
```
echo 'main(){}'| gcc -E -v - // 它会列出头文件目录、库目录(LIBRARY_PATH)
```
* 可以不使用系统lib目录吗？可以，编译时指定参数`-nostdlib`
* 可以自己指定库文件目录
```
-L <库文件目录>
```

* 指定库文件

```
-l <abc> // 链接 libabc.so 或 lib.a
```
# 3. 开发板程序编译示例
最后链接时，使用arm-linux-ld而不是使用arm-linux-gcc
* 前者可以完全自己指定所连接的文件
* 后者会链接一些默认的启动文件
# 4. 参考书籍
`《嵌入式Linux应用开发完全手册》中的《3.1 交叉编译工具选项说明》`

# Makefile的引入及规则
使用keil, mdk,avr等工具开发程序时点击鼠标就可以编译了，它的内部机制是什么？它怎么组织管理程序？怎么决定编译哪一个文件？

答：实际上windows工具管理程序的内部机制，也是Makefile，我们在linux下来开发裸板程序的时候，使用Makefile组织管理这些程序，本节我们来讲解Makefile最基本的规则。Makefile要做什么事情呢？
组织管理程序，组织管理文件，我们写一个程序来实验一下：
文件a.c

```C
#include <stdio.h>

int main()
{
func_b();
return 0;
}
```

文件b.c
```c
#include <stdio.h>
void func_b()
{
printf("This is B\n");
}
```

编译：
```bash
gcc -o test a.c b.c
```

运行：
```bash
./test
```

结果：
```bash
This is B
```

**gcc -o test a.c b.c** 这条命令虽然简单，但是它完成的功能不简单。

我们来看看它做了哪些事情，

我们知道.c程序 ==》 得到可执行程序它们之间要经过四个步骤：
* 1.预处理
* 2.编译
* 3.汇编
* 4.链接
我们经常把前三个步骤统称为编译了。我们具体分析：gcc -o test a.c b.c这条命令
它们要经过下面几个步骤：
* 1）对于**a.c**：执行：预处理 编译 汇编 的过程，**a.c ==>xxx.s ==>xxx.o** 文件。
* 2）对于**b.c**：执行：预处理 编译 汇编 的过程，**b.c ==>yyy.s ==>yyy.o** 文件。
* 3）最后：**xxx.o**和**yyy.o**链接在一起得到一个**test**应用程序。

提示：**gcc -o test a.c b.c -v** ：加上一个**‘-v’**选项可以看到它们的处理过程，

第一次编译 a.c 得到 xxx.o 文件，这是很合乎情理的， 执行完第一次之后，如果修改 a.c 又再次执行：**gcc -o test a.c b.c**，对于 a.c 应该重新生成 xxx.o，但是对于 b.c 又会重新编译一次，这完全没有必要，b.c 根本没有修改，直接使用第一次生成的 yyy.o 文件就可以了。

缺点：对所有的文件都会再处理一次，即使 b.c 没有经过修改，b.c 也会重新编译一次，当文件比较少时，这没有没有什么问题，当文件非常多的时候，就会带来非常多的效率问题如果文件非常多的时候，我们，只是修改了一个文件，所用的文件就会重新处理一次，编译的时候就会等待很长时间。

对于这些源文件，我们应该分别处理，执行：预处理 编译 汇编，先分别编译它们，最后再把它们链接在一次，比如：
编译：

```bash
gcc -o a.o a.c
gcc -o b.o b.c
```

链接：
```bash

gcc -o test a.o b.o

```
比如：上面的例子，当我们修改a.c之后,a.c会重现编译然后再把它们链接在一起就可以了。b.c就不需要重新编译。
那么问题又来了，怎么知道哪些文件被更新了/被修改了？
比较时间：比较 a.o 和 a.c 的时间，如果a.c的时间比 a.o 的时间更加新的话，就表明 a.c 被修改了，同理b.o和b.c也会进行同样的比较。比较test和 a.o,b.o 的时间，如果a.o或者b.o的时间比test更加新的话，就表明应该重新生成test。Makefile就是这样做的。我们现在来写出一个简单的Makefile:
makefie最基本的语法是规则，规则：
```bash

目标 : 依赖1 依赖2 ...

[TAB]命令

```

当“依赖”比“目标”新，执行它们下面的命令。我们要把上面三个命令写成makefile规则，如下：
```bash
test ：a.o b.o //test是目标，它依赖于a.o b.o文件，一旦a.o或者b.o比test新的时候，就需要执行下面的命令，重新生成test可执行程序。
gcc -o test a.o b.o
a.o : a.c //a.o依赖于a.c，当a.c更加新的话，执行下面的命令来生成a.o
gcc -c -o a.o a.c
b.o : b.c //b.o依赖于b.c,当b.c更加新的话，执行下面的命令，来生成b.o
gcc -c -o b.o b.c
```
我们来作一下实验：
在改目录下我们写一个Makefile文件：
文件：Makefile
```Makefile
test:a.o b.o
gcc -o test a.o b.o

a.o : a.c

	gcc -c -o a.o a.c

b.o : b.c
    gcc -c -o b.o b.c

```
上面是makefile中的三条规则。makefile,就是名字为“makefile”的文件。当我们想编译程序时，直接执行make命令就可以了，一执行make命令它想生成第一个目标test可执行程序,
如果发现a.o 或者b.o没有，就要先生成a.o或者b.o，发现a.o依赖a.c，有a.c但是没有a.o,他就会认为a.c比a.o新，就会执行它们下面的命令来生成a.o，同理b.o和b.c的处理关系也是这样的。
如果修改a.c ，我们再次执行make，它的本意是想生成第一个目标test应用程序,它需要先生成a.o,发现a.o依赖a.c(执行我们修改了a.c)发现a.c比a.o更加新，就会执行gcc -c -o a.o
a.c命令来生成a.o文件。b.o依赖b.c，发现b.c并没有修改，就不会执行gcc -c -o b.o
b.c来重新生成b.o文件。现在a.o b.o都有了，其中的a.o比test更加新，就会执行 gcc -o
test a.ob.o来重新链接得到test可执行程序。所以当执行make命令时候就会执行下面两条执行：
```bash
gcc -c -o a.o a.c
gcc -o test a.o b.o
```
我们第一次执行make的时候，会执行下面三条命令(三条命令都执行)：
```bash
gcc -c -o a.o a.c
gcc -c -o b.o b.c
gcc -o test a.o b.o
```
再次执行make 就会显示下面的提示：
```bash
make: `test' is up to date.
```
我们再次执行make就会判断Makefile文件中的依赖，发现依赖没有更新，所以目标文件就不会重现生成，就会有上面的提示。当我们修改a.c后，重新执行make,  
就会执行下面两条指令：
```bash
gcc -c -o a.o a.c
gcc -o test a.o b.o
```

我们同时修改a.c b.c，执行make就会执行下面三条指令。
```bash
gcc -c -o a.o a.c
gcc -c -o b.o b.c
gcc -o test a.o b.o
```
a.c文件修改了，重新编译生成a.o, b.c修改了重新编译生成b.o，a.o,b.o都更新了重新链接生成test可执行程序，makefile的规则其实还是比较简单的。规则是Makefie的核心，
执行make命令的时候，就会在当前目录下面找到名字为：Makefile的文件，根据里面的内容来执行里面的判断/命令。

# Makefile的语法
本节我们只是简单的讲解Makefile的语法，如果想比较深入
学习Makefile的话可以：

* a. 百度搜 "gnu make 于凤昌"。
* b. 查看官方文档: [http://www.gnu.org/software/make/manual/](http://www.gnu.org/software/make/manual/)
## a. 通配符
假如一个目标文件所依赖的依赖文件很多，那样岂不是我们要写很多规则，这显然是不合乎常理的,我们可以使用通配符，来解决这些问题。
我们对上节程序进行修改代码如下：
```bash
test: a.o b.o
gcc -o test $^
%.o : %.c
gcc -c -o $@ $<
```
%.o：表示所用的.o文件
%.c：表示所有的.c文件
\$\@：表示目标
\$\<：表示第1个依赖文件
\$\^：表示所有依赖文件
我们来在该目录下增加一个 c.c 文件，代码如下：
```bash
#include <stdio.h>

void func_c()
{
printf("This is C\n");
}
```

然后在main函数中调用修改Makefile，修改后的代码如下：
```bash
test: a.o b.o c.o
gcc -o test $^
%.o : %.c
gcc -c -o $@ $<
```

执行：
```bash
make
```

结果：
```bash
gcc -c -o a.o a.c
gcc -c -o b.o b.c
gcc -c -o c.o c.c
gcc -o test a.o b.o c.o
```

运行：
```bash
./test
```

结果：
```bash
This is B
This is C
```
## b. 假想目标: .PHONY
1.我们想清除文件，我们在Makefile的结尾添加如下代码就可以了：
```bash
clean:
rm *.o test
```

*1）执行 make ：生成第一个可执行文件。
*2）执行 make clean : 清除所有文件，即执行： rm \*.o test。

make后面可以带上目标名，也可以不带，如果不带目标名的话它就想生成第一个规则里面的第一个目标。

2.使用Makefile
执行：**make [目标]** 也可以不跟目标名，若无目标默认第一个目标。我们直接执行make的时候，会在makefile里面找到第一个目标然后执行下面的指令生成第一个目标。当我们执行 make clean 的时候，就会在 Makefile 里面找到 clean 这个目标，然后执行里面的命令，这个写法有些问题，原因是我们的目录里面没有 clean 这个文件，这个规则执行的条件成立，他就会执行下面的命令来删除文件。

如果：该目录下面有名为clean文件怎么办呢？  
我们在该目录下创建一个名为 “clean” 的文件，然后重新执行：make然后make
clean，结果(会有下面的提示：)：
```bash
make: \`clean' is up to date.
```

它根本没有执行我们的删除操作，这是为什么呢？
我们之前说，一个规则能过执行的条件：
* 1)目标文件不存在
* 2)依赖文件比目标新
现在我们的目录里面有名为“clean”的文件，目标文件是有的，并且没有
依赖文件，没有办法判断依赖文件的时间。这种写法会导致：有同名的"clean"文件时，就没有办法执行make clean操作。解决办法：我们需要把目标定义为假象目标，用关键子PHONY
```bash
.PHONY: clean //把clean定义为假象目标。他就不会判断名为“clean”的文件是否存在，
```
然后在Makfile结尾添加.PHONY: clean语句，重新执行：make clean，就会执行删除操作。
## C. 变量
在makefile中有两种变量：
1), 简单变量(即使变量)：
A := xxx  \# A的值即刻确定，在定义时即确定
对于即使变量使用 “:=” 表示，它的值在定义的时候已经被确定了
2）延时变量
B = xxx \# B的值使用到时才确定
对于延时变量使用“=”表示。它只有在使用到的时候才确定，在定义/等于时并没有确定下来。
想使用变量的时候使用“\$”来引用，如果不想看到命令是，可以在命令的前面加上"\@"符号，就不会显示命令本身。当我们执行make命令的时候，make这个指令本身，会把整个Makefile读进去，进行全部分析，然后解析里面的变量。常用的变量的定义如下：
```bash
:= # 即时变量
= # 延时变量
?= # 延时变量, 如果是第1次定义才起效, 如果在前面该变量已定义则忽略这句
\+= # 附加, 它是即时变量还是延时变量取决于前面的定义
?=: 如果这个变量在前面已经被定义了，这句话就会不会起效果，
```

实例：
```bash
A := $(C)
B = $(C)
C = abc
#D = 100ask
D ?= weidongshan

all:
	@echo A = $(A)
	@echo B = $(B)
	@echo D = $(D)
	
C += 123
```

执行：

```bash
make
```

结果：
```bash
A =
B = abc 123
D = weidongshan
```
分析：
1) A := \$(C)：
A为即使变量，在定义时即确定，由于刚开始C的值为空，所以A的值也为空。
2) B = \$(C)：
B为延时变量，只有使用到时它的值才确定，当执行make时，会解析Makefile里面的所用变量，所以先解析C= abc,然后解析C += 123，此时，C = abc 123，当执行：\@echo B = \$(B) B的值为 abc 123。
3) D ?= weidongshan：
D变量在前面没有定义，所以D的值为weidongshan，如果在前面添加D = 100ask，最后D的值为100ask。
我们还可以通过命令行存入变量的值 例如：
执行：make D=123456 里面的 D ?= weidongshan 这句话就不起作用了。
结果：
```bash
A =
B = abc 123
D = 123456
```

# Makefile函数
makefile里面可以包含很多函数，这些函数都是make本身实现的，下面我们来几个常用的函数。引用一个函数用“\$”。
## 函数foreach
函数foreach语法如下：
```bash
$(foreach var,list,text)
```

前两个参数，‘var’和‘list’，将首先扩展，注意最后一个参数 ‘text’ 此时不扩展；接着，对每一个 ‘list’ 扩展产生的字，将用来为 ‘var’ 扩展后命名的变量赋值；然后 ‘text’ 引用该变量扩展；因此它每次扩展都不相同。结果是由空格隔开的 ‘text’。在 ‘list’ 中多次扩展的字组成的新的 ‘list’。‘text’ 多次扩展的字串联起来，字与字之间由空格隔开，如此就产生了函数 foreach 的返回值。

实例：
```bash
A = a b c
B = $(foreach f, &(A), $(f).o)

all:
	@echo B = $(B)
```

结果：
```bash
B = a.o b.o c.o
```

## 函数filter/filter-out
函数filter/filter-out语法如下：
```bash
$(filter pattern...,text) # 在text中取出符合patten格式的值
$(filter-out pattern...,text) # 在text中取出不符合patten格式的值
```
实例：
```bash
C = a b c d/
D = $(filter %/, $(C))
E = $(filter-out %/, $(C))

all:
	@echo D = $(D)
	@echo E = $(E)
```

结果：
```bash
D = d/
E = a b c
```

## Wildcard
函数Wildcard语法如下：
```bash
$(wildcard pattern) # pattern定义了文件名的格式, wildcard取出其中存在的文件。
```
这个函数 wildcard 会以 pattern 这个格式，去寻找存在的文件，返回存在文件的名字。

实例：
在该目录下创建三个文件：a.c b.c c.c
```bash
files = $(wildcard *.c)

all:
	@echo files = $(files)
```

结果：
```bash
files = a.c b.c c.c
```
我们也可以用wildcard函数来判断，真实存在的文件

实例：
```bash
files2 = a.c b.c c.c d.c e.c abc
files3 = $(wildcard $(files2))

all:
	@echo files3 = $(files3)
```
结果：
```bash
files3 = a.c b.c c.c
```
## patsubst函数
函数 patsubst 语法如下：
```bash
$(patsubst pattern,replacement,\$(var))
```
patsubst 函数是从 var 变量里面取出每一个值，如果这个符合 pattern 格式，把它替换成 replacement 格式，
实例：
```bash
files2 = a.c b.c c.c d.c e.c abc

dep_files = $(patsubst %.c,%.d,$(files2))

all:
	@echo dep_files = $(dep_files)
```

结果：
```bash
dep_files = a.d b.d c.d d.d e.d abc
```

# Makefile实例 #
前面讲了那么多Makefile的知识，现在开始做一个实例。
之前编译的程序`002_syntax`，有个缺陷，将其复制出来，新建一个`003_example`文件夹，放在里面。
在`c.c`里面，包含一个头文件`c.h`，在`c.h`里面定义一个宏，把这个宏打印出来。
c.c:
```
#include <stdio.h>
#include <c.h>
void func_c()
{
	printf("This is C = %d\n", C);
}
```
c.h:
```
#define C 1
```
然后上传编译，执行`./test`,打印出：
```
This is B
This is C =1
```
测试没有问题，然后修改`c.h`：
```
#define C 2
```
重新编译，发现没有更新程序，运行，结果不变，说明现在的Makefile存在问题。
为什么会出现这个问题呢， 首先我们test依赖c.o，c.o依赖c.c，如果我们更新c.c，会重新更新整个程序。
但c.o也依赖c.h，我们更新了c.h，并没有在Makefile上体现出来，导致c.h的更新，Makefile无法检测到。
因此需要添加:
```
c.o : c.c c.h
```
现在每次修改c.h，Makefile都能识别到更新操作，从而更新最后输出文件。
这样又冒出了一个新的问题，我们怎么为每个.c文件添加.h文件呢？对于内核，有几万个文件，不可能为每个文件依次写出其头文件。
因此需要做出改进，让其自动生成头文件依赖，可以参考这篇文章：http://blog.csdn.net/qq1452008/article/details/50855810
```
gcc -M c.c // 打印出依赖  

gcc -M -MF c.d c.c // 把依赖写入文件c.d  

gcc -c -o c.o c.c -MD -MF c.d // 编译c.o, 把依赖写入文件c.d
```
修改Makefile如下：
```
objs = a.o b.o c.o

dep_files := $(patsubst %,.%.d, $(objs))
dep_files := $(wildcard $(dep_files))

test: $(objs)
	gcc -o test $^

ifneq ($(dep_files),)
include $(dep_files)
endif

%.o : %.c
	gcc -c -o $@ $< -MD -MF .$@.d

clean:
	rm *.o test

distclean:
	rm $(dep_files)
	
.PHONY: clean
```

首先用obj变量将.o文件放在一块。
利用前面讲到的函数，把obj里所有文件都变为.%.d格式，并用变量dep_files表示。
利用前面介绍的wildcard函数，判断dep_files是否存在。
然后是目标文件test依赖所有的.o文件。
如果dep_files变量不为空，就将其包含进来。
然后就是所有的.o文件都依赖.c文件，且通过-MD -MF生成.d依赖文件。
清理所有的.o文件和目标文件
清理依赖.d文件。

现在我门修改了任何.h文件，最终都会影响最后生成的文件，也没任何手工添加.h、.c、.o文件，完成了支持头文件依赖。

下面再添加CFLAGS，即编译参数。比如加上编译参数-Werror，把所有的警告当成错误。

```
CFLAGS = -Werror -Iinclude
…………

%.o : %.c
gcc $(CFLAGS) -c -o $@ $< -MD -MF .$@.d
```
现在重新make，发现以前的警告就变成了错误，必须要解决这些错误编译才能进行。在`a.c`里面声明一下函数：

```
void func_b();
void func_c();
```
重新make，错误就没有了。
除了编译参数-Werror，还可以加上-I参数，指定头文件路径，-Iinclude表示当前的inclue文件夹下。
此时就可以把c.c文件里的`#include ".h"`改为`#include <c.h>`，前者表示当前目录，后者表示编译器指定的路径和GCC路径。








