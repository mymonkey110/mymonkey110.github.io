title: Spring Boot反序列对象失败
date: 2016-12-21 10:21:55
tags: 
- debug 
- springboot 
- deserialized 
- redis
categories: debug
description: 本文记录了一次我遇到的诡异的反序列对象失败的debug过程，希望对遇到该问题的读者有帮助。
---

现在Spring Boot这个项目很火，尤其是微服务的流行，Spring Boot作为Java语言最热门的微服务框架之一，它极大地简化了Spring的配置过程。只需要一个注解就可以把整个工程拉起来，大大地降低了Spring的学习成本。我记得Spring Boot的某个开发人员说过，Spring Boot最令开发者激动的功能是可以自定义banner，哈哈，我也非常喜欢这个功能。

言归正传，开始介绍今天我遇到的一个诡异的问题。我使用Redis来缓存一些数据，但是这些数据在反序列的时候报错了。由于原工程涉及一些敏感信息，我新建了一个demo工程来说明这个问题。报错信息如下：
> java.lang.ClassCastException: com.netease.boot.dal.Product cannot be cast to com.netease.boot.dal.Product

看到这个报错我就懵逼了，以致于我对了好几遍来确认眼睛没有看花。经过若干次重试，还是一样的错误。有人可能会对`Product`的实现产生怀疑，是不是没有加`serialVersionUID`，作为一个专业老司机，这点错误我还是不会犯得。我贴一下相关的代码：

Product类如下：

```
public class Product implements Serializable {
    private static final long serialVersionUID = -5837342740172526607L;

    @Size(min = 1, max = 32)
    private String code;
    @Size(min = 1, max = 16)
    private String name;
    @Size(max = 255)
    private String description;
    @NotNull
    private EMailAddress principalEmail;

    public Product(String code, String name, String description, EMailAddress principalEmail) {
        this.code = code;
        this.name = name;
        this.description = description;
        this.principalEmail = principalEmail;
    }

    public void changeName(String newName) {
        this.name = newName;
    }

    public void changeDescription(String newDescription) {
        this.description = newDescription;
    }

    public void changePrincipalEMail(EMailAddress newPrincipalEMail) {
        this.principalEmail = newPrincipalEMail;
    }

    public String getCode() {
        return code;
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    public EMailAddress getPrincipalEmail() {
        return principalEmail;
    }

    @Override
    public String toString() {
        return "Product{" +
                "bizCode='" + code + '\'' +
                ", name='" + name + '\'' +
                ", description='" + description + '\'' +
                ", principalEmail=" + principalEmail +
                '}';
    }
}
```

redis service相关的代码如下：

```
　　@Override
    public void put(String key, Serializable content) throws RedisException {
        Jedis jedis = null;
        try {
            jedis = redisPoolConfig.getJedis();
            byte[] contentBytes = SerializationUtils.serialize(content);
            jedis.set(key.getBytes(ENCODING), contentBytes);
        } catch (Exception e) {
            LOG.error("Put error:{}.", e.getMessage(), e);
            throw new RedisException(e);
        } finally {
            if (jedis != null) {
                redisPoolConfig.releaseJedis(jedis);
            }
        }
    }
    
　　@Override
    public <T> T get(String key) throws RedisException {
        Jedis jedis = null;
        try {
            jedis = redisPoolConfig.getJedis();
            byte[] valueBytes = jedis.get(key.getBytes(ENCODING));
            if (valueBytes == null || valueBytes.length == 0) {
                return null;
            }
            return SerializationUtils.deserialize(valueBytes);
        } catch (Exception e) {
            LOG.error("Get error:{}.", e.getMessage(), e);
            throw new RedisException(e);
        } finally {
            if (jedis != null) {
                redisPoolConfig.releaseJedis(jedis);
            }
        }
    }
```

实在没办法，这尼玛是什么问题。因为我以前这么使用过，而且工作的非常好，为毛这次就不行了。没办法了，加debug代码，我让get方法返回Object，再外面强转，（冥冥中有一种感觉，像是泛型的问题）。修改后的代码如下：

```
@Override
    public Object get(String key) throws RedisException {
        Jedis jedis = null;
        try {
            jedis = redisPoolConfig.getJedis();
            byte[] valueBytes = jedis.get(key.getBytes(ENCODING));
            if (valueBytes == null || valueBytes.length == 0) {
                return null;
            }
            Object o = SerializationUtils.deserialize(valueBytes);
            return o;
        } catch (Exception e) {
            LOG.error("Get error:{}.", e.getMessage(), e);
            throw new RedisException(e);
        } finally {
            if (jedis != null) {
                redisPoolConfig.releaseJedis(jedis);
            }
        }
    }
```

在反序列化之后加断电debug，观察变量o，得到如下所示的图：
![redis-get](http://7xnwpq.com1.z0.glb.clouddn.com/redis-deser.png)

WTF! IDE都识别出来了变量o是Product类型，但是后续的强转还是失败。经过我的测试发现所有的通过redis反序列化出来的类都有这个问题。万般无奈之下，我陷入了深深地沉思之中...之中...中...

我开始怀疑是序列化的姿势不对，但是为毛以前可以啊。不管了，先加一段测试代码：

```Java
Product product = new Product("comb","蜂巢","云计算基础设施产品",new EMailAddress("hzxx@corp.netease.com"));
        /*FileOutputStream fileOutputStream = new FileOutputStream("/home/mj/work/product.data");
        fileOutputStream.write(SerializationUtils.serialize(policyContext));
        fileOutputStream.flush();
        fileOutputStream.close();*/
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("/home/mj/work/product.data"));
        oos.writeObject(product);
        oos.flush();
        oos.close();

        /*FileInputStream fileInputStream=new FileInputStream("/home/mj/work/product.data");
        byte[] rawPolicyContext=new byte[fileInputStream.available()];
        fileInputStream.read(rawPolicyContext);
        PolicyContext pc = SerializationUtils.deserialize(rawPolicyContext);
        System.out.println(pc);*/
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("/home/mj/work/product.data"));
        Product pc = (Product) ois.readObject();
        System.out.println(pc);
```

在倒数第二行打点，截图如下：

![diy-deser](http://7xnwpq.com1.z0.glb.clouddn.com/diy-deser.png)

没截图没毛病啊，很正常啊。我还专门测试了`SerializationUtils`版的序列化方式(把上面的注释去掉)，发现结果也很正常，这尼玛到底是怎么回事。实际上，`SerializationUtils`也就是jdk自带的`ObjectOutputStream`和`ObjectInputStream`的简单封装。

在我走投无路之际，正准备研究`instanceof`的工作原理的时候，脑中闪过一道灵感——难道是classloader的问题？说干就干，debug得到如下情况：

![classloader-1](http://7xnwpq.com1.z0.glb.clouddn.com/classloader-1.png)
![classloader-2](http://7xnwpq.com1.z0.glb.clouddn.com/classloader-2.png)

终于发现问题所在了，原来两个classloader不一样，而`instanceof`是对同一个classloader而言的。再确定原因后，借助强大的google发现了这是Spring Boot DevTools的一个限制，相关的文档链接: <http://docs.spring.io/spring-boot/docs/1.4.2.RELEASE/reference/htmlsingle/#using-boot-devtools-known-restart-limitations>

原话是这样的：
> Restart functionality does not work well with objects that are deserialized using a standard ObjectInputStream. If you need to deserialize data, you may need to use Spring’s ConfigurableObjectInputStream in combination with Thread.currentThread().getContextClassLoader().
Unfortunately, several third-party libraries deserialize without considering the context classloader. If you find such a problem, you will need to request a fix with the original authors.

DevTools是Spring Boot中一个很有用的工具，可以自动帮你重启应用，而不用你每次重启应用来debug，提高了生产效率。具体的用法可以参考相关的文档。这里的限制条件说的很清楚了，重启功能不能和使用标准的`ObjectInputStream`来反序列对象一起使用，如果你非要使用，那么请从线程的上下文中来获取classloader。

看到这里我瞬间明白了。因为devtools使用两个classloader，你工程中使用的第三方jar包被一个叫"base"的classloader所加载，而你正在开发的代码被一个叫"restart"的classloader所加载。如果检测到你的classpath路径下文件有变化，restart就会重新加载你工程的类。这样做以后能提高你的类加载速度，这在开发阶段是很有用的一个功能。

既然知道了原因，就很好解决了。因为我目前的工程比较小，而且只是一个restful后端应用，所有devtools对我的应用帮组不大。注释掉devtools依赖后就解决了上面的问题。如果你想使用这个工具，同时又有反序列化的需求，有两种方式解决：

1. 自定义一个`ObjectInputStream`，重写`resolveClass`方法，也可以使用Spring提供的`ConfigurableObjectInputStream`类。然后从`Thread.currentThread().getContextClassLoader()`获取classloader就可以解决该问题。
2. 配置`spring-devtools.properties`文件，把你使用的第三方序列化工具也加入`restart　classloader`的控制范围内就行了。

这两种方法均可以在Spring Boot的官方文档中有详细描述：<http://docs.spring.io/spring-boot/docs/1.4.2.RELEASE/reference/htmlsingle/#using-boot-devtools>。

总结，从发现问题到定位原因耗时两个多小时，还是要加强对基础概念的深入理解才能快速定位原因啊！

文本的示例demo我已上传到github，有兴趣的同学可以下载自己debug一下：<https://github.com/mymonkey110/boot-demo.git>

参考资料:

[Spring Boot官方手册](http://docs.spring.io/spring-boot/docs/1.4.2.RELEASE/reference/htmlsingle/#using-boot-devtools)
[spring-boot issue](https://github.com/spring-projects/spring-boot/issues/3805)
[redis serialization](http://stackoverflow.com/questions/30795262/redis-serialization-and-deserialization)
[classcastexception](http://stackoverflow.com/questions/37977166/java-lang-classcastexception-dtoobject-cannot-be-cast-to-dtoobject)
