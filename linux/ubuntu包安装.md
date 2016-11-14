## 包管理
起初，linux系统中只有.tar.gz，用户需要自己编译程序；
Debinan后，dpkg机制管理软件包，软件称为package；
红帽子建立自己的包管理系统rpm;
包之间有大量的依赖关系，通过APT（Advanced Packaging Tool）管理，被Conectiva移植到红帽子上管理rpm，其他系统也有移植；
aptitude删除包本身时，会删除依赖的包，所以系统会更加干净；

## dpkg
debian用来安装、删除、构建和管理Debian的软件包
提供一些子命令

```bash
dpkg-deb
dpkg-divert
dpkg-query
dpkg-split
dpkg-statoverride
start-stop-daemon
```
deb命令规则
- 软件包名称(Package Name)
- 版本(Version Number)
- 修订号(Build Number)
- 平台(Architecture)
	- i386
	- all: 平台无关. 即适用于所有平台.比如文本, 网页, 图片, 媒体, pdf 等
比如：nano_1.3.10-2_i386.deb
- 软件包名称: nano
- 版本: 1.3.10
- 修订号: 2
- 平台: i386

dpkg绕过apt包管理数据库对软件包进行操作
## apt-get
..

## aptitude
aptitude更好的地方

```bash
install
remove
reinstall（apt-get无此功能）
show（apt-get无此功能
search（apt-get无此功能）
hold（apt-get无此功能）
unhold（apt-get无此功能）
```
apt-get 解决得更好的地方
 
```bash
source（aptitude无此功能）
build-dep（低版本的aptitude没有build-dep功能）
```
apt-get 跟 aptitude 没什么区别的地方

```bash
update
upgrade (apt-get upgrade=aptitude safe-upgrade, apt-get dist-upgrade=aptitude full-upgrgade)
```