# Lecture 1 Notes: Linux, Vim & Shell, Git & GitHub

> Course: Missing Semester (2027)
> Author: elbowsports
> Environment: Ubuntu 24.04.5 LTS, kernel 7.0.0-31-generic, bash 5.2.21

These notes summarize the first lecture of the training course. The three parts
cover the Unix/Linux environment, text editing with Vim together with shell
scripting, and version control with Git and GitHub. Every command below was
practised on the Ubuntu 24.04 machine described above.

## Table of Contents

1. [Linux](#1-linux)
2. [Vim & Shell](#2-vim--shell)
3. [Git & GitHub](#3-git--github)
4. [Cheat Sheet](#4-cheat-sheet)
5. [Common Pitfalls](#5-common-pitfalls)

---

## 1. Linux

### 1.1 The shell and the file system

A Linux system is organized as a single tree that starts at `/`. The shell is a
program that reads commands and asks the kernel to execute them. Bash is the
default interactive shell on Ubuntu.

```bash
pwd                 # print working directory
ls -alh             # list with hidden files, long format, human sizes
cd /etc && cd -     # change directory, then return to the previous directory
```

Important locations:

| Path | Meaning |
| --- | --- |
| `/` | root of the file system |
| `/home/<user>` | home directory, written as `~` |
| `/etc` | system-wide configuration |
| `/var/log` | log files |
| `/usr/bin` | most installed executables |
| `/tmp` | temporary files, wiped on reboot |

Relative paths use `.` for the current directory and `..` for the parent.
A path containing spaces must be quoted: `cd "my notes"`.

### 1.2 Files, permissions and users

Every file carries an owner, a group and three permission triples. The string
`-rw-r--r--` means the owner may read and write, while group and others may only
read. Numerically `r=4`, `w=2` and `x=1`, so `rw-r--r--` equals `644`.

```bash
chmod 755 script.sh          # owner rwx, everyone else rx
chmod 600 ~/.ssh/id_ed25519  # a private key must never be group/world readable
chown user:group file        # change owner, requires root
id                           # show current user and groups
sudo command                 # run a single command as root
```

The `sudo` group grants administrative rights on Ubuntu. Commands that modify
system state usually need it, which is why package installation and service
management are prefixed with `sudo`.

### 1.3 Package management with apt

```bash
sudo apt update              # refresh the package index (no install)
sudo apt upgrade -y          # upgrade installed packages
sudo apt install vim         # install
sudo apt remove vim          # uninstall, keep configuration
sudo apt purge vim           # uninstall and delete configuration
apt search <keyword>         # search
apt show <package>           # details
apt policy <package>         # candidate version and source
```

Use `apt` interactively and `apt-get` inside scripts, because the latter has
stable output that is easier to parse.

### 1.4 Processes and services

```bash
ps aux                       # every process
ps aux | grep roscore        # filter
pgrep -af roscore            # match by name and show the command line
kill <pid>                   # terminate a process politely
kill -9 <pid>                # force termination, last resort
top                          # live view, press q to quit
```

Ubuntu uses systemd to supervise services:

```bash
systemctl status docker       # state of a unit
sudo systemctl enable --now docker   # start now and on every boot
journalctl -u docker -n 50    # last 50 log lines of a unit
journalctl -u docker -f       # follow logs in real time
```

### 1.5 Redirection, pipes and environment variables

```bash
command > file      # overwrite stdout into file
command >> file     # append stdout
command 2> err.log  # redirect stderr only
command &> all.log  # redirect both streams
cmd1 | cmd2         # feed stdout of cmd1 into stdin of cmd2
```

Environment variables store configuration for processes:

```bash
echo $PATH                    # where the shell looks for executables
export MY_VAR=value           # set for the current session and children
unset MY_VAR
```

To make a variable permanent, append the `export` line to `~/.bashrc` and run
`source ~/.bashrc`. Changes in a child shell never propagate upwards.

---

## 2. Vim & Shell

### 2.1 Why Vim

Vim is a modal editor. Instead of holding modifier keys it switches between
modes, so the whole keyboard becomes available for editing. This is slower to
learn but much faster once the motions become muscle memory.

| Mode | Enter with | Purpose |
| --- | --- | --- |
| Normal | `Esc` | navigation and editing commands |
| Insert | `i`, `a`, `o`, `A` | typing text |
| Visual | `v`, `V`, `Ctrl-v` | selecting text |
| Command | `:` | saving, quitting, substitution |

When lost, press `Esc` twice and vim returns to Normal mode.

### 2.2 Navigation and editing

The home row replaces the arrow keys: `h` left, `j` down, `k` up, `l` right.
Word motions are `w`, `b` and `e`; line boundaries are `0` and `$`; the file
start and end are `gg` and `G`.

The real power comes from combining a count, an operator and a motion:

```vim
5j      5 lines down
d2w     delete two words
3dd     delete three lines
ciw     change the word under the cursor
```

Saving and quitting:

| Command | Effect |
| --- | --- |
| `:w` | write |
| `:q` | quit |
| `:wq` or `:x` or `ZZ` | write and quit |
| `:q!` | quit without saving |

### 2.3 Search and substitution

```vim
/pattern      search forward, n and N jump between matches
*             search the word under the cursor
:%s/old/new/g substitute in the whole file
:%s/old/new/gc substitute with confirmation
```

`%` means the whole file, `g` means every occurrence on a line and `c` asks
before each replacement.

### 2.4 Shell scripting

A script is a text file executed by the shell. The first line is the shebang,
which selects the interpreter:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

`set -e` exits on a failed command, `-u` rejects undefined variables, `-o
pipefail` propagates failures through a pipeline and `-E` makes the `ERR` trap
apply inside functions as well.

Variables hold strings. Quote every expansion, because an unquoted variable is
split on whitespace and may expand to nothing:

```bash
name="ROS"
echo "Hello, ${name}!"    # braces make the boundary explicit
```

Parameter expansion covers most string handling needs:

| Form | Meaning |
| --- | --- |
| `${#var}` | length |
| `${var:0:5}` | first five characters |
| `${var//a/b}` | replace every `a` |
| `${var:-default}` | default when empty or unset |
| `${var#*=}` | remove the shortest leading match up to `=` |
| `${var%%=*}` | remove the longest trailing match from `=` |

Conditions, loops and functions:

```bash
if [[ -f "$file" ]]; then
    echo "regular file"
elif [[ -d "$file" ]]; then
    echo "directory"
fi

for f in *.txt; do echo "$f"; done
for ((i = 0; i < 3; i++)); do echo "$i"; done

while IFS= read -r line; do
    echo "$line"
done < input.txt

greet() {
    local who="${1:-world}"   # local keeps the variable inside the function
    echo "Hello, $who"
}
```

Prefer `[[ ]]` over `[ ]` and `$(...)` over backticks. Read files with `while
read -r` instead of `for x in $(cat file)`, because the latter breaks on spaces
in file names.

---

## 3. Git & GitHub

### 3.1 The mental model

Git is a distributed version control system. A repository is a directed acyclic
graph of commits, where each commit is a snapshot plus a pointer to its parents.
Working directory changes flow through the staging area into a commit:

```
working directory  --git add-->  staging area  --git commit-->  commit
```

`HEAD` points to the current branch, and a branch is simply a movable pointer to
a commit.

### 3.2 First-time configuration

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
git config --global core.quotepath false   # show non-ASCII names literally
```

Configuration has three levels: `--system`, `--global` and `--local`. Local
settings override global ones.

### 3.3 Everyday workflow

```bash
git status                 # what changed, what is staged
git diff                   # unstaged changes
git add file.txt           # stage one file
git add .                  # stage everything
git diff --staged          # staged changes
git commit -m "Add notes"  # record a snapshot
git log --oneline --graph --decorate
```

A remote is a named URL. The conventional name for the main one is `origin`:

```bash
git remote -v
git remote add origin git@github.com:user/repo.git
git push -u origin main    # first push also sets the upstream
git push
git pull
```

### 3.4 Branches

```bash
git branch feature         # create
git switch feature         # create and switch (modern form)
git switch -c feature
git merge feature          # merge into the current branch
git branch -d feature      # delete after merging
```

`merge` preserves history and creates a merge commit. `rebase` replays commits
onto another base and produces a linear history, but it rewrites commit hashes,
so never rebase a branch that others already pulled.

### 3.5 Undoing mistakes

| Goal | Command |
| --- | --- |
| discard unstaged edits | `git restore file` |
| unstage a file | `git restore --staged file` |
| undo last commit, keep changes | `git reset --soft HEAD~1` |
| undo and discard changes | `git reset --hard HEAD~1` |
| revert a pushed commit safely | `git revert <commit>` |
| recover a lost commit | `git reflog` then `git reset --hard HEAD@{n}` |

`git reflog` records every movement of `HEAD`, so almost nothing is truly lost
within the default retention window.

### 3.6 Merge conflicts

A conflict appears when two branches change the same lines. Git inserts markers:

```
<<<<<<< HEAD
local version
=======
incoming version
>>>>>>> feature
```

Resolve by editing the file, deleting the markers, keeping the intended content,
then `git add` and `git commit`. To abandon the merge use `git merge --abort`.
Setting `merge.conflictstyle = zdiff3` shows the original text as well, which
makes the three versions easier to compare.

### 3.7 SSH authentication

HTTPS asks for credentials on every push, so SSH keys are the practical choice:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub      # paste this into GitHub and Gitee
ssh -T git@github.com          # verify
```

The private key (`id_ed25519`) stays on the machine with mode `600`; only the
public key (`id_ed25519.pub`) is uploaded. A passphrase protects the key at rest
and `ssh-agent` avoids retyping it:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### 3.8 Fork, branch, pull request

A pull request cannot be opened from a branch that only exists locally. The
standard contribution workflow is:

1. Fork the upstream repository on GitHub, which creates `you/repo`.
2. Clone the fork and add the original as a second remote:
   ```bash
   git clone git@github.com:you/repo.git
   git remote add upstream git@github.com:owner/repo.git
   ```
3. Create a topic branch, never work on `master` directly:
   ```bash
   git switch -c my-feature
   ```
4. Commit with a clear message, push to the fork, then open a PR against
   `owner:master` from `you:my-feature`.

Because the upstream keeps moving, a branch eventually conflicts. The correct
fix is to merge (or rebase onto) `upstream/master`, resolve the conflict, commit
and push again, which updates the existing PR automatically. Closing the PR and
opening a new one to dodge the conflict throws away the review history and is
penalized in this course.

```bash
git fetch upstream
git merge upstream/master     # or: git rebase upstream/master
# resolve conflicts, git add ... , git commit
git push
```

---

## 4. Cheat Sheet

### Linux

```bash
pwd; ls -alh; cd -              navigate
cp -r src dst; mv a b; rm -i f  copy, move, remove
find . -name "*.py"             locate files
grep -rn "pattern" .            search inside files
chmod 755 f; chown u:g f        permissions and ownership
sudo apt install pkg            install software
ps aux | grep p; kill <pid>     inspect and stop processes
systemctl status unit           service state
cmd > out 2>&1                  redirect both streams
```

### Vim

```text
i a o A       insert modes
h j k l       move
w b e         words
0 $           line start and end
gg G          file start and end
dd yy p       delete, yank, paste
u Ctrl-r      undo, redo
/ * n         search
:%s/a/b/gc    substitute
:w :q :wq :q! write, quit
```

### Shell

```bash
#!/usr/bin/env bash
set -Eeuo pipefail            safety net
var="value"; echo "${var}"    quoting
[[ -f x ]] && ...             tests
for f in *; do ...; done      loops
f() { local a="$1"; }         functions
cmd | tee log                 keep and show output
```

### Git

```bash
git status; git diff; git add -p
git commit -m "Message."
git switch -c branch          new branch
git push -u origin branch     publish
git fetch upstream; git merge upstream/master
git log --oneline --graph
git restore --staged f        unstage
git reset --soft HEAD~1       undo commit, keep work
git reflog                    recovery tool
```

---

## 5. Common Pitfalls

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Permission denied (publickey)` | public key not uploaded | add `id_ed25519.pub` to the hosting service |
| `Bad permissions: id_ed25519` | private key too permissive | `chmod 600 ~/.ssh/id_ed25519` |
| `Please tell me who you are` | identity not configured | set `user.name` and `user.email` |
| `Updates were rejected` | remote is ahead | `git pull` then push |
| `error: Your local changes would be overwritten` | dirty working tree | `git stash` or commit first |
| script exits silently with `set -e` | `((count++))` returns 1 when the result is 0 | `((count++)) || true` |
| variable disappears in a script | assigned in a subshell | avoid piping into `while`, use process substitution |
| file name with spaces split | unquoted expansion | always quote `"$var"` |
| `vi` lacks syntax highlighting | Ubuntu ships `vim-tiny` | `sudo apt install vim` |
| editted `~/.bashrc` has no effect | not sourced | `source ~/.bashrc` |
| `dpkg` fails on unrelated package | a package stuck in `half-configured` blocks every install | `sudo dpkg --configure -a` |

A recurring lesson from this lecture: the error message you see is often not
about the command you typed. Reading `journalctl`, `/var/log/apt/term.log` or
`git reflog` usually reveals the real cause one level deeper.
