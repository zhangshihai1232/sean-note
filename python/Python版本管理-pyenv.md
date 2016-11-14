## virtualenv
virtualenv用于创建独立的Python环境，多个Python相互独立，互不影响：
1. 在没有权限的情况下安装新套件
2. 不同应用可以使用不同的套件版本
3. 套件升级不影响其他应用

## pyenv
切换python方式
1. 设置PATH 
2. 脚本前写入`#!/usr/bin/env python2.7`

python可以一键（命令）切换全局、本地或当前 shell 使用的 Python 版本
在$PATH最前面添加了垫片路径（shims）,比如

```bash
~/.pyenv/shims:/usr/local/bin:/usr/bin:/bin
```
Python可执行文件，会在这里被截获

## 依赖

```bash
sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm
```
## 安装
自动安装

```bash
curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
```
给pyenv添加环境变量`~/.bashrc`

```bash
export PATH="/home/sean/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

## 命令

```bash
$ pyenv versions  # 列出系统当前已安装的版本
$ pyenv install python_version   # 安装指定版本 python_version
$ pyenv global 3.4.0  # 设置全局的 Python 版本，通过将版本号写入 ~/.pyenv/version 文件的方式。
$ pyenv rehash  # 为所有已安装的可执行文件 （如：~/.pyenv/versions/*/bin/*） 创建 shims，因此，每当你增删了 Python 版本或带有可执行文件的包（如 pip）以后，都应该执行一次本命令
$ pyenv local  ## 设置面向程序的本地版本，通过将版本号写入当前目录下的 .python-version 文件的方式。通过这种方式设置的 Python 版本优先级较 global 高。pyenv 会从当前目录开始向上逐级查找 .python-version 文件，直到根目录为止。若找不到，就用 global 版本。
$ pyenv shell  # 设置面向 shell 的 Python 版本，通过设置当前 shell 的 PYENV_VERSION 环境变量的方式。这个版本的优先级比 local 和 global 都要高。--unset 参数可以用于取消当前 shell 设定的版本。
$ pyenv shell pypy-2.2.1
$ pyenv shell --unset
```