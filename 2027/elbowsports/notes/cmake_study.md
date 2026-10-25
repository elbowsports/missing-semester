# CMake 与 Catkin 学习笔记

第二课补充资料学习笔记。记录 CMake、Catkin 的学习内容，以及使用大模型辅助学习的交互过程和思考。

实验环境：Ubuntu 24.04 宿主机 + Docker 容器内 ROS Noetic（CMake 3.16.3 / g++ 9.4.0）

> 说明：文中标注「实测」的部分是在本地 ROS 容器里实际执行得到的真实输出。

## 目录

- [一、从源代码到可执行程序](#一从源代码到可执行程序)
- [二、Makefile 与 CMake](#二makefile-与-cmake)
- [三、CMake 语法要点](#三cmake-语法要点)
- [四、Catkin](#四catkin)
- [五、补充资料学习](#五补充资料学习)
- [六、用大模型辅助学习的过程记录](#六用大模型辅助学习的过程记录)
- [七、对「用大模型帮助学习」的思考](#七对用大模型帮助学习的思考)

---

## 一、从源代码到可执行程序

计算机只执行二进制，我们写的高级语言要经过翻译。PPT 里把这个过程拆成四步：

| 步骤 | 做什么 | 产物 |
| --- | --- | --- |
| 预处理 | 展开宏、处理 `#include`、去掉注释 | `.i` |
| 编译 | 高级语言 → 汇编 | `.s` |
| 汇编 | 汇编 → 机器码 | `.o` |
| 链接 | 把多个 `.o` 和库合并成可执行文件 | 可执行文件 |

容易误以为「编译」是一个动作，实际上编译器内部是四个阶段。区分这四步的实际用处在于定位报错：`undefined reference` 是链接阶段的问题（找不到实现），`No such file or directory: xxx.h` 是预处理阶段的问题（找不到头文件）。

### 为什么不直接敲 g++

单个文件直接 `g++ main.cpp -o main` 没问题。但文件一多，每次都要手写一长串文件名和选项，还容易漏。而且改动一个文件没必要全部重新编译。这就是需要构建系统的原因。

---

## 二、Makefile 与 CMake

### Makefile 解决什么

Makefile 相当于一份自动编译说明书，告诉 `make`：哪些文件要编译、编译顺序、用什么命令。关键是它**按时间戳判断**是否需要重新编译——只重编改动过的文件。

```makefile
main: main.o utils.o
	g++ main.o utils.o -o main

main.o: main.cpp
	g++ -c main.cpp

clean:
	rm -f *.o main
```

写了 Makefile 之后，改完代码只要敲一次 `make`。

### 为什么还需要 CMake

Makefile 有两个明显问题：

1. **语法晦涩**，规则多的时候维护困难
2. **不跨平台**，GNU make 的语法在 Windows 的 Visual Studio 上不通用

CMake 的思路是**再抽象一层**：你写一份 `CMakeLists.txt`（与平台无关），CMake 读取后在本地生成对应的构建文件（Linux 下生成 Makefile，Windows 下生成 VS 工程）。

```
CMakeLists.txt  --cmake-->  Makefile  --make-->  可执行文件
```

所以 CMake 不是编译器，也**不是替代 make**，它是「构建系统的生成器」（build system generator）。它负责**生成** makefile，真正编译的还是 make 调用的 g++。

### 三者关系

PPT 那张演进图可以归纳成：

```
gcc/g++  ──文件多了要手写→  Makefile  ──更大更复杂→  CMake
                (自动化)                    (跨平台 + 可维护)
```

每一层都是为了解决上一层的规模问题，不是后面的一定要淘汰前面的。

---

## 三、CMake 语法要点

### 最小可运行示例

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject CXX)
add_executable(MyApp main.cpp)
```

三行就够了。`cmake_minimum_required` 必须在最顶部。

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

**为什么要在 build 目录里构建**：CMake 会生成大量中间文件（CMakeCache.txt、CMakeFiles/ 等），混在源码目录里很难清理。外部构建只要删掉 build 目录就干净了。

### 变量与缓存

普通变量用 `set` 定义，作用域是当前目录及子目录，每次运行 cmake 重新计算。

缓存变量会写进 `CMakeCache.txt`，跨多次运行保留：

```cmake
set(MY_PATH "/usr/local" CACHE PATH "安装路径")
set(ENABLE_LOG ON CACHE BOOL "是否开启日志")
```

命令行覆盖：

```bash
cmake .. -DMY_PATH=/opt/myapp -DENABLE_LOG=OFF
```

这里有个容易踩的坑：改了 CMakeLists.txt 里缓存变量的默认值之后不生效，因为 CMakeCache.txt 里已经存了旧值。必须删掉 build 目录（或 CMakeCache.txt）重新配置。表象很像 bug，实际是缓存机制。

### PUBLIC / PRIVATE / INTERFACE

这是 CMake 里最容易搞混的地方。直觉上的理解：

- 一个库有自己的头文件，**它自己编译**需要这些路径 → `PRIVATE`
- 别人用这个库的时候**也需要**这些头文件才能编译 → `PUBLIC`
- 别人用这个库，但库本身没东西要编译（纯头文件库）→ `INTERFACE`

| 关键字 | 目标自己可用 | 依赖它的目标可用 | 典型场景 |
| --- | --- | --- | --- |
| `PRIVATE` | ✅ | ❌ | 内部实现用的头文件 |
| `PUBLIC` | ✅ | ✅ | 库对外暴露的 API |
| `INTERFACE` | ❌ | ✅ | header-only 库 |

理解方式：这个关键字描述的是**「这个路径要不要传递给消费者」**，而不是「谁能用它」。`PRIVATE` 也能用，只是不传出去。

### 新式写法 vs 老式写法

菜鸟教程里提到，`include_directories()` / `link_directories()` 是全局指令，会影响当前目录下所有目标；`target_include_directories()` / `target_link_libraries()` 只作用于指定目标。

```cmake
# 老式：全局污染，不推荐
include_directories(${PROJECT_SOURCE_DIR}/include)
link_directories(/opt/lib)
target_link_libraries(MyApp mylib)

# 新式：绑定到目标，推荐
target_include_directories(MyApp PRIVATE ${PROJECT_SOURCE_DIR}/include)
target_link_libraries(MyApp PRIVATE /opt/lib/libmylib.so)
```

直观类比：老式写法像全局变量，新式写法像局部变量。项目大了之后全局变量的副作用很难排查，所以新写法更好。

---

## 四、Catkin

### Catkin 是什么

ROS 官方文档的说法：catkin 是 ROS 对 CMake 的**扩展**，也就是在 CMake 之上加了一层约定和工具。

为什么要扩展？因为 ROS 的包非常多，每个包都是独立的 CMake 工程，但编译时又要统一处理依赖。catkin 做的事情：

1. 规定每个 ROS 包必须有 `package.xml`（声明依赖）
2. 规定包的目录结构
3. 把所有包的 CMakeLists.txt 聚合成一个大的 CMake 工程来统一构建

这也是为什么 catkin 工作空间里**只有一个 build 目录**——虽然有很多包，但对 CMake 来说是一个大项目。

### 工作空间结构

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

**实测**：`catkin_init_workspace` 实际做的就是建一个符号链接：

```bash
Creating symlink "/home/ros/demo_ws/src/CMakeLists.txt"
  pointing to "/opt/ros/noetic/share/catkin/cmake/toplevel.cmake"
```

所以它不是什么黑魔法，只是把 catkin 的顶层 CMakeLists 链接到你的 `src` 下。这样 `catkin_ws/src` 本身就成了一个 CMake 工程的根。

### catkin_make 到底做了什么

`catkin_make` 的内部逻辑值得看清楚，因为它是理解 CMake 与 catkin 关系的关键。查源码（它是个 Python 脚本，用 `catkin.builder`），核心是两步：

1. `cmake ../src -DCATKIN_DEVEL_PREFIX=../devel -DCMAKE_INSTALL_PREFIX=../install`（在 build 目录里执行）
2. `make -j...`（在 build 目录里执行）

**实测**跑 `catkin_make` 的输出能直接看到这两步：

```
#### Running command: "cmake /home/ros/catkin_ws/src
     -DCATKIN_DEVEL_PREFIX=/home/ros/catkin_ws/devel
     -DCMAKE_INSTALL_PREFIX=/home/ros/catkin_ws/install
     -G Unix Makefiles" in "/home/ros/catkin_ws/build"
#### Running command: "make -j18 -l18" in "/home/ros/catkin_ws/build"
```

所以 **`catkin_make` = 帮你调用 cmake + make 的快捷方式**。它本身不编译任何东西，只是拼接参数、创建工作空间目录、然后调用这两个工具。

理解这一点之后，`catkin_make` 的各种参数就好理解了——它们大部分是直接透传给 cmake 的，比如 `-DCMAKE_BUILD_TYPE=Release`。

### 关于 catkin_make vs catkin build

PPT 里写的是 `catkin build`，但实际 ros-noetic 默认装的是 `catkin_make`。

- `catkin_make` 是官方最早提供的工具，行为是「一个大 CMake 工程」
- `catkin build` 来自 `catkin_tools` 包，需要额外 `sudo apt install python3-catkin-tools`

两者都能编译，区别在于 `catkin build` 是**每个包独立调用 cmake/make**，所以可以并行编译不同包、支持单独编译某个包，日志也更清晰。`catkin_make` 是全局一个大工程，所有包同时配置。

两个都能用，初学用 `catkin_make` 就够了。这里两个都提到，是为了避免以后看到 `catkin build` 时困惑。

---

## 五、补充资料学习

### 资料 1：菜鸟教程 CMake 基础

**链接**：https://www.runoob.com/cmake/cmake-basic.html

这篇是语法速查性质，把常用指令列了一遍，每个都带注释和示例。下面几个点值得单独记：

**1. `add_custom_command` 的 PRE_BUILD / PRE_LINK / POST_BUILD**

之前不知道能在构建的不同阶段插入自定义命令：

```cmake
add_custom_command(
    TARGET MyApp POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E echo "构建完成！"
    COMMENT "打印构建完成信息"
)
```

这个在 ROS 里常用来拷贝配置文件或生成消息头。

**2. `find_package` 的两种用法**

```cmake
find_package(Boost REQUIRED)              # 需要找得到
find_package(Boost 1.70 REQUIRED)         # 要求最低版本
find_package(OpenCV REQUIRED PATHS /opt)  # 指定搜索路径
```

找到之后推荐用**导入目标**而不是手动拼变量：

```cmake
target_link_libraries(MyApp PRIVATE Boost::Boost)   # 推荐
# 而不是
include_directories(${Boost_INCLUDE_DIRS})          # 老式
target_link_libraries(MyApp ${Boost_LIBRARIES})
```

原因是导入目标会把头文件路径、库路径、编译选项一次性带过来，不会漏。

**3. 缓存变量默认值不更新的坑**

文章里专门提醒：改了 `set(... CACHE ...)` 的默认值之后，`CMakeCache.txt` 里已有值不会被覆盖，要删掉重建。这是个容易被忽略的点，因为表面上改了配置却不生效，很像 bug。

**这篇的定位**：适合当速查表，但不适合作为入门的第一份材料。它一上来就是指令罗列，对「为什么要这样」讲得不多。比较合理的顺序是先理解 CMake 在整个构建流程里的位置（第二节），再回来查具体指令的写法。

---

### 资料 2：The Emperor Has No Clothes

**链接**：https://www.mihaileric.com/The-Emperor-Has-No-Clothes/

标题是「皇帝没有穿衣服」，用 200 行 Python 写一个能读写文件的 coding agent，目的是戳破「AI 编程助手很神奇」的幻觉。

**核心观点**：Claude Code 这类工具的核心不是魔法，就是大约 200 行直白的 Python。

**它的工作模型（核心循环）**：

```
用户发消息
   ↓
LLM 决定需要工具 → 返回结构化的工具调用
   ↓
你的程序在本地执行这个调用（真的去读写文件）
   ↓
结果发回给 LLM
   ↓
LLM 基于结果继续，或者给出最终回答
```

作者强调一句话：**LLM 从来没有真正碰你的文件系统**，它只是「请求」某些事情发生，是你的代码让它发生的。

**只需要三个工具**：

| 工具 | 作用 |
| --- | --- |
| `read_file` | 让 LLM 看到代码内容 |
| `list_files` | 让它能浏览项目结构 |
| `edit_file` | 让它能创建和修改文件 |

生产级的工具（Claude Code）会多 `grep`、`bash`、`websearch` 等，但核心就是这三个。

**几个值得注意的实现细节**：

1. **docstring 很重要**：工具函数的 docstring 会被拼进 system prompt 给 LLM 看，所以写清楚是影响效果的关键
2. **`edit_file` 的约定**：`old_str` 为空表示创建文件，非空表示查找替换。简单但够用
3. **解析方式极其朴素**：就是找以 `tool:` 开头的行，取括号里的 JSON。没有复杂的协议
4. **双层循环**：外层读用户输入，内层反复调 LLM + 执行工具，直到 LLM 不再要求调用工具为止

**和本课主题的关系**：这篇本身和 CMake 无关，但它解释了 Codex CLI 这类工具为什么能直接改本地文件——它只是一个带工具的对话循环。结合 PPT 里「小明装 codex cli 接入 deepseek」那个例子，链路就清楚了。

**文章的核心结论**：coding agent 的核心不是魔法，而是一个「LLM 请求 → 本地程序执行 → 结果回传给 LLM」的循环。由此可以推出一条重要推论：**工具的能力边界就是 agent 的能力边界**。它只能读写文件和执行命令，无法判断你的项目为什么这样组织。

---

### 资料 3：ROS wiki Catkin Tutorials

**链接**：https://wiki.ros.org/catkin/Tutorials

wiki 的教程索引页，其中两篇正文最有价值：

#### Creating a workspace for catkin

标准流程（实测一致）：

```bash
source /opt/ros/$ROS_DISTRO/setup.bash    # 先有 ROS 环境
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/
catkin_make                                # 第一次会自动创建 src/CMakeLists.txt
source devel/setup.bash
echo $ROS_PACKAGE_PATH                     # 验证 overlay 生效
```

有个容易忽略的细节：**`catkin_make` 第一次运行时会自动在 `src/` 下建一个 CMakeLists.txt 链接**（这和手动跑 `catkin_init_workspace` 效果一样）。

`ROS_PACKAGE_PATH` 这个变量是关键——source 之后它应该包含你的工作空间路径，且**你的路径在前**，所以能覆盖系统里的同名包。

#### Building and using catkin packages in a workspace

这篇讲了 `catkin_make` 背后到底发生了什么，是理解 CMake 和 catkin 关系最直接的材料。

**catkin_make 的常用参数**：

```bash
catkin_make -DCMAKE_BUILD_TYPE=Release   # 编译类型
catkin_make install                       # 执行 install 目标
catkin_make --force-cmake                 # 强制重新运行 cmake（新加了包时用）
catkin_make -DCMAKE_VERBOSE_MAKEFILE=ON   # 显示实际执行的编译命令
```

**不用 catkin_make 手动构建**（这篇最精华的部分）：

```bash
mkdir ~/catkin_ws/build && cd ~/catkin_ws/build
cmake ../src -DCMAKE_INSTALL_PREFIX=../install -DCATKIN_DEVEL_PREFIX=../devel
make
make install
```

这三步和 `catkin_make` 做的事完全一样，**印证了 catkin_make 本质上是个封装**。

**devel 和 install 的选择**：

wiki 明确说：**不要同时 source devel 和 install 的 setup.bash**。

- 开发阶段用 devel：改代码不装就能测，尤其 Python 改完直接生效
- 分发时用 install

**关于 catkin_make 的限制**：wiki 提到一个具体问题——`catkin_make` 里所有命令共享同一个环境，如果测试依赖生成的 Python 代码，生成的路径不在 PYTHONPATH 里就找不到。解决办法是用 `catkin_make_isolated`。这又一次说明了「一个大工程」这个设计的局限。

**关于装到哪儿**：wiki 强烈建议开发时不要装到 `/usr/local`，装在 workspace 里方便卸载和换 ROS 版本。而且**没有任何卸载命令**——这一点比较反直觉，装错了只能手动删。

#### 关于这篇 wiki 的一个背景

wiki 顶部有醒目提示：**ROS 1 的最后一个版本 Noetic 已于 2025 年 5 月 31 日 EOL，这个 wiki 也会转为存档**。

这也解释了相关内容为什么偏旧（很多页面最后编辑是 2013-2017 年）。也意味着——**以后学 ROS 应该学 ROS 2，catkin 已经被 colcon 取代**。catkin 的知识主要用于维护现有 ROS 1 项目。

---

### 资料 4：知乎文章（待补充）

**链接**：https://zhuanlan.zhihu.com/p/476697014

> **这一篇我没能读到。**
>
> 我尝试直接访问，返回 HTTP 403；换了几个 User-Agent 也还是 403。知乎现在对未登录的访问做了拦截，无法获取正文。
>
> 我没有去搜二手转述来「假装读过」——那样写出来的笔记没有意义，也可能把别人的错误理解当成事实。
>
> **以下留空，等我自己登录后读原文再补。** 记录一下我打算关注的问题：
>
> - 这篇讲的是 CMake 的哪个方面？（语法 / 工程组织 / 和 catkin 的关系？）
> - 有没有提到我没搞懂的点，比如多级 CMakeLists 的嵌套、`add_subdirectory`？
> - 和菜鸟教程那篇相比，深度和角度有什么不同？
>
> （待补充……）

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
