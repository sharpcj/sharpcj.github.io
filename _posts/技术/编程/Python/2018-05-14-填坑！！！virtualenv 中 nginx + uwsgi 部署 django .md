---
layout: post
title: "填坑！！！virtualenv 中 nginx + uwsgi 部署 django"
date:  2018-05-14 23:25:12 +0800
categories: ["技术", "编程", "Python"]
tag: ["Python", "django", "nginx", "uwsgi"]
---

# 一、为什么会有这篇文章
第一次接触 uwsgi 和 nginx ,这个环境搭建，踩了太多坑，现在记录下来，让后来者少走弯路。
本来在 Ubuntu14.04 上 搭建好了环境，然后到 centos7.4 就遇到了一堆问题。下面把步骤记录下来，中间会记录遇到的问题及解决方案。

# 二、开发环境搭建
## 安装 python3
我的 centos7.4 预装了 python2.7.5 ，首先安装 python3，这里我选择 python3.4。
添加epel源：

```
yum install epel-release
```

安装Python3.4

```
yum install python34
```

## 安装 pip3
```
yum install python34-setuptools
easy_install-3.4 pip
```

## 安装 django
版本选择：
Django 1.5.x 支持 Python 2.6.5 Python 2.7, Python 3.2 和 3.3
Django 1.6.x 支持 Python 2.6.X, 2.7.X, 3.2.X 和 3.3.X
Django 1.7.x 支持 Python 2.7, 3.2, 3.3, 和 3.4 （注意：Python 2.6 不支持了）
Django 1.8.x 支持 Python 2.7, 3.2, 3.3, 3.4 和 3.5. （长期支持版本 LTS)
Django 1.9.x 支持 Python 2.7, 3.4 和 3.5. 不支持 3.3 了
Django 1.10.x 支持 Python 2.7, 3.4 和 3.5
Django 1.11.x 支持 Python 2.7, 3.4, 3.5 和 3.6（长期支持版本 LTS) 最后一个支持 Python 2.7 的版本
Django 2.0.x 支持 Python 3.4, 3.5 和 3.6 （注意，不再支持 Python 2）

安装 django 1.11.13 ：

```
pip3 install Django==1.11.13
```

验证 django 是否安装成功，终端上输入 python3 ,点击 Enter，进入 python3 环境：

```
>>> import django
>>> django.VERSION
(1, 11, 13, 'final', 0)
>>> django.get_version()
'1.11.13'
```

## 安装 Virtualenv (虚拟环境依赖)
### 为什么要安装虚拟环境依赖
在开发Python应用程序的时候，我系统安装的 Python3 只有一个版本：3.4。所有第三方的包都会被pip3 安装到 Python3 的 site-packages 目录下。如果我们要同时开发多个应用程序，那这些应用程序都会共用一个Python3 ，就是安装在系统的Python 3。如果应用A应用需要 django1.11，而应用B需要 django 2.0 怎么办？
这种情况下，每个应用可能需要各自拥有一套“独立”的Python运行环境。virtualenv 就是用来为一个应用创建一套“隔离”的Python运行环境。
virtualenv 用的时候参数比较复杂，本文不细说了，可以上网搜索了解一下，这里在再安装 virtualenvwrapper ，顾名思义，virtualenvwrapper 就是对 virtualenv 的一个包装，让其用起来更简单。

### pip安装虚拟环境依赖

```
pip3 install virtualenv
pip3 install virtualenvwrapper
```
### 配置环境变量
修改 `~/.bashrc` 配置环境变量：
centos7.4 中：

```
if [ -f /usr/bin/virtualenvwrapper.sh ]; then
  export WORKON_HOME=$HOME/virtualenvs
  export PROJECT_HOME=$HOME/workspace
  export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
  source /usr/bin/virtualenvwrapper.sh
fi
```

ubuntu 14.04中：

```
if [ -f /usr/local/bin/virtualenvwrapper.sh ]; then
  export WORKON_HOME=$HOME/.virtualenvs
  export PROJECT_HOME=$HOME/workspace
  export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
  source /usr/local/bin/virtualenvwrapper.sh
fi
```

修改后使之立即生效(也可以重启终端使之生效)：

```
source ~/.bashrc
```

这是第一个坑：在 Ubuntu 14.04 中，`virtualenvwrapper.sh` 文件路径和 centos7.4 中不一样，这个坑很容易发现，因为下面，你执行命令的时候会报错，找不到文件，这个坑容易填。

### 虚拟环境使用方法
`mkvirtualenv env1`：创建运行环境 env1
`workon env1`: 工作在 env1 环境 或 从其它环境切换到 env1 环境
`deactivate`: 退出终端环境
其它的：
`rmvirtualenv ENV`：删除运行环境ENV
`mkproject mic`：创建mic项目和运行环境mic
`mktmpenv`：创建临时运行环境
`lsvirtualenv`: 列出可用的运行环境
`lssitepackages`: 列出当前环境安装了的包

创建的环境是独立的，互不干扰，无需 sudo 权限即可使用 pip 来进行包的管理。

# 三、部署环境搭建
## 安装 nginx
### 安装 nginx
ubuntu 中：

```
apt-get install nginx
```

centos中：

```
yum install epel-release
yum install nginx
```

### 验证 nginx 是否安装成功
查看 nginx 配置文件：

```
cat /etc/nginx/nginx.conf
```

截取部分内容如下：

![](/assets/images/技术/编程/python/部署%20django/pic1.png)

截图第二行，配置了 nginx 错误日志保存地址，

重点关注 http 下的 server 中 listen 显示了默认监听的端口是 80 ，可以修改端口号。
server 上面有一行： `include /etc/nginx/conf.d/*.conf;`，这样我们可以将自定义的配置文件，放到 `/etc/nginx/conf.d/` 目录下，以 `.conf` 后缀命名即可。

验证 nginx 是否安装成功,则启动 nginx ：

```
service start nginx
```

通过浏览器访问 该 ip 80 端口,能正确返回下面页面,说明 nginx 安装成功。

![](/assets/images/技术/编程/python/部署%20django/pic2.png)

坑：网上收到很多资料启动 nginx 的命令：`nginx start,nginx -s start`, `nginx -s reload`。可能是 nginx 版本不同原因，本人亲测，启动、关闭、重新加载 nginx 服务命令如下：

```
service start nginx
service stop nginx
service restart nginx
```

** 追加：**
```
systemctl start nginx
```

## 安装 uwsgi
### 安装 uwsgi
python 安装 uwsgi 方法有很多，但是也有坑。官方文档上说：
要构建uWSGI，您需要Python和一个C编译器（gcc并且clang受支持）。根据您希望支持的语言，您将需要他们的开发头文件。在Debian / Ubuntu系统上，您可以安装它们（以及构建软件所需的其他基础架构），具体如下：

首先安装依赖文件：
Ubuntu 中：

```
apt-get install build-essential
apt-get install python-dev
```

centos中：

```
yum install epel-release
yum groupinstall "Development Tools"
yum install python-devel
```

坑：上面的命令针对 python2，对应 python3 中，Ubuntu 中应该改成是 `apt-get install python3-dev`，而 centos 中，`yum install python3-devel` 会找不到 python3-devel 包，网上有人说执行 `yum install -y python3-devel.x86_64`,同样也找不到包。同时应注意， ubuntu 和 centos 中包名也不一样。解决方法:
[How to install python3-devel on red hat 7](https://stackoverflow.com/questions/43047284/how-to-install-python3-devel-on-red-hat-7)
所以针对 python3：
ubuntu 中：

```
apt-get install python3-dev
```

centos 中：
先执行：`yum search python3 | grep devel`,

```
python34-cairo-devel.x86_64 : Libraries and headers for python34-cairo
python34-greenlet-devel.x86_64 : C development headers for python34-greenlet
python34-devel.x86_64 : Libraries and header files needed for Python 3
                      : development
python34-gobject-devel.x86_64 : Development files for embedding Python 3.4
python36-devel.x86_64 : Libraries and header files needed for Python development
shiboken-python34-devel.x86_64 : Development files for shiboken
```

然后依次执行：

```
yum install -y python34-cairo-devel.x86_64
yum install -y python34-greenlet-devel.x86_64
yum install -y python34-devel.x86_64
yum install -y python34-gobject-devel.x86_64
yum install -y shiboken-python34-devel.x86_64
```

我把 python3.4 相关的包都安装了。

- 下载源码编译：

```
wget https://projects.unbit.it/downloads/uwsgi-latest.tar.gz
tar zxvf uwsgi-latest.tar.gz
cd <dir>
make
```

坑：上面步骤也是对应于 python2， 对于 python3 在 make 之前，一定要先执行：

![](/assets/images/技术/编程/python/部署%20django/pic3.png)

原因是 python-devel 对应的 python3 依赖包没有安装。如果不巧，你刚好没有执行这个命令，就直接编译，并且通过了，则相当于，到时候，会出现 uwsgi 执行时找不到 module 或者 app ,
诸如 "No module named site " 或者下面信息之类的错误。

```
cannot open shared object file: No such file or directory
unable to load app 0
```

- pip3 安装（推荐）

```
pip3 install uwsgi
```

如果出现错误，则是依赖包没安装好，见上面下载源码编译的坑。

### 验证 uwsgi 是否安装成功
下面的来自 uwsgi 官方文档：

我们从一个简单的“Hello World”例子开始：

```
def application （env ， start_response ）：
    start_response （'200 OK' ， [（'Content-Type' ，'text / html' ）]）
    return  [ b “Hello World” ]
```

（保存为foobar.py）。
如您所见，它由一个Python函数组成。它被称为“应用程序”，因为这是uWSGI Python加载程序将搜索的默认函数（但您明显可以自定义它）。
部署HTTP端口9090上
现在启动uWSGI运行一个HTTP服务器/路由器，将请求传递给你的WSGI应用程序：

```
uwsgi --http：9090 --wsgi-file foobar.py
```

就这样。

下面通过浏览器访问 该 ip 80 端口,能正确返回“ Hello World”。
**注意**：如果前面没有成功安装 python3 相关的依赖包，这里也能正确访问。但是部署 django 网站时会出错。

如果出现下面错误：

```
your processes number limit is 16384
your memory page size is 4096 bytes
detected max file descriptor number: 65536
lock engine: pthread robust mutexes
thunder lock: disabled (you can enable it with --thunder-lock)
probably another instance of uWSGI is running on the same address (:8080).
bind(): Address already in use [core/socket.c line 769]
```

则是 uwsgi 启动太多次了，可以用命令杀掉这个端口在重启：

```
sudo fuser -k 8080/tcp
```

(用自己配置的端口号)

# 四、virtualenv + nginx + uwsgi 部署 django 网站
如果前面的步骤都没问题了，这一步只要把配置文件写正确，就没什么问题了。记得在虚拟环境中安装所有的 project 需要依赖包。
下面是一个简单示例：

## uwgsi 配置
uwsgi 支持多种形式的配置，可以执行 uwsgin 直接带参数，可以用 xml 文件配置等等，这里用 ini 文件配置。
新建一个 ini 文件，比如命名为：`dj_uwsgi.ini`

```
[uwsgi]
socket = :9090
chdir = /home/sharpcj/PythonProjects/testsite     # 切换到 project 目录
env = DJANGO_SETTINGS_MODULE=testsite.settings    
module = testsite.wsgi
master = true
wsgi-file = testsite/wsgi.py
home = /home/sharpcj/.virtualenvs/testsite_evn   # python 及相关依赖包所在 path

touch-file = testsite/wsgi.py

processes = 4
threads = 2

chmod-socket = 664
vacuum = true
```

注意：为了配合 nginx 工作，端口协议是 socket , 如果换成 http ,则可以直接运行uwsgi，就能通过浏览器访问页面了。命令为 `uwsgi dj_uwsgi.ini`。

## nginx 配置
在 `/etc/nginx/conf.d/` 下创建 `testsite.conf` 文件：

```
server {
    listen         9999; 
    server_name    127.0.0.1 
    charset UTF-8;
    access_log      /var/log/nginx/myweb_access.log;
    error_log       /var/log/nginx/myweb_error.log;

    client_max_body_size 75M;

    location /media  {
        alias /home/sharpcj/PythonProjects/testsite/media;
    }
 
    location /static {
        alias /home/sharpcj/PythonProjects/testsite/static;
    }
 
    location / {
        uwsgi_pass  127.0.0.1:9090;
        include     /etc/nginx/uwsgi_params;
    }
}
```

listen 指定的是nginx代理uwsgi对外的端口号。

server_name 网上大多资料都是设置的一个网址（例，www.example.com），我这里如果设置成网址无法访问，所以，指定的到了本机默认ip。在进行配置的时候，我有个问题一直想不通。nginx到底是如何uwsgi产生关联。现在看来大概最主要的就是这两行配置。

```
include uwsgi_params;

uwsgi_pass 127.0.0.1:9090;
```

`include` 必须指定为 `uwsgi_params` ；而 `uwsgi_pass` 指的本机IP的端口号与 `dj_uwsgi.ini` 配置中的文件中的必须一致。

此时启动 nginx 服务，并启动 uwsgi 服务，即可通过 ip:9999 访问网站。
通过这个IP和端口号的指向，请求应该是先到 nginx 的。如果你在页面上执行一些请求，就会看到，这些请求最终会转到 uwsgi 来处理。

ps： 这个过程本应不算复杂，前天花了一下午时间没搞定，昨天又花了一下午时间才搞定。网上搜到的文章比较乱，有些太简单的看不懂，有些又太啰嗦的不知道核心的几步是什么，有些又因为版本不对，或者环境不同，不能成功，希望本文能帮到后面的人。