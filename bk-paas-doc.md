
# 使用CentOS 7.x快速部署技术运营中台。

## 环境准备

### 1.系统初始化
```
[root@paas-node-1 ~]# cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
```
关闭SELinux、Iptables、可参考文档：http://k8s.unixhot.com/example-manual.html

### 2.安装依赖软件包
```
[root@paas-node-1 ~]# yum install -y git mariadb mariadb-server nginx supervisor \
 python-pip pycrypto gcc glibc python-devel mongodb mongodb-server
```

### 3.初始化MySQL数据库
```
[root@paas-node-1 ~]# vim /etc/my.cnf
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
[root@paas-node-1 ~]# systemctl enable mariadb && systemctl start mariadb
[root@paas-node-1 ~]# mysql_secure_installation 
[root@paas-node-1 ~]# mysql -u root -p
MariaDB [(none)]> CREATE DATABASE IF NOT EXISTS dev_paas DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
MariaDB [(none)]> grant all on dev_paas.* to paas@localhost identified by 'dev_paas';
```

### 4.初始化MongoDB数据库
```
[root@paas-node-1 ~]# systemctl enable mongod && systemctl start mongod
[root@linux-node1 ~]# netstat -ntlp | grep 27017
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN      28513/mongod
[root@linux-node1 ~]# mongo
> use cmdb
switched to db cmdb
> db.createUser({user: "cmdb",pwd: "cmdb",roles: [ { role: "readWrite", db: "cmdb" } ]});
Successfully added user: {
	"user" : "cmdb",
	"roles" : [
		{
			"role" : "readWrite",
			"db" : "cmdb"
		}
	]
}
> exit

```

### 5.克隆代码
```
[root@paas-node-1 ~]# cd /opt
[root@paas-node-1 opt]# git clone git@git.womaiyun.com:womaiyun/dev-paas.git
[root@paas-node-1 opt]# pip install virtualenv
[root@ops ~]# cd /opt/dev-paas/
[root@ops dev-paas]# mkdir paas-runtime
```

## 部署paas服务

### 1.初始化Python虚拟环境
```
# 创建Python虚拟环境
[root@ops ~]# cd /opt/dev-paas/paas-runtime/
[root@ops paas-runtime]# virtualenv paas

# 使用Python虚拟环境
[root@ops paas-runtime]# source /opt/dev-paas/paas-runtime/paas/bin/activate

# 安装依赖软件包
(paas) [root@ops paas]# cd /opt/dev-paas/paas-ce/paas/paas/
(paas) [root@ops paas]# pip install -r requirements.txt 
```

### 2.配置paas

- 修改数据库配置，可以根据需求修改域名和端口，这里保持默认。
```
(paas) [root@ops paas]# cd conf/
(paas) [root@ops conf]# cp settings_production.py.sample settings_production.py
(paas) [root@ops conf]# vim settings_production.py
(runtime-paas) [root@paas-node-1 paas]# vim conf/settings_development.py 
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'dev_paas',
        'USER': 'paas',
        'PASSWORD': 'dev_paas',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}

PAAS_DOMAIN = 'dev.womaiyun.com'
BK_COOKIE_DOMAIN = '.womaiyun.com'
```

- 进行数据库初始化（如果遇到权限问题请检查数据库授权）

```
(paas) [root@ops paas]# export BK_ENV="production"
(paas) [root@ops paas]# python manage.py migrate
```
退出python虚拟环境
```
(runtime-paas) [root@paas-node-1 paas]# deactivate
```


## 部署login服务

### 1.初始化Python虚拟环境
```
# 创建Python虚拟环境
[root@ops ~]# cd /opt/dev-paas/paas-runtime/
[root@ops paas-runtime]# virtualenv login

# 使用Python虚拟环境
[root@ops paas-runtime]# source /opt/dev-paas/paas-runtime/login/bin/activate
(login) [root@ops paas-runtime]# cd /opt/dev-paas/paas-ce/paas/login

# 安装依赖软件包
(login) [root@ops login]# pip install -r requirements.txt 
```

### 2.配置login

- 修改数据库配置，可以根据需求修改域名和端口，这里保持默认。

```
(login) [root@ops login]# cd conf/
(login) [root@ops conf]# cp settings_production.py.sample settings_production.py
(login) [root@ops conf]# vim settings_production.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'dev_paas',
        'USER': 'paas',
        'PASSWORD': 'dev_paas',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}
# cookie访问域
BK_COOKIE_DOMAIN = '.womaiyun.com'
```

- 进行数据库初始化（如果遇到权限问题请检查数据库授权）
```
(login) [root@ops conf]# cd ..
(login) [root@ops login]# export BK_ENV="production"
(login) [root@ops login]# python manage.py migrate
```
#退出python虚拟环境
```
(runtime-login) [root@paas-node-1 login]# deactivate
```

## 部署appengine服务

### 1.初始化Python虚拟环境
```
# 创建Python虚拟环境
[root@ops ~]# cd /opt/dev-paas/paas-runtime/
[root@ops paas-runtime]# virtualenv appengine

# 使用Python虚拟环境
[root@paas-node-1 opt]# source /opt/runtime-appengine/bin/activate
(runtime-appengine) [root@paas-node-1 opt]# cd /opt/bk-PaaS/paas-ce/paas/appengine/

# 安装依赖软件包
(appengine) [root@ops paas-runtime]# cd /opt/dev-paas/paas-ce/paas/appengine/
(appengine) [root@ops appengine]# pip install -r requirements.txt 
```

### 2.配置appengine

- 修改数据库配置，可以根据需求修改域名和端口，这里保持默认。
```
[root@paas-node-1 appengine]# vim controller/settings.py
(appengine) [root@ops appengine]# vim controller/settings.py 
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'dev_paas',
        'USER': 'paas',
        'PASSWORD': 'dev_paas',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

- 退出python虚拟环境
```
(appengine) [root@ops appengine]# deactivate
```


## 部署esb服务

### 1.初始化Python虚拟环境
```
# 创建Python虚拟环境
[root@ops ~]# cd /opt/dev-paas/paas-runtime/
[root@ops paas-runtime]# virtualenv esb

# 使用Python虚拟环境
[root@ops paas-runtime]# source /opt/dev-paas/paas-runtime/esb/bin/activate

# 安装依赖软件包
(esb) [root@ops paas-runtime]# cd /opt/dev-paas/paas-ce/paas/esb/
(esb) [root@ops esb]# pip install -r requirements.txt 
```

### 2.配置esb

- 修改数据库配置，可以根据需求修改域名和端口，这里保持默认。
```
(esb) [root@ops esb]# cd configs/
(esb) [root@ops configs]# cp default_template.py default.py
(runtime-esb) [root@paas-node-1 configs]# vim default.py 
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'open_paas',
        'USER': 'paas',
        'PASSWORD': 'open_paas',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

- 进行数据库初始化（如果遇到权限问题请检查数据库授权）

```
(esb) [root@ops esb]# export BK_ENV="production"
(esb) [root@ops esb]# python manage.py migrate
```
退出python虚拟环境
```
(esb) [root@ops esb]# deactivate
```


## 使用screen启动服务

   对于screen不熟悉的用户可以参考https://www.ibm.com/developerworks/cn/linux/l-cn-screen/index.html

### 启动paas
```
[root@paas-node-1 ~]# screen -t paas
[root@paas-node-1 ~]# source /opt/runtime-paas/bin/activate
(runtime-paas) [root@paas-node-1 ~]# cd /opt/bk-PaaS/paas-ce/paas/paas/
(runtime-paas) [root@paas-node-1 paas]# python manage.py runserver 0.0.0.0:8001
#按Ctrl+A+D退出screen

```
### 启动login
```
[root@paas-node-1 ~]# screen -t login
[root@paas-node-1 ~]# source /opt/runtime-login/bin/activate
(runtime-login) [root@paas-node-1 ~]# cd /opt/bk-PaaS/paas-ce/paas/login/
(runtime-login) [root@paas-node-1 login]# python manage.py runserver 0.0.0.0:8003
#按Ctrl+A+D退出screen
```

### 启动esb
```
[root@paas-node-1 ~]# screen -t esb
[root@paas-node-1 ~]# source /opt/runtime-esb/bin/activate
(runtime-esb) [root@paas-node-1 ~]# cd /opt/bk-PaaS/paas-ce/paas/esb/
(runtime-esb) [root@paas-node-1 esb]# python manage.py runserver 0.0.0.0:8002
#按Ctrl+A+D退出screen
```

### 启动appengine
```
[root@paas-node-1 ~]# screen -t appengine
[root@paas-node-1 ~]# source /opt/runtime-appengine/bin/activate
(runtime-appengine) [root@paas-node-1 ~]# cd /opt/bk-PaaS/paas-ce/paas/appengine/
(runtime-appengine) [root@paas-node-1 appengine]# python manage.py runserver 0.0.0.0:8000
#按Ctrl+A+D退出screen
```

## 使用Nignx配置访问

### 配置Nginx

```
[root@paas-node-1 ~]# cd /opt/bk-PaaS/paas-ce/paas/examples/
[root@paas-node-1 examples]# cp nginx_paas.conf /etc/nginx/conf.d/
[root@paas-node-1 ~]# vim /etc/nginx/conf.d/nginx_paas.conf 
#在location / 下面增加
location /static {
        proxy_pass http://OPEN_PAAS_LOGIN;
        proxy_pass_header Server;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_read_timeout 600;
    }
[root@paas-node-1 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@paas-node-1 ~]# systemctl start nginx
```

### 访问蓝鲸PAAS
 - 设置本地Hosts绑定
 - http://dev.womaiyun.com/
 - 默认用户名密码：admin admin
