title: 使用 Hexo 搭建博客
---


## 一、购买 VPS

我使用的是 [DigitalOcean][1]。购买合适的套餐后，你会得到一个预装了指定操作系统的虚拟服务器，以及一个虚拟服务器 IP（VPS-IP）。


## 二、安装软件

登录 VPS，安装一些依赖软件。

### Node.js

1. 安装 Node.js

    这里安装 Node.js v0.12，其中自带了 NPM。参考 [nodesource/distributions][2]。

    ```bash
    $ curl -sL https://deb.nodesource.com/setup_0.12 | sudo -E bash -
    $ sudo apt-get install -y nodejs
    ```

2. 配置 NPM

    设置 NPM 的全局安装目录为用户主目录。参考 [Install NPM into home directory with distribution nodejs package (Ubuntu)][3]。

    ```bash
    # Create .npmrc
    NPM_PACKAAGES="$HOME/.npm-packages"
    mkdir -p "$NPM_PACKAAGES"

    echo "prefix = $NPM_PACKAAGES" >> $HOME/.npmrc

    # Add configurations into .bashrc (or .zshrc)
    echo '
    # NPM packages in homedir
    NPM_PACKAGES="$HOME/.npm-packages"

    # Tell our environment about user-installed node tools
    PATH="$NPM_PACKAGES/bin:$PATH"
    # Unset manpath so we can inherit from /etc/manpath via the `manpath` command
    unset MANPATH  # delete if you already modified MANPATH elsewhere in your configuration
    MANPATH="$NPM_PACKAGES/share/man:$(manpath)"

    # Tell Node about these packages
    NODE_PATH="$NPM_PACKAGES/lib/node_modules:$NODE_PATH"
    ' >> $HOME/.bashrc
    ```

### Git

``` bash
$ sudo apt-get update
$ sudo apt-get -y install git
```

### Supervisor

``` bash
$ sudo apt-get install -y supervisor
```

### Nginx

``` bash
$ sudo apt-get -y install nginx
```


## 三、使用 Hexo

### 安装 Hexo

```bash
$ npm install -g hexo
```

### 初始化 Hexo

```bash
$ mkdir hexo && cd hexo
$ hexo init blog
$ cd blog
$ npm install
```

### 配置 Hexo

Hexo 有两份主要的配置文件（_config.yml），一份位于站点根目录下，另一份位于主题目录下。为了描述方便，在以下说明中，将前者称为 `站点配置文件`，后者称为 `主题配置文件`。

#### 站点信息

编辑 `站点配置文件`：

```
title: RussellLuo
subtitle: 让思想在文字间徜徉
description:
author: RussellLuo
language: zh-Hans
timezone: Asia/Chongqing
```

#### 绑定博客仓库

考虑到独立性和可维护性，我的博客文章都是放在 GitHub 的 [blog 仓库][4] 里的。

为了保证所有文章能被 Hexo 正确地识别和处理，blog 仓库会被克隆并绑定到 Hexo 中 `post layout` 所在的默认路径 `source/_posts`：

1. 克隆 blog 仓库

    先从 GitHub 克隆 blog 仓库：

    ```bash
    $ mkdir github && cd github
    $ git clone https://github.com/RussellLuo/blog.git
    ```

2. 绑定前的目录结构

    ```
    /home/user/
        hexo/        # Hexo 站点
            blog/
                source/
                    _posts/
        github/      # GitHub 仓库
            blog/
                blog/
                    2015/
                        2015-08-30-hello.md
                        ...
                    ...
    ```

3. 绑定 blog 仓库

    这里采用软链接的方式实现绑定，简单直接。

    ```bash
    $ cd hexo/blog/source/_posts
    $ ln -s ~/github/blog/blog blog
    ```

4. 绑定后的目录结构

    ```
    /home/user/
        hexo/        # Hexo 站点
            blog/
                source/
                    _posts/
                        blog/
                            2015/
                                2015-08-30-hello.md
                                ...
                            ...
        github/      # GitHub 仓库
            blog/
                blog/
                    2015/
                        2015-08-30-hello.md
                        ...
                    ...
    ```

#### 文章链接

基于上述绑定后的文章目录结构，我希望文章的链接格式形如 `/yyyy/mm/title.html`。以文章 `blog/2015/2015-08-30-hello.md` 为例，我希望它对应的链接是 `/2015/08/hello.html`。

为了到达上述效果，需要编辑 `站点配置文件` 如下：

```
root: /
permalink: :year/:month/:title.html
new_post_name: blog/:year/:year-:month-:day-:title.md
```


### 配置主题

我个人喜欢简洁优雅的风格，因此选择了 [NexT][6] 主题。

#### 1. 下载 NexT 主题

```bash
$ cd blog
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```

#### 2. 启用 NexT 主题

编辑 `站点配置文件`：

```
theme: next
```

#### 3. 设置主题参数

启用 NexT 主题中的 Mist 主题：编辑 `主题配置文件`，将 `#scheme: Mist` 前面的 `#` 注释去掉。

#### 4. 侧边栏

1. 头像

    编辑 `站点配置文件`，新增字段 `avatar`，将值设置成头像的链接地址。

2. 社交链接

    编辑 `站点配置文件`，新增字段 `social`，然后添加社交站点名称与地址即可。例如：

    ```
    social:
      GitHub: https://github.com/RussellLuo
      豆瓣: http://douban.com/people/RussellLuo
    ```

#### 5. 评论系统

虽然 `Disqus` 是国外最流行的第三方评论系统，但是在国内感觉使用 `多说` 更接地气。

`NexT` 主题内置支持 `多说` 评论系统，因此配置很简单：

1. 创建站点

    登录 [多说][5] 后，在首页点击“我要安装”，创建站点，填写必要信息。

    其中，`多说域名` 一栏就是你的 `duoshuo_shortname`。

2. 配置 duoshuo_shortname

    编辑 `站点配置文件`，新增 `duoshuo_shortname`：

    ```
    duoshuo_shortname: russellluo
    ```

> **更新于 2017 年 5 月 18 日**
> 多说在 2017 年 3 月 21 日宣布：[将于 2017 年 6 月 1 日正式关停服务][7]。Disqus 会被墙，畅言要备案，最后发现了 [网易云跟帖][8]。
>
> 切换到网易云跟帖的步骤：
> 1. 注册网易云跟帖的账号
> 2. 按提示填写相关信息（特别注意 “站点网址” 需要跟博客网址相同，比如 `http://russellluo.com`）
> 3. 升级到最新版本的 NexT
> 4. 编辑 `站点配置文件`，注释掉 `duoshuo_shortname`
> 5. 编辑 `主题配置文件`，[设置 gentie_productKey][9]
>
> 更多详情，可以参考 [这篇博客][10]。


> **更新于 2017 年 9 月 26 日**
> 继多说关闭之后，网易云跟帖在 2017 年 8 月 1 日也停止了服务:( 不打算折腾了，直接切换到 Disqus（墙内用户需要自备梯子）。
>
> 切换到 Disqus 的步骤：
> 1. 注册 [Disqus](https://disqus.com/) 的账号并登录
> 2. 依次点击 “GET STARTED”、“I want to install Disqus on my site”
> 3. 按提示填写站点信息（注意 “Website Name” 的内容就是 shortname，稍后会用到，比如 `russellluo`）
> 4. 按提示 “1. Select Plan”（没钱就选择 Basic）、“2. Install Disqus”（NexT 主题的可以忽略）、“3. Configure Disqus”（填写 “Website URL” 后，点击 “Complete Setup” 即可）
> 5. 编辑 `主题配置文件`，启用 Disqus：
>
>     ```
      disqus:
        enable: true
        shortname: russellluo
        count: true
      ```
> 更多详情，可以参考 [这篇博客](http://www.cylong.com/blog/2017/03/26/hexo-next-disqus/)。


## 四、正式部署

### 1. 使用 Supervisor 管理 Hexo 服务

使用 `hexo server` 启动的 Hexo 服务是非 Daemon 模式的。为了便于管理，这里使用 [Supervisor][7]。

创建 Supervisor 配置文件：

```bash
$ vi /etc/supervisor/conf.d/blog.conf

[program:blog]
command=/home/user/.npm-packages/bin/hexo server
directory=/home/user/hexo/blog
autostart=true
autorestart=true
startsecs=5
stopsignal=HUP
stopasgroup=true
stopwaitsecs=5
stdout_logfile_maxbytes=20MB
stdout_logfile=/var/log/supervisor/%(program_name)s.log
stderr_logfile_maxbytes=20MB
stderr_logfile=/var/log/supervisor/%(program_name)s.log
```

启动 Supervisor 守护进程：

```bash
$ supervisord
```

查看 blog 程序（即 Hexo 服务）的状态：

```bash
$ supervisorctl status
blog                             RUNNING    pid 28974, uptime 0:00:32
```

可以看出，blog 程序已经处于运行状态，监听端口为 `hexo server` 命令的默认端口 `4000`。在浏览器中访问 `http://<VPS-IP>:4000` 可以看到博客的运行效果。

### 2. 配置 Nginx 代理

作为一个对外公开的网站，使用 4000 端口显然是不合适的。可以直接改成 80 端口，但是这样直接把 Hexo 服务暴露给用户，并不恰当。更好的办法是使用 Nginx 做代理。

创建 Nginx 配置文件：

```bash
$ vi /etc/nginx/conf.d/blog.conf

server {
    listen 80;
    server_name <VPS-IP>;

    location / {
        proxy_pass http://localhost:4000;
    }

    access_log  /var/log/nginx/blog.access.log;
    error_log /var/log/nginx/blog.error.log;
}
```

重启 Nginx：

```bash
$ nginx -t
$ nginx -s reload
```

此时，在浏览器中访问 `http://<VPS-IP>`，就可以体验到高效、稳定的博客网站。


## 五、购买域名

让用户通过 `http://<VPS-IP>` 访问博客，显然是反 Web 的行为。作为一名专业的博主，我在 [GoDaddy][8] 购买了自己博客的域名。

启用了高大上的域名后，需要修改上述的 Nginx 配置，将 server_name 从 IP 改成域名：

```bash
$ vi /etc/nginx/conf.d/blog.conf

server {
    ...
    server_name russellluo.com;
    ...
}
```


## 六、设置 DNS

对外开放的最后一步，是设置 DNS，让域名 russellluo.com 真正解析到我的虚拟服务器 IP。我用 [DNSPod][9]，快速、免费、稳定！

That's all! 如果你在我的博客上看到了这篇文章，说明我已经成功了。


[1]: https://www.digitalocean.com
[2]: https://github.com/nodesource/distributions#installation-instructions
[3]: http://stackoverflow.com/questions/10081293/install-npm-into-home-directory-with-distribution-nodejs-package-ubuntu
[4]: https://github.com/RussellLuo/blog
[5]: http://duoshuo.com
[6]: https://github.com/iissnan/hexo-theme-next
[7]: http://dev.duoshuo.com/threads/58d1169ae293b89a20c57241
[8]: https://gentie.163.com
[9]: http://theme-next.iissnan.com/third-party-services.html#yungentie
[10]: http://www.jianshu.com/p/3d0cb3becc6d
[11]: http://supervisord.readthedocs.org
[12]: https://www.godaddy.com
[13]: https://www.dnspod.cn
