## 用户相关

```bash
##添加用户 test
adduser test
##修改test密码
passwd test
##删除用户test
userdel test
##删除用户以及用户目录
userdel -f test
```

## 赋予root权限
修改`/etc/sudoers`
在下面的位置添加一行

```bash
root    ALL=(ALL)     ALL
sean   ALL=(ALL)     ALL
```

## 其他
**修改/home路径**
指定路径

```bash
auseradd -d /home/goal floatboat
```
**加入组**
默认会创建和用户同名的组，如果直接加入已存在的组

```bash
useradd -g webusers floatboat
```
也可以指定加入多个组

```bash
useradd -G ftpusers,webusers floatboat
```

## 图形界面网卡开启
`/etc/sysconfig/network-scripts/ifcfg-eth0`文件中
ONBOOT设置为yes即可

## 安装vmwaretools
挂载光驱

```bash
mount -t iso9660 -o ro /dev/cdrom /mnt 
```