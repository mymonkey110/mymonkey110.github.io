---
title: 让百度索引你的github的博客
date: 2016-06-23 16:53:05
tags: life blog
category: 生活
description: Github屏蔽了百度的爬虫，导致众多托管在github上的博客无法进入百度的索引，损失了很多阅读量。本文分享一下解决这个问题的个人心得。
---

不知不觉中，写博客是一件很潮的事情，尤其是程序员。自然，我也是其中的一员。博客无非两种类型，一种是动态类型的，以Wordpress为代表；另外一种则是存静态的，以Hexo, Jekyll为代表。现在，程序员都喜欢把博客托管在github上。一来省去了买虚拟主机的费用，二来可以通过git工具来管理博客，用起来十分方便。我的博客就是用hexo搭建的。

首先，github是不支持动态博客的，它只能托管存静态的网页，也就是说你只能放置一些html,js,css,jpg,png…之类的静态文件。其次，github屏蔽了百度的爬虫，也就是说百度不能索引你的博客内容。虽说程序员基本上都用google，但是你肯定还是想能被百度搜索到的。想知道自己的博客是否被索引可以这样查询，在搜索引擎中输入：site: 你的博客域名。

解决github屏蔽百度爬虫的思路就是“迁出”我们的博客，让百度爬虫不直接访问github就行了。

## 方案一：利用CDN制作镜像网站

我们知道cdn能缓存静态资源，如果我们利用cdn制作我们的镜像网站，再将百度索引的解析cdn上，那么爬虫就不会访问github服务器了，而是访问cdn缓存服务器。国内最火的cdn服务商就是七牛和又拍云了。我发现七牛不支持自动回源功能，而又拍云在这方面做得比较好，我们可以使用又拍云来做为我们博客的镜像网站。

我以本站为例，讲一下配置的流程：

#### 1. 创建服务
   
   ![创建服务](/images/baidu_index/3.pic_hd.jpg)

#### 2. 配置回源
   
   ![配置回源](/images/baidu_index/5.pic_hd.jpg)

#### 3. 绑定域名
完成上面的操作后，又拍云会自动分配一个域名给我。此时，我们就需要绑定自己的域名。添加需要绑定的域名：

   ![绑定域名](/images/baidu_index/5.pic.jpg)

如果你希望博客能以www的方式来访问，那你还需要添加www的二级域名

#### 4. 配置解析
	
添加完域名绑定后，此时我们就只需要配置dns解析到又拍云了。我使用的是阿里云的域名系统，下图就是我设置的域名解析配置。

   ![配置解析](/images/baidu_index/6.pic_hd.jpg)

因为github在国外访问速度还是很快的，所以对于海外的用户直接访问github就可以了，不用再访问又拍云了。
添加解析后一般需要几分钟才生效，看自己添加的域名dns解析生效了没有可以使用nslookup命令：

```
    ~/blog  ᐅ nslookup michael-j.net
    Server:		192.168.199.2
    Address:	192.168.199.2#53
    
    Non-authoritative answer:
    michael-j.net	canonical name = mj-blog.b0.aicdn.com.
    mj-blog.b0.aicdn.com	canonical name = ctn.b9.aicdn.com.
    Name:	ctn.b9.aicdn.com
    Address: 183.134.101.194
```

  此时，我发现michael-j.net的域名已经成功解析到了又拍云。

5. 完成

完成以上步骤后，你会收到又拍云发给你关于域名绑定通过的邮件。此时你就可以在浏览器中访问你的博客啦！

最关键的问题是，我们要验证百度是否能正常的抓取我们的博客？ 我们使用百度站长的测试工具来测试一下：

  ![抓取诊断](/images/baidu_index/7.pic_hd.jpg)

哈哈，现在百度终于可以正常爬去我们的网站啦，接下来就是耐心的等待了，一般最多七天百度就会收录了。

## 方案二：自己托管博客

与利用cdn来制作镜像网站的思路一样，我们完全可以把博客托管在自己的服务器上，当然你得掏银子啦！💰 我个人觉得自己买一台属于自己的虚拟主机还是值得投入了，除了博客外你可以利用这台机器做很多事情，最低配的ecs也花不了多少钱，可以几个人合用一台。

Nginx是世界有名的反向代理服务器，同时它对静态文件的支持非常好，性能很高。我们完全可以利用Nginx来作我们博客的服务器。

#### 1. 安装Nginx

Ubuntu\Debian：`apt-get install nginx`

Centos\Redhat: `yum install nginx`

其他系统自行google

#### 2. 配置Nginx

在/etc/nginx/conf.d新作配置，一定要以`.conf`结尾。我新建名为`michael-j.net.conf`的配置文件：

```
server {
   	listen 80;
   	server_name michael-j.net;

   	location / {
   	  root /home/michael/mymonkey110.github.io;
   	  index index.html;
   	}

   	access_log /var/log/nginx/michael-j.access.log;
}
```

注意root是我们博客的目录，后面会提到。

#### 3. 重启nginx

执行`nginx -s reload`生效

#### 4. 自动下载博客内容

我希望每次博客仓库有更新的时候能自动重建本地仓库，为此我专门写了一个工具git-watcher: <https://github.com/mymonkey110/git-watcher>。当有新的内容push到你的仓库是，它会自动拉去并重建本地仓库。基本原理就是利用github的webhook功能，当有新的push事件发生时，github会发布相应的事件到指定的接口。git-watcher监听push事件，当接受到push事件去pull仓库。如果你觉得这个工具有点儿意思，Please star it.

##### 4.1 安装git-watcher & git

`pip install git-watcher`

`apt-get install git`

##### 4.2 启动git-watcher

`git-watcher -u https://github.com/mymonkey110/mymonkey110.github.io.git -s 654321`

`-u`参数配置我们的博客仓库地址

`-s`参数是我们webhook的secret key

git-watcher还支持其他一些参数配置，-h见说明

##### 4.3 配置dns解析

将默认的dns解析到我们自己的主机上

![配置解析](/images/baidu_index/10.pic_hd.jpg)

##### 4.4 配置webhook

进入仓库的`settings` －> `Webhooks & services`

设置：`Payload URL`，这里输入我们主机的地址，这里只能用ip地址。同时，还要设置Secret，这个是用来签名body内容用的，一定要与git-watcher中配置一致

![配置Webhook](/images/baidu_index/13.pic.jpg)

注意，我们只选择发送`push`事件就可以了。

##### 4.5 测试

我们进行一些修改，然后push到博客的仓库，检测一下网站内容是否更新。如果正常更新，那说明已经大功告成了。这是可以再用百度的抓取工具进行诊断。

## 总结

解决百度抓取github内容的问题基本思路都是让百度不直接访问Github，而是通过一个中间服务器来缓存内容。两种方式都需要付费，相对来说使用又拍云搭建镜像服务器在流量较小的情况下比较有优势，速度快，费用少；而自己租用主机在博客流量较大的时候比较经济，你可以选择按带宽计费的方式，同时你还获得了一台完全由你控制的主机，何乐而不为呢？！

	
