--

### Makefile
```makefile
main: main.o utils.o
	g++ main.o utils.o -o main

main.o: main.cpp
	g++ -c main.cpp

clean:
	rm -f *.o main
```

写了 Makefile 之后，改完代码只要敲一次 `make`。

### CMake

```
CMakeLists.txt  --cmake-->  Makefile  --make-->  可执行文件
```


```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject CXX)
add_executable(MyApp main.cpp)
```



### 常用指令

| 指令 | 作用 |
| --- | --- |
| `cmake_minimum_required(VERSION x.y)` | 最低版本要求 |
| `project(名字 CXX)` | 项目名和语言 |
| `add_executable(目标 源文件...)` | 生成可执行文件 |
| `add_library(目标 STATIC/SHARED 源文件...)` | 生成库 |
| `target_link_libraries(目标 库...)` | 链接库 |
| `target_include_directories(目标 作用域 路径)` | 头文件搜索路径 |
| `set(变量 值...)` | 定义变量 |
| `find_package(包 REQUIRED)` | 查找外部依赖 |
| `add_dependencies(目标 依赖)` | 显式声明构建顺序 |

### 构建流程

```bash
mkdir build && cd build     # 外部构建，把产物和源码分开（推荐）
cmake ..                    # 生成 Makefile
make                        # 实际编译
# 或者用 cmake --build . 代替 make（跨平台写法）
```

---

### Catkin工作空间结构

```
catkin_ws/
├── src/          # 源代码空间，放各个功能包
├── build/        # 构建空间，中间文件（一般不用管）
├── devel/        # 开发空间，编译产物 + setup.bash
└── install/      # 安装空间，make install 之后才有
```

- **build**：CMake 生成的中间文件，基本用不到，删了能重新生成
- **devel**：编译出来的可执行文件、库，以及 `setup.bash`
- **install**：只有执行 `catkin_make install` 才有，用于分发

### 常用命令

```bash
# 初始化工作空间
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
catkin_init_workspace       # 在 src 下创建指向 toplevel.cmake 的软链接

# 编译
cd ~/catkin_ws
catkin_make
source devel/setup.bash     # 让环境找到刚编译的包

# 创建功能包
cd ~/catkin_ws/src
catkin_create_pkg 包名 依赖1 依赖2
```


---

## 六、用大模型辅助学习的过程记录

> **这一节留给你自己写。**
>
> 下面是几个引导问题，挑你实际遇到过的写，不必全填。没有的就删掉，别硬凑。
> 真实的过程即使粗糙也比编造的完整更有价值。

### 6.1 我实际问过的问题

记录 2-3 个你真实问过的、和 CMake/catkin 有关的问题，以及得到回答后你的判断：

```text
我的问题：
（原话写出来）

AI 的回答要点：
（它说了什么）

我核实了吗 / 怎么核实的：
（跑命令？看源码？查文档？结果和它说的一样吗？）

结论：
（这个回答可信吗？哪里对、哪里不对）
```

### 6.2 被误导或者没用的经历

有没有哪次 AI 说的东西你后来发现是错的、过时的、或者根本不可运行？

```text
场景：

它说的：

实际情况：

我是怎么发现的：
```

> 如果还没有这种经历，写「暂未遇到」并说明你目前怎么防范这类问题。

### 6.3 哪些地方 AI 帮到了我

```text
（例如：解释某个报错、帮我理解某个概念、出练习题检验……）
```

### 6.4 哪些地方 AI 帮不上

```text
（例如：判断哪种写法更好、知道具体版本的行为、理解项目为什么要这样组织……）
```

---

## 七、对「用大模型帮助学习」的思考

> **这一节也留给你自己写。** 课件里提到「对团队负责任，对自己负责任」，可以从这几个方向想：

### 7.1 它改变了我的学习方式吗

```text
（学 CMake 之前 vs 之后，遇到不懂的东西我的第一反应变了吗？）
```

### 7.2 我担心什么

```text
（例如：会不会产生「我看过了 = 我懂了」的错觉？会不会懒得查官方文档？）
```

### 7.3 我打算怎么用它

```text
（具体的自我约束，比如：涉及版本行为必须自己跑一遍确认）
```


## 参考

- 菜鸟教程 CMake 基础：https://www.runoob.com/cmake/cmake-basic.html
- The Emperor Has No Clothes：https://www.mihaileric.com/The-Emperor-Has-No-Clothes/
- ROS wiki Catkin Tutorials：https://wiki.ros.org/catkin/Tutorials
  - [Creating a workspace](https://wiki.ros.org/catkin/Tutorials/create_a_workspace)
  - [Building and using packages](https://wiki.ros.org/catkin/Tutorials/using_a_workspace)
- 知乎文章：https://zhuanlan.zhihu.com/p/476697014 （未能访问，待补充）
