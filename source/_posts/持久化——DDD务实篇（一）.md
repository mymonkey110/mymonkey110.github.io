---
layout: post
title: 持久化——DDD务实篇（一）
date: 2020-05-15 10:47:42
tags:
 - DDD
 - 持久化
categories: DDD
description: DDD实质上还是关于代码架构的一种方法论，那么如何做好持久化是一个至关重要的问题，本篇我将分享一下自己在实践过程中的一些思考与技巧。
---

持久化就是Java对象存储到数据库以及从数据库中构建出Java对象的技术。DDD中的持久化无非是对实体、值对象和聚合的持久化，这与传统的针对POJO的持久化还是存在一些差异的。我将从模块依赖、ORM框架和具体对象的持久化来说明DDD中持久化的技术与思考。

## 模块依赖

DDD的模块划分一般如下：

{% asset_img DDD-Module.png DDD-Module %}

其中领域层(Domain)是依赖基础设施层（Infrastructure）的。我个人认为实际上这个关系应该反过来，让基础设施层依赖领域层，而领域层不依赖任何其他的Module。

这样做的好处在于：

1. 领域层更加纯粹。采用DDD项目的核心就在于领域层，这也是整个项目最有价值的地方。领域层的建设应该尽量排除基础设施的干扰，专注于业务逻辑的实现。
2. 让领域层具有可移植性。虽然更换基础设施的概率不大，但还是需要考虑这样的问题。随着项目的不断发展，传统的RDBMS很可能适应不了持久化方面的要求，私有化部署可能要求依赖的开源软件固定在几个少数的数据库中，公司技术层面战略变化导致底层迁移等等这些可能的因素要求我们具有一定的前瞻性。

既然决定让Domain不依赖Infrastructure，那么Reposistory应该放到哪里？

我推荐的做法是将接口放到Domain中，而Reposistory的实现放到Infrastructure里面。直接将Reposistory的Interface和业务模型放到同一个package下面就可以了，让业务模型和负责业务模型的存储接口放到一个目录下。但这里感觉有点“坏味道”的产生，好像存储层的内容“侵入”到了领域层。我举得这是可以接受的，因为在领域层的模型是以业务维度划分的，实体与存储库是可以共存。反而，如果让领域层依赖基础设施，那么这种侵入性将更加难以控制，很容易造成深度耦合。

## 持久化的时机

领域层的实体和聚合说白了就是业务属性和业务行为的集合，业务属性对应表中的字段，而行为则反映了模型本身的能力。业务逻辑的实现从内存层面去看就是通过一系列计算来改变属性的值，但这个改变后的实体对象对应着本机内存的一个普通Java对象。那么在哪里对这个Java对象进行持久化呢？

大部分情况下，一般在Application层调用Reposistory进行持久化。大部分项目主要是针对实体的CRUD操作，实体的行为基本局限在实体本身，通过调用一个或者少数几个实体方法就可得到满足要求的实体对象。这种情况下，在Application进行持久化是比较简单容易的。

Application的Service进行持久化的一般步骤如下：

1. 检索出对应的实体
2. 调用对应的实体方法
3. 最后调用实体对应的Reposistory进行持久化

一个典型的Service方法如下所示：

```java
@Override
@Transactional
public void changeRoleRemark(long primaryAccountId, String roleName, Stringremark) throws RoleNotFoundException {
    Optional<Role> result = getUserRoleByName(primaryAccountId, roleName);
    Role role = result.orElseThrow(RoleNotFoundException::new);
    role.changeRemark(remark);
    roleRepository.update(role);
}
```

我们要修改`Role`（Role对应一个实体）的备注，上面是一个典型的Service实现。再调用完Role的changeRemark方法后，直接调用`roleRepository`的update方法将更新后的Role持久化到数据库中。

关系表的持久化也是一样的处理方式，如下所示。

```java
@Override
@Transactional
public void deleteUserRole(long primaryAccountId, String roleName)
        throws RoleNotFoundException {
    Optional<Role> result = getUserRoleByName(primaryAccountId, roleName);
    Role role = result.orElseThrow(RoleNotFoundException::new);
    RoleRelationParam roleRelationParam = new RoleRelationParam();
    roleRelationParam.setPrimaryAccountId(primaryAccountId);
    roleRelationParam.setRoleId(role.getId());
    roleRelationRepository.revokeRolePolicy(roleRelationParam);
    roleRepository.delete(primaryAccountId, roleName);
}
```

上诉的持久化方式可以满足大部分项目的需求，但是总有特殊情况。我在一些项目中发现，针对具有长流程的实体行为，持久化行为下沉到实体方法里面会更为合适。

举个例子，云服务中的IaaS和iPaaS服务的管控端一般都有着很多长流程的业务逻辑，如"创建一个云主机“，”对一个云主机实例升配“，”创建一个高可用版的RDS实例“等等。这些长流程的业务逻辑一般有一个个原子的步骤操作构造，如"共享镜像给租户"、"指定物理机创建VM"、"挂在硬盘"、"启动指定脚本"等等。做的比较好的管控端会将这些原子步骤抽离出来，形成预编码的Step；同时长流程过程做成一个抽象后的实体Process。一个Process由多个Step组成，Process控制着Step的运行过程。

这个例子中，Process具有一个Start或者Execute的行为方法，代表整个流程的启动和执行。在执行过程中，Process需要保存每个Step的执行上下文，为了执行过程异常中断后可继续整个执行流程。那么一个合理的持久化方式如下：

```java
@Override
public void execute() {
    StepRepository stepRepository = BeanUtil.getBean(StepRepository.class);
    for(Step step: steps) {
        step.execute();
        stepRepository.save(step);
    }
}
```

当然，我们也可以将Step的保存和Step的execut放到一起，这样Process就不需要关心如何对Step进行持久化了。

之所以长流程的持久化方式不同于短流程，其根本原因在于我们关心长流程运行过程中产生的中间变量。这些中间变量如果也是实体，那么我们就必须对齐进行持久化。对中间变量持久化已经成为了业务逻辑的一部分，所以我们需要将其放到领域层的模型中进行。

## ORM框架

如果可以，请在使用DDD的项目中使用更加自动化ORM框架，如Hibernate；MyBatis这种半自动化SQL映射工具对DDD项目来说不太友好。

由于我自己对Hibernate不熟悉，不适合做两者的对比。使用DDD架构模式的项目其核心在于领域模型的建模与演化，实质上是一种以Java对象为核心的开发方式。而三段式的开发方式中，持久化层是以表为核心的，Java对象（一般是POJO）只是表的容器而已。因为核心的关注点不同，导致在框架的设计理念是不一样的。Mybatis对DDD的领域模型侵入性比较强，而Hibernate对DDD中的实体和值对象都有着很好的支持。个人认为一款好的ORM框架是能最大程度地贴合你的领域模型，能灵活的支持模型对象的存取和模型间关系的映射。

目前我还无法细致的描绘ORM对DDD的影响，后期有机会在对其进行更加细致的分析。

## 实体持久化

实体的持久化注意以下几点：

### 善用Constructor

Construct在POJO中基本未被使用，而在DDD中它十分重要。因为我们的开发思维已经从面向表编程转变为面向对象编程，那么对象的每一个属性、构造器和方法我们都应该反复的思考与推敲。

Constructor代表了一个对象的开始，它是如何被构造出来的。在实体中，Constructor应该接受这个实体构造必不可少的参数，当然可以提供多个Constructor来接受更加丰富的参数。下面是一个`UserGroup`的实体对象。

```java
/**
 * 子账号用户组
 *
 * Created by Michael Jiang on 16-12-5.
 */
public class UserGroup extends Entity {
    private static final long serialVersionUID = -3431132267526140851L;
    private static final int MAX_GROUP_NUMBER = 50;

    @Min(1)
    private long primaryAccountId;

    @Pattern(regexp = "^[a-z]([a-z0-9-]{0,62}[a-z0-9]$|[a-z0-9]?)", message = "组名称只能是英文字母，数字和中划线")
    private String name;

    @Size(max = 255)
    private String remark;

    @Size(max = 50)
    private List<String> subAccountList;

    public UserGroup(Long primaryAccountId, String name) {
        this(primaryAccountId, name, "");
    }

    public UserGroup(Long primaryAccountId, String name, String remark) {
        this.primaryAccountId = primaryAccountId;
        this.name = name;
        this.remark = remark;
        this.subAccountList = Lists.newArrayList();
    }

    public long getPrimaryAccountId() {
        return primaryAccountId;
    }

    public String getName() {
        return name;
    }

    public String getRemark() {
        return remark;
    }

    public void changeName(String newGroupName) {
        this.name = newGroupName;
    }

    public void changeRemark(String newRemark) {
        this.remark = newRemark;
    }

    public void addSubAccount(String subAccount) throws MaxGroupMemberException, AlreadyJoinGroupException {
        if (subAccountList.size() < MAX_GROUP_NUMBER) {
            if (subAccountList.contains(subAccount)) {
                throw new AlreadyJoinGroupException();
            }
            this.subAccountList.add(subAccount);
        } else {
            throw new MaxGroupMemberException();
        }
    }

    public void removeSubAccount(String subAccount) {
        this.subAccountList.remove(subAccount);
    }

    public List<String> getSubAccountList() {
        return subAccountList;
    }

    protected void setSubAccountList(List<String> subAccountList) {
        this.subAccountList = subAccountList;
    }

    @Override
    public Set<String> validate() {
        return Entity.validate(this);
    }

    @Override
    public String toString() {
        return "UserGroup{" + "primaryAccountId='" + primaryAccountId + '\'' + ", name='" + name + '\'' + ", remark='"
                + remark + '\'' + ", subAccountList=" + subAccountList + '}';
    }
}
```

UserGroup提供了两个Constructor。由于一个UserGroup必须属于一个主账号，同时必须有一个组名称，那么我们一般需要提供这样一个Constructor。

> public UserGroup(Long primaryAccountId, String name)

对应的MyBatis ResultMap如下：

```xml
 <resultMap id="groupResult" type="userGroup">
        <constructor>
            <arg column="primary_account_id" javaType="long"/>
            <arg column="name" javaType="String"/>
            <arg column="remark" javaType="String"/>
        </constructor>
        <id column="id" property="id"/>
        <result column="gmt_create" property="gmtCreate"/>
        <result column="gmt_modify" property="gmtModify"/>
        <result column="is_deleted" property="isDeleted"/>
        <collection property="subAccountList" javaType="list" column="id" ofType="String"
                    select="selectSubAccountUser" fetchType="eager">
            <result column="sub_account_user"/>
        </collection>
    </resultMap>
```

### 戒掉Setter

DDD推荐使用更有表现力的方法名称，例如上面`UserGroup`的`changeName`，而不是`setName`。这两者的语义与使用心智是不一样的。`changeName`表示我已经有一个`name`，而现在要接受一个新的`name`，而`setName`是看不出来任何的业务语义的。

不用`setter`带来的一个问题就是Mybatis无法正确的识别对应的反射方法，如果使用属性注入，那么Mybatis要求一定要存在对应的`setter`。

解决办法有两个：

1. 如果这个属性是构造时可以附带的，那么就通过Mybatis的`constrcutor`进行注入，而不是`property`。
2. 如果这个属性不合适放到`Constructor`中，那么增加一个`protect`级别的`setter`方法。这样一来就会有两个注入点了，这也是Mybatis不适合DDD的原因之一。

### 处理关联关系

聚合里面一般多个其他实体或者值对象，这种关联关系时如何建立起来的呢？

如`UserGroup`中的`subAccountList`属性，这个代表了用户组中的子账号集合。通过`addSubAccount`和`removeSubAccount`来增加和删除组中的子账号。通过`setSubAccountList`来完成MyBatis注入子账号的集合，对应`ResultMap`中的`collection`元素。下面会在[值对象持久化](#值对象持久化)中讲到如何利用值对象来保存关联关系。

## 值对象持久化

值对象的持久化一般分为两种场景：

### 描述客观事物

大部分值对象是用来描述一种客观存在的事物，如Email、Phone、Url等等。这些对象的持久化需要使用专门的`TypeHandler`。如处理EMail可使用如下的`EmailAddressTypeHandler`。

```java
/**
 * EMailAddress值对象 Type Handler
 *
 * Created by Michael Jiang on 16-12-6.
 */
@MappedTypes(EMailAddress.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class EMailAddressTypeHandler extends BaseTypeHandler<EMailAddress> {
    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, EMailAddress eMailAddress,
            JdbcType jdbcType) throws SQLException {
        preparedStatement.setString(i, eMailAddress.email());
    }

    @Override
    public EMailAddress getNullableResult(ResultSet resultSet, String s) throws SQLException {
        return new EMailAddress(resultSet.getString(s));
    }

    @Override
    public EMailAddress getNullableResult(ResultSet resultSet, int i) throws SQLException {
        return new EMailAddress(resultSet.getString(i));
    }

    @Override
    public EMailAddress getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        return new EMailAddress(callableStatement.getString(i));
    }
}
```

这种场景下值对象是在实体或者聚合内部的，没有专属的数据表，只需要使用对应的TypeHandler完成值对象的存取就可以了。

### 描述关联关系

我们经常会遇到关联关系表，这种表对应着一种N:N的实体关系。我们可以利用值对象的特性来表示这种关联关系，因为关联关系一旦建立就不可改变直到你销毁它。

这种情况下值对象本身就对应了一张关系表，无法通过TypeHandler来完成持久化。我们需要使用`IdentifiedValueObject`这个带ID的值对象。

```java
/**
 * 带id的值对象 Created by jiangwenkang on 16-11-7.
 */
public abstract class IdentifiedValueObject<V extends IdentifiedValueObject> implements ValueObject<V> {
    private static final long serialVersionUID = -7815856088988915247L;

    protected Long id;
    protected Date gmtCreate;

    protected Long getId() {
        return id;
    }

    protected void setId(Long id) {
        this.id = id;
    }

    public Date getGmtCreate() {
        return gmtCreate;
    }

    protected void setGmtCreate(Date gmtCreate) {
        this.gmtCreate = gmtCreate;
    }

    @Override
    public String toString() {
        return "IdentifiedValueObject{" + "id=" + id + ", gmtCreate=" + gmtCreate + '}';
    }
}
```

然后具体的关系值对象继承这个`IdentifiedValueObject`即可。之所以会存在这个`IdentifiedValueObject`是因为我们关系表中还会存在`主键ID`,`创建时间`之类的通用字段，我们把这些字段封装在这个带ID值的基类中，这样我们具体的值对象就不需要冗余这些字段了。

综上所述，DDD中的持久化要考虑的东西更多一些。从模块的依赖关系、持久化的时机、ORM的选型都需要仔细考量。大部分情况下我们都是对实体和聚合进行持久化，需要利用好Constructor、尽量避免使用Setter、利用Collection来加载聚合关联的实体对象。一般情况下值对象是跟着实体一起进行持久化的，某些情况下需要专门对值对象进行持久化，那么我们可以构造一个带ID的值对象基类来屏蔽技术层面的细节，从而保持具体值对象的纯粹性。

---
趁着最近换工作之际，可以静下心来写点DDD的东西了。最近微服务和中台概念的流行，似乎又让国内的开发者重拾DDD的开发理念，这让我感到有些兴奋。后续我将更多分享关于DDD落地方面的问题。
