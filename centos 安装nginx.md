# centos 安装nginx

##### 先安装 yum

```c
yum -y install gcc gcc-c++ autoconf pcre-devel make automake
yum -y install wget httpd-tools vim
```

##### 建立相关测试文件夹

```c
mkdir
```

##### yum安装Nginx

先检查yum是否存在，命令

```c
yum list | grep nginx
```

![image-20201020162914452](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20201020162914452.png)

如果不存在，或者想升级版本

```c
vim /etc/yum.repos.d/nginx.repo
```

然后放入以下代码

```c
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

##### 再一次检查

```c
yum list | grep nginx
```

##### 然后安装

```c
sudo yum install nginx
```

```c
nginx -s reload 
## 重载
```

##### 查看所有开启的端口

```c
netstat -tlnp
```

##### nginx允许及不允许访问

```c
允许 allow   不允许 deny (全部加all)
 
deny 10.5.81.23

allow 10.5.81.33
deny  all
```

