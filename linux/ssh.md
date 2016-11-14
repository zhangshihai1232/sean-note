## 口令登录
第一次登录的提示

```bash
The authenticity of host '10.91.40.1 (10.91.40.1)' can't be established.
RSA key fingerprint is SHA256:fMOJMxAVmiQ+UFYmsDzfF7vYFBlbCw3ltt5nF2IHRCY.
Are you sure you want to continue connecting (yes/no)?
```
无法认证的主机(kown_hosts中没有保存)，是否需要继续连接；
公钥长度较长，比如RSA算法1024位，使用MD5计算出128位的指纹，这样容易比对,陌生的指纹可能存在风险

```bash
Warning: Permanently added '10.91.40.1' (RSA) to the list of known hosts.
```
输入密码之后

```bash
Warning: Permanently added 'host,12.18.429.21' (RSA) to the list of known hosts.
```
远程公钥会保存在known_hosts下，系统中`/etc/ssh/ssh_kown_hosts`保存一些所有用户都可以信赖的主机；
再次连接主机，会跳过警告；

## 公钥认证
**session key生成**
1. 客户端请求连接服务器，服务器将public key和会话id发送给客户端
2. 客户端生成秘钥，利用会话id对这个秘钥处理，产生加密结果r，并把r用服务器公钥加密，发送给服务器
3. 服务器解密获得r，利用会话id处理，获得客户端公钥
4. 这样双方就都知道了对方的公钥

认证前，客户端需要将公钥上传到服务器上，并配置到服务器中；
**认证**
1. client发送一个认证的id(session key)到server；
2. server检查authorized_keys，若找到id对应的public key,生成一个随机数，利用这个public key加密,并把加密后的信息发送给client
3. 如果client有这个public key对应的private key，会利用private key解密，获取到数字；
4. clinet使用共享的session key加密，并通过MD5加密后，发送给server
5. server使用同一个共享session key解密，获得MD5，再解密获得值，比较与最初生成的数字是否一致，如果一致认证成功；

authorized_keys权限需要`644`
## 信任关系
公钥
证书：public_key_auth

http://www.ibm.com/developerworks/cn/aix/library/au-sshsecurity/
http://jiangym19860710.blog.163.com/blog/static/56153248201301211201826/
man 5 sshd_config
man ssh
http://forlinux.blog.51cto.com/8001278/1352900

## 免密登录
A登录B
A产生RSA key
```bash
ssh-keygen -t rsa -P "
```
把A的公钥添加到B的authorized_keys
修改权限
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys  
```