# 第一课笔记：Linux、Vim & Shell、Git & GitHub

DynamicX 27 赛季控制视觉招新培训第一课
Ubuntu 24.04

## 一、Linux 基础

### 系统与目录

Linux 通常指内核，发行版 = 内核 + 系统工具与软件包 + 默认配置。常见的有 Ubuntu、Debian、Fedora、Arch。团队用 Ubuntu，原因是 ROS 兼容性、部署一致性和现有开发生态。

机器人开发中，开发电脑通过网络连接机器人主机，控制程序可以在无桌面环境运行，所以需要习惯终端操作。

目录结构从 `/` 开始：

| 路径 | 内容 |
| --- | --- |
| `/home/user` | 用户文件 |
| `/etc` | 系统配置 |
| `/usr` | 程序与库 |
| `/var` | 日志等可变数据 |
| `/dev` | 设备文件 |

路径区分大小写。`~` 是家目录，`.` 是当前目录，`..` 是上一级。绝对路径从 `/` 开始，相对路径从当前位置开始。

### 终端与 Shell

在终端输入命令，Shell 解释后交给程序执行。

命令结构是 `命令 [选项] [参数]`，例如 `ls -la ~/projects`：

- 命令：做什么
- 选项：怎样执行，`-l` 详细信息，`-a` 含隐藏项
- 参数：操作哪个对象

常用命令：

| 命令 | 作用 |
| --- | --- |
| `pwd` | 显示当前目录 |
| `ls` | 列出目录内容 |
| `cd 目录` | 进入指定目录，`cd ..` 上一级，`cd ~` 家目录 |
| `mkdir -p` | 创建目录，允许补齐父目录 |
| `touch` | 创建空文件或更新时间 |
| `cp` | 复制文件 |
| `mv` | 移动或重命名 |
| `rm` | 删除文件 |
| `cat` / `less` | 直接查看 / 分页查看 |

`rm` 通常不进回收站，删除前先确认路径和文件名。

### 四种常用组合

```bash
name="RM"; echo "$name"           # 变量
echo "hello" > hello.txt          # 重定向：> 覆盖，>> 追加
ls -la | grep txt                 # 管道：输出交给下一个程序
mkdir -p demo && cd demo          # 条件连接：前一条成功才执行下一条
```

遇到陌生命令先查帮助：`ls --help` 快速看用法，`man ls` 打开手册（按 `q` 退出）。`Tab` 补全命令或路径，`↑`/`↓` 调出历史命令。

## 二、Vim

Vim 常用于 SSH 和无桌面环境中的配置、代码编辑。它是有模式的编辑器：

- 普通模式：移动、删除、复制
- 插入模式：输入文本
- 命令行：保存、退出

普通模式按 `i` 进入插入模式，按 `Esc` 返回普通模式，按 `:` 进入命令行。

必备按键：

| 按键 | 操作 |
| --- | --- |
| `h j k l` | 左 / 下 / 上 / 右 |
| `dd` | 删除当前行 |
| `yy` / `p` | 复制当前行 / 粘贴 |
| `u` | 撤销 |
| `/word` | 搜索 word |

命令行：`:w` 保存，`:q` 退出，`:wq` 保存退出，`:q!` 放弃修改退出。

练习流程：`vim hello.txt` 打开 → `i` 输入文本 → `Esc` → `:wq` 保存退出 → 再次打开，用 `dd`、`u`、`/word` 练习删除、撤销和搜索。

## 三、Git 与 GitHub

### 分工

Git 负责本地版本控制：记录每次提交、比较修改、查看历史、为功能创建分支。

GitHub 负责远程托管与协作：托管 Git 仓库、通过 PR 讨论评审、合并团队认可的修改。

### 四个位置

```
工作区 --git add--> 暂存区 --git commit--> 本地仓库 --git push--> GitHub 远程仓库
```

commit 只记录到本地仓库，push 才把提交发送到远程。

### 首次配置

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

姓名与邮箱会写入提交记录，用来回答"这次修改由谁提交"。

### SSH 密钥

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

公钥 `id_ed25519.pub` 复制到 GitHub 的 SSH 设置；私钥 `id_ed25519` 不上传、不截图、不分享。

添加到 GitHub 的步骤：Settings → SSH and GPG keys → New SSH key，选 Authentication Key，填名称并粘贴完整公钥。然后用 `ssh -T git@github.com` 测试，首次连接先核对官方主机指纹。

### 常用命令

```bash
git clone <仓库地址>            # 下载仓库及提交历史
git status                     # 查看当前分支和文件状态
git diff                       # 查看尚未暂存的修改
git log --oneline              # 简短格式查看提交历史

git switch -c feature/hello    # 创建并切换到功能分支
git add hello.txt              # 暂存指定文件
git commit -m "Add greeting"   # 记录一次本地提交
git push -u origin feature/hello   # 推送并建立跟踪关系
git switch main                # 切回主分支
git pull                       # 获取并整合远程更新
```

切分支前先看 `status`。

### 分支与 PR 工作流

main 保持稳定，feature 分支承载本次修改，评审后再合并。分支让同一仓库中的开发互不打扰。

团队默认用 Fork 协作：Fork 仓库副本到个人账号，Clone 到本地，在 feature 分支上修改、add、commit，push 到个人 Fork，再向团队仓库发起 PR，评审后 Merge。有团队仓库推送权限时可以省略 Fork，直接用同仓库分支。

Fork 是仓库副本，Branch 是仓库内的开发线，这是两者的区别。

## 四、回顾

| 环节 | 作用 |
| --- | --- |
| Linux | 进入开发环境 |
| Shell | 定位与操作文件 |
| Vim | 编辑文件内容 |
| Git | 分支、暂存与提交 |
| GitHub | PR、评审与合并 |

## 五、课后任务

仓库：https://github.com/OceanCing/missing-semester

1. fork 目标仓库，clone 到本地
2. 阅读 `2027` 文件夹中的 `TASK.md`，理解任务要求后完成
3. 按工作流规范提交 PR 到目标仓库
4. 该仓库后续还会用于提交其他作业
5. 学有余力可以提前了解 catkin 与 CMake
