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

我推荐的做法是将接口放到Domain中，而Reposistory的实现放到Infrastructure里面，如下图所示。

{% asset_img DDD-Repository.png DDD-Repository %}

直接将Reposistory的Interface和业务模型放到同一个package下面就可以了，让业务模型和负责业务模型的存储接口放到一个目录下。但这里感觉有点“坏味道”的产生，好像存储层的内容“侵入”到了领域层。我举得这是可以接受的，因为在领域层的模型是以业务维度划分的，实体与存储库是可以共存。反而，如果让领域层依赖基础设施，那么这种侵入性将更加难以控制，很容易造成深度耦合。

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
    public void changeRoleRemark(long primaryAccountId, String roleName, String remark) throws RoleNotFoundException {
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

上诉的持久化方式可以满足大部分项目的需求，但是总有特殊情况。

## ORM框架

## 实体持久化

## 值对象持久化



---
趁着最近换工作之际，可以静下心来写点DDD的东西了。最近微服务和中台概念的流行，似乎又让国内的开发者重拾DDD的开发理念，这让我感到有些兴奋。后续我将更多分享关于DDD落地方面的问题。
