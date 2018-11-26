---
title: Centos 7 下部署Django + uWSGI + Nginx
date: 2018-10-16 16:00
tags: [web,Python,Django,nginx]
---

<!-- # Centos 7 下部署Django + uWSGI + Nginx -->

## 环境：
Python: 3.6

Django: 2.1

OS: CentOS 7 x86_64

uwsgi: 2.0.17


<!-- more -->

## 目录 :

* [安装安装Python3.6](#安装Python3.6)
    * [创建虚拟环境](#创建虚拟环境)
    * [安装uWSGI](#安装uWSGI)
* [安装Nginx](#安装Nginx)
* [配置项目](#配置项目)
    * [安装django项目依赖的包](#安装django项目依赖的包)
    * [在项目目录下配置uwsgi启动django的参数](#在项目目录下配置uwsgi启动django的参数)
    * [配置项目文件](#配置项目文件)
* [配置nginx](#配置nginx)
* [重启Nginx](#重启Nginx)
* [uwsgi启动django](#uwsgi启动django)
* [代理](#代理)
* [问题汇总](#问题汇总)

## 安装Python3.6

* 不要删除自带的python2.7，否则会出问题，因为centos许多软件需要依赖系统自带python
* 安装依赖工具 yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel
* 下载 wget https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz
* 解压 tar -zxvf Python-3.6.5.tgz
* 移动至规范的放软件的目录下 mv Python-3.6.5 /usr/local
* 安装：
```bash
cd /usr/local/Python-3.6.5/

./configure

make &amp; make install
```

* 验证
* python -V

### 创建虚拟环境

* python3.6 -m venv /home/cosmic/py3.6env
* source /home/cosmic/py3.6env/bin/activate   进入虚拟环境

### 安装uWSGI

* 首先进入虚拟环境,在虚拟环境下安装uwsgi
* 安装 pip install uwsgi 
* 验证 

```bash
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello Django"]
```
```bash
uwsgi --http :8001 --wsgi-file test.py
```
浏览器访问，网页能显示 Hello Django 那么就没问题
* 如果安装失败(可能未安装依赖环境python-devel)
* deactivate 退出虚拟环境
* yum install -y python-devel 
* easy_install uwsgi

## 安装Nginx

* 配置源
    vi /etc/yum.repos.d/nginx.repo 添加下面内容
    ```bash
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/mainline/centos/7/x86_64/
    gpgcheck=0
    enabled=1
    ```
    
   gpkcheck=0 表示对从这个源下载的rpm包不进行校验；
   enable=1 表示启用这个源。

 * yum install nginx
 * 启动nginx：
   systemctl start nginx
   
 * 修改默认端口号（默认为80）
    ```bash
    vim /etc/nginx/conf.d/default.conf


    server {
        listen       8089;
        listen [::]:8089;
        ...
        ...
    }
    ```
    
 * systemctl restart nginx 重启nginx，直接访问http://ip:8089 能看到nginx的欢迎界面即可。
 
## 配置项目

### 安装django项目依赖的包

* 虚拟环境下

```
    Django==2.1.2
    django-haystack==2.8.1
    mysqlclient==1.3.13
    pytz==2018.5
    uWSGI==2.0.17.1
    Whoosh==2.7.4

    blablabla...
```

### 在项目目录下配置uwsgi启动django的参数
 
```bash
vim django_uwsgi.ini # 名称可自定义,用于启动django项目

[uwsgi]
# 通过uwsgi访问django需要配置成http
# 通过nginx请求uwsgi来访问django 需要配置成socket
# 9000 是django的端口号
socket = 0.0.0.0:9000

# web项目根目录
chdir = /home/root/pydj/django_one

# module指定项目自带的的wsgi配置文件位置
module = django_one.wsgi

# 允许存在主进程
master = true

# 开启进程数量
processes = 3

# 服务器退出时自动清理环境
vacuum = true
```

### 配置项目文件

> 配置项目下的`settings.py`文件

```python
...
DEBUG = False  # 关闭调试模式

ALLOWED_HOSTS = ["*"]  # 允许访问的主机
...
# 数据库配置
DATABASES = {

    'default': {

        'ENGINE': 'django.db.backends.mysql',

        'NAME': 'fresh',

        'HOST':'192.168.98.1',  #数据库服务器的地址

        'USER':'root',

        'PASSWORD':'123456',

        'PORT':3306
    }
}
...
# 静态资源配置
STATIC_URL = '/static/'   # 默认

# STATIC_ROOT用于收集项目下静态资源,STATICFILES_DIRS和STATIC_ROOT不能共存,注销STATICFILES_DIRS
#STATICFILES_DIRS = [

#     os.path.join(BASE_DIR,'static')

# ]

STATIC_ROOT = os.path.join(BASE_DIR, "static/")
...
```
> 配置完settings.py文件后在项目根目录下执行: python manage.py collectstatic
> 用于收集静态文件,执行后在项目根目录下的static文件夹下生成一个admin的文件夹

## 配置nginx

```bash
vi /etc/nginx/nginx.conf

user root ; # 以root用户启动nginx

...
```
 
```bash
vi /etc/nginx/conf.d/default.conf

# 在文件最后，新加一个server 或者在同级目录下添加xxx.conf文件,在其中添加server
# 可查看/etc/nginx/nginx.conf便于理解

server {
    listen       8089;
    listen      [::]:8089;
    server_name 127.0.0.1 192.168.10.114; 

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:9000;
    }
    location /static{
        alias /home/root/pydj/django_one/sign/static;
    }

}
```
 
* 8089 是对外的端口号
* server_name nginx代理uwsgi对外的ip,192.168.10.114为nginx服务器所在的ip
* 127.0.0.1:9000 即当nginx服务器收到8089端口的请求时，直接将请求转发给 127.0.0.1:9000

## 重启Nginx

systemctl restart nginx

secrtcrt
filiza
xshell

## uwsgi启动django

> 临时关闭防火墙

```bash
    systemctl stop firewalld.service
```

> 如果在虚拟机中部署测试项目时,主机访问时可能需要在控制面板中关闭防火墙

> 注意先进入虚拟环境下,这样创建的venv虚拟环境才能生效

```bash
# 进入项目根目录
/home/root/pydj/django_one

# 启动
uwsgi --ini django_uwsgi.ini
```

## 代理

> 配置nginx配置文件,在`/etc/nginx/conf.d/default.conf`追加如下内容:

```bash
upstream fresh{
        # 设置多个uwsgi代理端口
        server 192.168.52.193:9000;
        server 192.168.52.210:9000;
        server 127.0.0.1:9000;
}
server{
        listen  8009;  # nginx监听端口
        location / {
                include /etc/nginx/uwsgi_params;
                # 应用uwsgi代理端口
                uwsgi_pass fresh;
        }
        location /static{
                alias /home/zy/fresh/fresh/static;
        }
}
```
* 然后访问'nginx服务器ip:8009/...',端口为nginx服务监听的端口,此时轮询每个uwsgi服务.


## 问题汇总

> 启动时切换root用户提权
    
    su     # 切换root用户

    su - zy  # 切换普通用户

> 无法连接数据库,可能数据库未正确安装

    yum install mysql-devel gcc gcc-devel python-devel

    semanage port -l | grep http_port_t

> 配置mysql可以被局域网任意主机访问; 1130错误 提示主机没有访问数据库权限; 1045错误 提示主机拒绝访问

```
mysql&gt; use mysql; 
mysql&gt; update user set host = &#039;%&#039; where user = &#039;root&#039;; 
mysql&gt; select host, user from user; 
mysql&gt; flush privileges;
```

> [emerg] 31879#31879: bind() to 0.0.0.0:8089 failed (13: Permission denied)

绑定端口失败,端口占用或者不支持或者selinux权限控制
```
sudo semanage port -l | grep http_port_t      # 查看可用端口

sudo semanage port -a -t http_port_t  -p tcp 8024    # 可将自定义端口加入其中

netstat -lnp|grep 88 , lsof -i : 8000  , ps --help    # 查看端口状态

kill -9 1777     # 指定pid杀掉进程
```

> 样式文件失效

Selinux 控制访问权限,默认严格模式,通过以下命令临时放宽访问权限
```
# 临时关闭:

[root@localhost ~]# getenforce
Enforcing

[root@localhost ~]# setenforce 0

[root@localhost ~]# getenforce
Permissive

# 永久关闭

[root@localhost ~]# vim /etc/sysconfig/selinux

SELINUX=enforcing 改为 SELINUX=disabled

重启服务reboot
```

> 部署大致流程

* 修改setting文件之后
* python manage.py collectstatic 收集admin静态文件
* 修改uwsgi.ini
* 启动
*  uwsgi --ini django_uwsgi.ini   --buffer-size 32768
* 添加nginx配置文件
* 重启nginx