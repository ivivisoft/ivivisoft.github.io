---
title: 用Python实现自己的shell
excerpt: Shell 是每一个Linux操作系统必备的基本工具，让我们用Python实现自己的shell！
category:
  - Python
tags:
  - Shell
date: 2022-02-07 14:20:15
---

### 项目结构

![项目结构](项目结构.png)

### 代码实现

#### 程序入口

```python
[shell.py] []
# -*- coding: utf-8 -*-：


import sys
import shlex
import os

### 导入常量
from ysh.constants import *

### 导入所有内建函数引用
from ysh.builtins.cd import *
from ysh.builtins.exit import *


### 注册内建函数到内建命令的哈希映射中
def register_command(name, func):
    built_in_cmds[name] = func


### 在此注册所有的内建命令
def init():
    register_command('cd', cd)
    register_command('exit', exit)


### 使用哈希映射来存储内建的函数名及其引用
built_in_cmds = {}


def shell_loop():
    status = SHELL_STATUS_RUN
    while status == SHELL_STATUS_RUN:
        ### 显示命令提示符

        sys.stdout.write('$'+os.getlogin()+' '+os.getcwd() + '> ')
        sys.stdout.flush()

        ### 读取命令输入
        cmd = sys.stdin.readline()

        ### 切分命令输入
        cmd_tokens = tokenize(cmd)

        ### 执行该命令并获取新的状态
        status = execute(cmd_tokens)


def tokenize(string):
    return shlex.split(string)


def execute(cmd_tokens):
    ###从元组中分拆命令名与参数
    cmd_name = cmd_tokens[0]
    cmd_args = cmd_tokens[1:]
    ### 如果该命令是一个内建命令,使用参数调用该函数
    if cmd_name in built_in_cmds:
        return built_in_cmds[cmd_name](cmd_args)
    ### 分叉一个子shell进程
    ### 如果当前进程是子进程,期'pid'被设置为'0'
    ### 如果当前进程是父进程,'pid'的值是其子进程的进程ID
    pid = os.fork()
    if pid == 0:
        ###子进程
        ###用被exec调用程序替换该子进程
        os.execvp(cmd_tokens[0], cmd_tokens)
    elif pid > 0:
        ### 父进程
        while True:
            ### 等待其子进程的响应状态(以进程ID来查找)
            wpid, status = os.waitpid(pid, 0)
            ### 当其子进程正常退出时
            ### 或者其被信号中断时,结束等待状态
            if os.WIFEXITED(status) or os.WIFSIGNALED(status):
                break
    ### 返回状态以告知在shell_loop中等待下一个命令
    return SHELL_STATUS_RUN


def main():
    init()
    shell_loop()


if __name__ == '__main__':
    main()
```

#### 常量

```python
[constants.py] []# -*- coding: utf-8 -*-：

SHELL_STATUS_RUN = 1
SHELL_STATUS_STOP = 0
```

#### 内建命令

我们fork了一个子进程去执行命令,执行命令的过程没有发生在父进程上.比如cd命令,这样只是改变了子进程的当前目录,而没有改变父进程的.所以像这种与shell自己相关的命令必须是内置命令.他必须在shell进程中执行,而不是分叉中!

- (cd)

```python
[cd.py] []
# -*- coding: utf-8 -*-：

import os
from ysh.constants import *


def cd(args):
    if len(args) == 0:
        os.chdir(os.environ['HOME'])
    else:
        os.chdir(args[0])
    return SHELL_STATUS_RUN
```

- (exit)

```python
[exit.py] []
# -*- coding: utf-8 -*-：

import os
from ysh.constants import *

def exit(args):
    return SHELL_STATUS_STOP
```
