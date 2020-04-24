## ssh免密登陆

使用密码登录，每次都必须输入密码，非常麻烦。好在SSH还提供了公钥登录，可以省去输入密码的步骤。
所谓"公钥登录"，原理很简单，就是用户将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

#### 1、产生公私钥
ssh-keygen -t rsa -b 2048 -f xx/.ssh/id_rsa -P ""
这样会在xx/.ssh/目录下产生id_rsa(私钥文件)、id_rsa.pub(公钥文件)

#### 2、authorized_keys文件
路径：.ssh/authorized_keys
私钥认证文件，里面保存的是ssh客户端的公钥内容
可以追加多个ssh的客户端的公钥内容，比如：
```sh
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCiCqSZlpb7c6OXx6cXJBNDDomIvlbLQKGNGmgk7j0Ki1GdTcq4IAl3biB2y0UitIQlHMx4nVylIC+XIMz24rVak4CoBY3CRk20AiuXm3Ql2KacD7LaBXxpd/jR1eRe4C9Vc4qW9ntLh9w9ZFD4oPhCFTRGjxtCqkWoObn22d/egnjyTPAcxrZJkdEKsc9EZldx6pw163jengU344WwaIWQKWwaMLGD/e9UuaOHFnaI/nqSiErw3IPACCIqzb/gw/at3CGGF2uFJTOGbXyJalyFd4lFy4ufwCaGPyTowbnSe9UiomQLNtjjutehOe34D6QzfPVTBDjjsvB2/kiuek7r ts@client1
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDmr9YeRMlvafpV/kW9pj6XWmSXE8orazgo8rO+Xu81Ye4M/fPU1Jfh3cJRxigZ/FCTkDSbES5Fy/UYZ2e/ZVykmXDAWfXxIVBhv20Uc7FgQHN1yWTKfn6fp1jy1dh2IXqNph8biI4VLIkvqr4l7uYC9BgKeBbZTGd54+2dRg/zKrsVoHV9Ns8u3BfFGqmGNcdRIUwHTv9HxZ+WiadxcZrOuLtCXmKASUp1QLgGGwzIVGuTFw6cqoPXLfKGuQBRsRHga8y4M+/25zrIE6LvPkB63jbEKyDOttCOGZyr2hm5rRAqweB80sBBk7+UJyODsgnLKHbdjqjNA0VdMRnao9Kn ts@client2
```
其中client1/client2为ssh客户端主机名

所以，如果想要某个ssh客户端免密登陆到某个ssh服务器，就需要将该客户端的id_rsa.pub内容追加到该服务器的authorized_keys中

注：注意./ssh目录及文件权限，chmod 700 ./ssh ；chmod 600 ./ssh/authorized_keys

#### 3、访问
做了以上步骤之后，这时客户端就可以免密登陆服务器了，使用如下命令：
```sh
ssh -v -i xx/./ssh/id_rsa ts@x.x.x.x
```

#### 4、安全
在实际的项目中，为了安全起见，一般还会将xx/.ssh/id_rsa进行加密保存，在使用的时候再解密


#### 5、执行命令
远程之后想切换root执行命令的话，比如
```sh
ssh -v -i xx/./ssh/id_rsa ts@x.x.x.x "sudo /usr/bin/cp xxx xxx"
```
还需要再/etc/sudoers.d/[user]文件中添加对该命令的免密说明
```sh
[user] ALL = (root) NOPASSWD:/usr/bin/cp
```


注：-i identity_file也适用于scp等命令