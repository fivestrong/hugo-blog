---
title: pyenv python多版本控制工具
date: 2016-03-25 17:04:49
tags: ["pyenv"]
categories: ["python"]
---
pyenv:一个简单的python版本管理工具，它能够让你改变全局python版本，安装并同时启用多个版本，并且可以创建python虚拟环境virualenv.
它可以在linux和OS X上运行，并且无需root权限。
<!--more-->
### 安装pyenv

    $ git clone git://github.com/yyuu/pyenv.git ~/.pyenv
    $ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
    $ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
    $ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
    $ exec  $SHELL

我使用的是zsh，所以将变量导入到 `~/.zshrc` 文件而不是`~/.bash_profile`
Ubuntu的用户是 `~/.bashrc`

### 安装Python
#### 查看可安装的版本

    $ pyenv install --list

该命令会列出可以用pyenv安装的Python版本，仅列举几个:

    2.7.11           # Python 2最新版本
    3.5.1            # Python 3最新版本
    anaconda2-2.5.0  # 支持Python 2.6和2.7
    anaconda3-2.5.0  # 支持Python 5

其中形如 x.x.x 这样的只有版本号的为Python官方版本，其他的形如 xxxxx-x.x.x 这种既有名称又有版本后的属于“衍生版”或发行版。

#### 安装Python的依赖包

因为pyenv是根据源码包进行编译安装，所以可能需要用到一些其他的依赖软件包，已知的一些需要预先安装的库如下。

    sudo yum install readline readline-devel readline-static
    sudo yum install openssl openssl-devel openssl-static
    sudo yum install sqlite-devel
    sudo yum install bzip2-devel bzip2-libs

#### 安装指定版本

使用如下命令即可安装python 3.4.4：

    $ pyenv install 3.4.4 

#### 更新数据库

    $ pyenv rehash

#### 查看当前已安装的python版本

    $ pyenv versions                                                             
    * system (set by /root/.pyenv/version)
    2.7.11
    3.4.4

其中的星号表示当前正在使用的是系统自带的python。

### 更新pyenv

    $ cd ~/.pyenv
    $ git pull

### 卸载pyenv

从~/.bash_profile中移除 `pyenv init`

    rm -rf `pyenv root`

#### 移除特定版本python

    pyenv uninstall 2.7.11

或者直接删除版本所在的文件夹，使用如下命令查看：

    pyenv prefix 2.7.11

### pyenv 可用命令

常用命令如下:

* [`pyenv commands`](#pyenv-commands)
* [`pyenv local`](#pyenv-local)
* [`pyenv global`](#pyenv-global)
* [`pyenv shell`](#pyenv-shell)
* [`pyenv install`](#pyenv-install)
* [`pyenv uninstall`](#pyenv-uninstall)
* [`pyenv rehash`](#pyenv-rehash)
* [`pyenv version`](#pyenv-version)
* [`pyenv versions`](#pyenv-versions)
* [`pyenv which`](#pyenv-which)
* [`pyenv whence`](#pyenv-whence)

#### `pyenv commands`   

    列出所有可用命令

#### `pyenv local`

    设置一个当前账户可用的本地版python，它的优先级高于全局版本（即本机自带版本）

    $ pyenv local 2.7.11

    取消版本设置：

    $ pyenv local --unset

    你可以一次启用多个python版本，放在前面的版本会优先使用

    $ pyenv local 2.7.11 3.4.4 
    $ pyenv versions                                                             
    system
    * 2.7.11 (set by /root/.python-version)
    * 3.4.4 (set by /root/.python-version)
    
    $ python --version                                                           
    Python 2.7.11
    $ python2.7 --version                                                        
    Python 2.7.11
    $ python3.4 --version                                                        
    Python 3.4.4
#### `pyenv global`

    设置全局python版本替换系统自带版本（不推荐，可能会导致系统某些配置失效）

    $ pyenv global 2.7.11

    跟local版本一样，全局配置也可以同时启用多个版本，先配的优先启用。

#### `pyenv shell`

    $ pyenv shell pypy-2.2.1
    $ pyenv shell --unset

#### `pyenv install`

    安装某一个版本的python

    Usage: pyenv install [-f] [-kvp] <version>
    pyenv install [-f] [-kvp] <definition-file>
    pyenv install -l|--list

    -l/--list List all available versions
    -f/--forceInstall even if the version appears to be installed already
    -s/--skip-existingSkip the installation if the version appears to be installed already

    python-build options:

    -k/--keepKeep source tree in $PYENV_BUILD_ROOT after installation
    (defaults to $PYENV_ROOT/sources)
    -v/--verbose Verbose mode: print compilation status to stdout
    -p/--patch   Apply a patch from stdin before building
    -g/--debug   Build a debug version

#### `pyenv uninstall`

    卸载某一个版本的python

    Usage: pyenv uninstall [-f|--force] <version>

     -f  Attempt to remove the specified version without prompting
         for confirmation. If the version does not exist, do not
         display an error message.

#### `pyenv rehash`

    刷新数据库

    pyenv rehash

#### `pyenv version`

    显示当前激活的python版本
    $ pyenv version

#### `pyenv versions`

    列出所有pyenv可知版本（即通过它安装的版本），并且会显示当前激活的版本，已*标记。

#### `pyenv which`

    列出你所指定的python版本可执行文件的全路径

    # pyenv which python3.4
    /root/.pyenv/versions/3.4.11/bin/python3.4

#### `pyenv whence`

    列出指令所安装的所有版本

     $ pyenv whence 2to3 
     2.7.11
     3.4.4

#### `pyenv install --list`

    列出所有可以安装的python以及其衍生版本

    $ pyenv install --list

### 参考

 1. [https://github.com/yyuu/pyenv](https://github.com/yyuu/pyenv)

 