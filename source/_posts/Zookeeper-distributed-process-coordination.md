---
title: Zookeeper:distributed process coordination中文版
date: 2016-08-02 16:53:05
tags: zookeeper
category: 翻译
description: Zookeeper:distributed process coordination中文译本。
---

最近使用zookeeper比较多，但是国内关于zookeeper方面的数据太少。能介绍其使用同时也讲解原理的书太少了。Zookeeper:distributed process coordination是一本关于zookeeper不可多得的好书。读完以后我对zookeeper有能一个非常直观的了解。

现在分布式应用开发越来越常见，基本上大部分的分布式应用都需要与其它应用进行协同。Zookeeper非常擅长于处理分布式协同。所以我决定利用工作之余的时间翻译这本书籍，完全出于个人兴趣。

![zookeeper:distributed process coordination](/images/zookeeper/zookeeper.jpg)

[GitBook阅读地址](https://www.gitbook.com/book/mymonkey110/zookeeper-distributed-process-coordination)

[GitHub阅读地址](https://github.com/mymonkey110/zookeeper-book/blob/master/SUMMARY.md)

由于本人第一次翻译技术书籍，肯定会有很多翻译不当的地方，欢迎大家能及时指正。如果有对本书翻译有兴趣的小伙伴，可以通过以下方式参与贡献：

参与讨论：邮件列表：<zk_translator@groups.163.com>，申请加入地址：http://163.fm/UJNWGHS

部分贡献：通过issue进行讨论，如果通过，我会进行修改。这种方式我无法统计贡献者的名字，建议使用下面的方式参与翻译。

在 GitHub 上 fork 到自己的仓库，如 user/zookeeper-book，然后 clone 到本地，并设置用户信息。

```
$ git clone git@github.com:user/zookeeper-book.git
$ cd zookeeper-book
$ git config user.name "yourname"
$ git config user.email "your email"

```

修改代码后提交，并推送到自己的仓库。

```
$ #do some change on the content
$ git commit -am "Fix issue #1: change helo to hello"
$ git push
```

在 GitHub 网站上提交 pull request。
定期使用项目仓库内容更新自己仓库内容。

```
$ git remote add upstream https://github.com/mymonkey110/zookeeper-book.git
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
$ git push -f origin master
```

PS: 2016/8/15 Update:

很遗憾，因为授权的问题，不得不停止翻译的工作。本书已经有中文版的译本了，我后来才得知，所以我也不会取得中文版的翻译授权了。因为本人第一次翻译，事先没有搞清这些事情，才导致了现在的情况。不得不说，十分遗憾，感谢关注本书翻译的伙伴。我相信已有的中文译本应该还不错，如果需要的伙伴可以去购买。So, that's it, it's over, thanks for your attention.