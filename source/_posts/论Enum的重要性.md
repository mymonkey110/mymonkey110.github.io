---
title: 论Enum的重要性
date: 2016-09-20 22:18:11
tags: 
- enum 
- OO
- 代码技巧
category: Better code
description: Enum是Java中非常重要的一种数据类型，用好Enum是写好代码必须掌握的技术。
---

我们都知道Java中有一种数据类型是枚举类，实际上很多强类型的语言都有枚举。但是很多人对枚举类不那么重视，或者不能正确地应用枚举，也就不能发挥其威力了。这里分享一下我对枚举的理解，及其常见的用法。

既然Java专门为枚举建立了类型，那么我们应该在什么时候去使用enum呢，我认为在以下两个场景中可以尝试使用。

#### 1. 封装有限的的变化

相信很多人都遇到这样一个场景，我们有一个父类，父类下面有几个子类，而这几个子类是可以确定的。我们并不想父类被不相干的类所继承，那么我们可以通过enum来限制子类。实际上你想把代码控制在预期的范围之类时，都可以通过enum来达到效果。

#### 2. 状态代码

我们经常会遇到使用状态码的情况，例如在任务处理过程中。我发现很多人喜欢使用int或者long来表示状态码，然后通过定义对应的变量来表示其意义。不是说这种方式不好，但我从中嗅出了一丝坏味道。如果通过int或者long来表示状态码，如果出现了不在业务范围内的值该怎么办？为什么状态码不能直接表示其意义，还需要通过文档来说明呢？我一直比较推崇Self-Explained的编程习惯，代码和文档合二为一。

那么使用Enum有什么好处了，我们为什么要用Enum呢？相比于int或者string，enum最大的优势就是有它是类型安全的。如何理解类型安全呢，我举一个例子：很多APP都有第三方登陆的功能，服务器要根据客户端传过来的登陆类型(type)来调用对应平台的接口来获取用户信息。我的代码是这样写的：

```
@Component
public class TPAccountRouterImpl implements TPAccountRouter {
    @Resource
    @Qualifier("wbAccountResolver")
    private TPAccountResolver wbTPAccountResolver;

    @Resource
    @Qualifier("wxAccountResolver")
    private TPAccountResolver wxTPAccountResolver;

    @Resource
    @Qualifier("qqAccountResolver")
    private TPAccountResolver qqTPAccountResolver;

    @Override
    public TPAccount getAccountInfo(final String tuid, String accessToken, AccountType accountType) throws TPException {
        TPAccountResolver tpAccountResolver;

        switch (accountType) {
            case WB:
                tpAccountResolver = wbTPAccountResolver;
                break;
            case WX:
                tpAccountResolver = wxTPAccountResolver;
                break;
            case QQ:
                tpAccountResolver = qqTPAccountResolver;
                break;
            default:
                throw new TPException("unknown account type");
        }

        return tpAccountResolver.getAccountInfo(tuid, accessToken);
    }
}
```

`TPAccountRouter`是一个账号解析的路由器，根据`AccountType`来调用对应平台的解析器来解析。配合switch-case语法，利用策略模式我们就可以写出一个还算优美的代码。如果把`accountType`换成`int`会怎样？那么我们不得不加上一句及其烦人的
```
if(accountType<0 || accountType>3) {
  throw new IllegalArgumentException("type illegal");
}
```
保护性代码，同时将case子句换成一个一个静态常量，最后还在API文档上配上说明，1,2,3各代表什么意义。我相信大家一定能感受出来两种代码写法带来的区别。

另外一个有点，我认为就是enum的self-explain特性，上面的例子中也直观的反应了这一点。Enum结合了int和String的优点，并将其发扬光大。

关于Enum怎么用，网上有很多的介绍，可以参考这篇文章：<http://www.cnblogs.com/happyPawpaw/archive/2013/04/09/3009553.html>，还是比较全面的。最常用的就是直接申明各个枚举值，基本上能满足大部分业务场景了。也有很多场景下，我们会在enum中加入成员变量，这是因为业务中存在和Enum相对应的文档和动作。再举一个我写过的代码例子：
```
public abstract class AbstractCheckedException extends Exception {

    private static final long serialVersionUID = -3143228702981231790L;

    public AbstractCheckedException() {
    }

    protected abstract ErrorCode errorCode();

    public int code() {
        return errorCode().code();
    }

    public String msg() {
        return errorCode().msg();
    }

    public static int successCode() {
        return ErrorCode.SUCCESS.code();
    }

    public enum ErrorCode {
        SUCCESS(1000, "success"),
        PARAM_ERROR(1001, "parameter error"),
        ILLEGAL_REQUEST(1002, "illegal request"),
        SYS_ERROR(1003, "system error"),
        NAMESPACE_NOT_FOUND(2001, "namespace not found"),
        NAMESPACE_ALREADY_EXIST(2002, "namespace already exist"),
        APP_NOT_FOUND(2003, "app not found"),
        APP_ALREADY_EXIST(2004, "app already exist"),
        TASK_NOT_FOUND(2005, "task not found"),
        TASK_ALREADY_EXIST(2006, "task already exist"),
        CRON_EXPRESSION_ERROR(2007, "cron expression error"),
        ZOOKEEPER_ERROR(3001, "zookeeper error"),
        NODE_NOT_EXIST(3002, "node not exist"),
        NODE_ALREADY_EXIST(3003, "node already exist"),
        UNKNOWN_ERROR(9999, "unknown error");

        private int code;
        private String msg;

        ErrorCode(int code, String msg) {
            this.code = code;
            this.msg = msg;
        }

        public int code() {
            return code;
        }

        public String msg() {
            return msg;
        }

        public static ErrorCode getErrorCode(int code) {
            for (ErrorCode it : ErrorCode.values()) {
                if (it.code() == code) {
                    return it;
                }
            }
            return UNKNOWN_ERROR;
        }
    }
}
```

我在[jscheduler](https://github.com/mymonkey110/jscheduler)中封装了高层了受检异常，这点收到了Zookeeper中`KeeperException`的启发。我在ErrorCode中加入了code和message，因为code和message是和这个枚举绑定的，放到枚举中再合适不过呢，我将之称为文档的绑定。还有情况是因为业务动作和枚举相关，比如第三方登陆的例子，我们完全可以第三方登陆接口的URL放到`AccountType`中，然后后续的解析方法直接从中取的URL进行调用就行，因为这个解析方法是和Enum一一对应的。这样的例子实在太多了，不胜枚举。

总之，如果你有一类相识的业务场景，并且这些业务场景只有有限的变化，是可以预期的，那么建议你考虑一下使用Enum。相信我，它值得尝试！




