# swoole-docker
仅为自己常用的swoole配套工具包

### 编译
```bash
docker build -t ezkuangren/laravel .
```

### 推送到hub
```bash
docker push ezkuangren/laravel:latest  
```

### 更新
```bash

```
# 启动
> 建议用一个镜像启动两个容器：测试和部署
### 拉取镜像
`docker pull ezkuangren/laravel`

#### 测试环境方便在命令行里操作：
```sh
docker run -it -p 本地端口:容器端口 -v /你的本地目录:/var/www/project --privileged=true ezkuangren/laravel /bin/bash
```
此时会出现一个容器id。

你成功的创建了一个镜像，下次再启动不需要加配置项了（端口，目录...），镜像好比是linux系统，容器是你要装的服务器，装好了之后下次启动不需要再配置各种项了。

/bin/bash 是通过命令行进入到容器，就可以在容器里启动、停止、重启swoole项目了

相关操作：

`docker start 容器id` 启动

`docker stop 容器id`  关闭

`docker rm 容器id`  删除，多个用空格间隔

`docker container update 配置 容器id` 修改

`docker rename 原容器名  新容器名` 改名字，方便找，不然默认是随机生成的


有时候，我们创建容器时忘了添加参数 --restart=always ，当 Docker 重启时，容器未能自动启动，
现在要添加该参数怎么办呢，方法有两种：

1、Docker 命令修改

docker container update --restart=always 容器名

2、直接改配置文件

首先停止容器，不然无法修改配置文件

配置文件路径为：/var/lib/docker/containers/容器ID

在该目录下找到一个文件 hostconfig.json ，找到该文件中关键字 RestartPolicy

修改前配置："RestartPolicy":{"Name":"no","MaximumRetryCount":0}

修改后配置："RestartPolicy":{"Name":"always","MaximumRetryCount":0}

最后启动容器。


#### 部署模式：

```sh
docker run -it -p 本地端口:容器端口 -v /你的本地目录:/var/www/project --privileged=true ezkuangren/laravel /bin/sh -c "你要执行的start命令"
```
如 FaShop 的启动：（还未上线，仅为演示）

.... /bin/sh -c "composer install --no-dev && composer dump-autoload -o && composer clearcache && php fashop start"

意思是安装composer的依赖包，重新加载（比如命名空间的改变），清理缓存，开启fashop的项目

# 如何安装easyswoole ? 

假设是开发模式下，通过/bash/bin进入docker 之后执行（确定在/var/www/project目录，pwd可以看当前目录），执行以下命令行完毕之后，打开浏览器访问127.0.0.1:你映射的端口，看到easyswoole的环境界面就属于正常啦。
```
composer require easyswoole/easyswoole=3.x-dev
php vendor/bin/easyswoole install
php easyswoole start
```


```

FROM php:7.4

LABEL maintainer="job@fashop.cn"

# Version
ENV PHPREDIS_VERSION 4.0.1
ENV HIREDIS_VERSION 0.13.3
ENV SWOOLE_VERSION 4.4.12

# Timezone
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo 'Asia/Shanghai' > /etc/timezone
# Libs
RUN apt-get update \
    && apt-get install -y \
    libmagickwand-dev \
    libmagickcore-dev \
    curl \
    wget \
    git \
    zip \
    libcurl4-gnutls-dev \
    libz-dev \
    libssl-dev \
    libnghttp2-dev \
    libpcre3-dev \
    && apt-get clean \
    && apt-get autoremove

# Composer
RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer \
    && composer self-update --clean-backups

# imagick gd extension
RUN pecl install imagick-3.4.3 \
    && docker-php-ext-enable imagick \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd

# PDO extension
RUN docker-php-ext-install pdo_mysql

# Bcmath extension
RUN docker-php-ext-install bcmath

# Zip extension
RUN docker-php-ext-install zip


# Redis extension
RUN wget http://pecl.php.net/get/redis-${PHPREDIS_VERSION}.tgz -O /tmp/redis.tar.tgz \
    && pecl install /tmp/redis.tar.tgz \
    && rm -rf /tmp/redis.tar.tgz \
    && docker-php-ext-enable redis

# Hiredis
RUN wget https://github.com/redis/hiredis/archive/v${HIREDIS_VERSION}.tar.gz -O hiredis.tar.gz \
    && mkdir -p hiredis \
    && tar -xf hiredis.tar.gz -C hiredis --strip-components=1 \
    && rm hiredis.tar.gz \
    && ( \
    cd hiredis \
    && make -j$(nproc) \
    && make install \
    && ldconfig \
    ) \
    && rm -r hiredis

# Swoole extension
RUN wget https://github.com/swoole/swoole-src/archive/v${SWOOLE_VERSION}.tar.gz -O swoole.tar.gz \
    && mkdir -p swoole \
    && tar -xf swoole.tar.gz -C swoole --strip-components=1 \
    && rm swoole.tar.gz \
    && ( \
    cd swoole \
    && phpize \
    && ./configure --enable-mysqlnd --enable-openssl \
    && make -j$(nproc) \
    && make install \
    ) \
    && rm -r swoole \
    && docker-php-ext-enable swoole



WORKDIR /var/www/project


```