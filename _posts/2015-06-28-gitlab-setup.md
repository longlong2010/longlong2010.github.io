---
title: 安装配置Gitlab
layout: post
---

> Gitlab是一个用Ruby on Rails实现的开源的并且基于Git的版本管理系统，基本上实现了一个类似于Github的相关功能——更够通过网页浏览代码，并控制代码库的访问修改权限，提交缺陷说明和代码注释，便于开发团队之间的协作。相比Phabricator，其更加注重的是对于代码库的管理。由于其是用Ruby on Rails实现，因此对于只熟悉PHP环境的人来说，安装和配置要先得复杂不少，这里简单记录一下，安装基于Archlinux。

#### 1. 安装配置Ruby环境

> 一般开源的系统对于相关运行环境的版本都有较高的要求，Ruby要求至少要在1.9以上，如果使用Centos等软件版本较旧的Linux发行版，则需要从源代码编译Ruby或手段安装相关的软件包。安装完成后修改Ruby的gem源到淘宝的源，并安装bundler。
>
```
pacman -S ruby
gem sources -r https://rubygems.org/
gem sources -a http://ruby.taobao.org/
gem install --no-user-install bundler
```

#### 2. 下载编译Gitlab源代码

> 官方的下载地址为[https://gitlab.com/gitlab-org/gitlab-ce](https://gitlab.com/gitlab-org/gitlab-ce)，Github的下载地址为[https://github.com/gitlabhq/gitlabhq](https://github.com/gitlabhq/gitlabhq)，直接git clone即可。
>
```bash
pacman -S git
git clone https://gitlab.com/gitlab-org/gitlab-ce.git gitlab
git clone https://gitlab.com/gitlab-org/gitlab-shell.git
```
> 完成后需要修改一下代码中的gem源，同样修改到淘宝，即将gitlab中Gemfile文件的第一行地址修改为http://ruby.taobao.org/，并安装相关的依赖软件包，完成后开始编译和安装gitlab依赖的相关gem
>
```bash
pacman -S gcc make patch cmake pkg-config
pacman -S icu
pacman -S mysql
cd gitlab && bundle install --deployment --without development test postgres aws
```

#### 3. 配置gitlab-shell

> 首先需要为gitlab创建一个系统用户，如果在安装了git后自动创建了用户则需要修改其home目录，并将gitlab和gitlab-shell移动到其home目录下，并修改权限
>
```bash
useradd --system --create-home --comment 'GitLab' git
mv gitlab /home/git/
mv gitlab-shell /home/git/
chown git:git -R /home/git/gitlab
chown git:git -R /home/git/gitlab-shell
cd /home/git
cp gitlab-shell/config.yml.example gitlab-shell/config.yml
sudo -u git gitlab-shell/bin/install
```
> 如果redis使用TCP服务，则最后需要注释掉config.yml中redis项目中的socket行
>
```yaml
redis:
	bin: /usr/bin/redis-cli
	host: 127.0.0.1
	port: 6379
	# pass: redispass # Allows you to specify the password for Redis
	database: 0
	#socket: /var/run/redis/redis.sock # Comment out this line if you want to use TCP
	namespace: resque:gitlab
```

#### 4. 安装配置相关的服务

> Gitlab需要MySQL、Redis、Nginx的支持才能够正常工作，以此还需要对这些服务进行安装和配置
>
```bash
pacman -S redis nginx mysql
cp lib/support/nginx/gitlab /etc/nginx/
```
> 并在nginx.conf的http段中增加
>
```nginx
include gitlab;
```
> 同时在gitlab文件的server段结尾增加，如果使用了域名还需要修改一下server\_name
>
```nginx
error_page 404 = @gitlab;
```
#### 5. 配置及初始化Gitlab

> 首先初始化一些配置文件
>
```bash
cp config/gitlab.yml.example config/gitlab.yml
cp config/database.yml.mysql config/database.ym
cp config/unicorn.rb.example config/unicorn.rb
```
> 然后开始运行初始化，运行前需要启动mysql，并创建gitlab的数据库，同时还需要安装一个js引擎，这里使用nodejs
>
```bash
systemctl start mysqld
mysql -u root -e "CREATE DATABASE IF NOT EXISTS gitlabhq_production DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_unicode_ci'"
pacman -S nodejs
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
```

#### 6. 启动gitlab

> 启动gitlab前需要先启动mysql、redis、nginx，启动脚本在lib/support/init.d/gitlab，复制一份并增加运行权限
>
```bash
systemctl start mysqld
systemctl start redis
systemctl start nginx
cp lib/support/init.d/gitlab gitlab
chmod +x gitlab
#注意这里必须是绝对路径
/home/git/gitlab/gitlab start
```
> 第一次运行时会比较慢一些需要耐心等待一下，默认的用户名和密码为admin@example.com/password
