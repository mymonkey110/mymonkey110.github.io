---
title: 利用Tair实现分布式并发锁
date: '2014-12-23'
tags: distributed lock
cateogries: 分布式
description: 最近大量使用到了Tair来控制并发，分享一下心得。
---
  
最近大量使用到了Tair来控制并发，有点心得，总结如下。

* 利用Tair实现全局并发锁
	
   现在基本上线上服务器都是集群环境，那么当我们需要对中心化数据（例如:Tair、数据库）的同一内容进行读写时就会碰到并发问题，这是一种非常常见的需求。解决并发问题的方法无非有两种，在并发点控制并发或者在并发源头控制。
![_](http://img3.tbcdn.cn/L1/461/1/48437662f3c9714c3dafff4c0a29fe84215571ba)

   图画的有点丑。并发点控制最常用的一种方式就是使用锁，每个需要访问数据的线程都需要先获取锁，然后才能去访问数据库。根据获取锁的策略的不同，又可以根据不同纬度分为乐观锁、悲观锁，忙等、闲等，互斥锁、读写锁等等。
   
   在并发源头控制就是利用第三方的工具，一般是消息队列来将并发访问串行化，然后由统一的数据操作者来访问数据。消息队列的使用不在本文的讨论范文内。比较有名的开源消息队列有,[RabbitMQ],[ZeroMQ]。当然,公司内部也有对应的产品，如[Notify],[MetaQ]。
   
  由于在分布式环境中，要实现全局的并发锁，那么我们必须借助第三方的服务来进行协调。数据库和缓存经常会成为我们的优先选择。出于性能的考虑，一般选用缓存来实现全局并发锁，其中的关键也就是借助Tair的[Version]控制，相比已经有很多人已经在这样做了。Tair提供了以下API：
  
  `ResultCode put(int namespace, Object key, Serializable value, int version, int expireTime)`
  
  利用该API实现并发控制轻而易举,伪代码如下：
  
  ```	    
  	    //加锁
  		public boolean lock(String key, int timeOut) {
            ResultCode rc = tairManager.put(NAMESPACE, key, DEFAULT_VALUE, INIT_VERSION, timeOut);
            return rc!=null&&ResultCode.SUCCESS.equals(rc)?true:false;
        }
        
        //解锁
        public boolean unlock(String key) {
            ResultCode rc = tairManager.invalid(NAMESPACE, key);
			return rc!=null&&ResultCode.SUCCESS.equals(rc)?true:false;
	    }
  ```
  	
  这主要是利用了Tair的VERSION特性。如果KEY不存在的话，传入一个固定的初始化VERSION，Tair会在保存这个缓存的同时设置这个缓存的VERSION为你传入的*VERSION+1*；然而KEY如果已经存在，Tair会校验你传入的VERSION是否等于现在这个缓存的VERSION，如果相等则允许修改，否则将失败。  其过程如下图所示：
  
![global_lock](http://img2.tbcdn.cn/L1/461/1/c8f6cdafb3401daf724c9e0649f69ce9de71b057)
  
   这是一个很通用的过程，但是却能涵盖大部分的场景。其实理解这个过程非常简单，这里可以把其想象成受精卵形成的过程。虽然有成千上万个精子会进入卵巢，但当第一个精子和卵子结合以后就会形成一层隔离层，以阻止其他精子的进入。而这里的隔离层就类似于TAIR的[VERSION]。如果想知道更多过程可以参考[VERSION]的文档。

[Version]:http://baike.corp.taobao.com/index.php/Faq#.E4.BB.80.E4.B9.88.E6.98.AFversion.EF.BC.9F
   [RabbitMQ]:http://www.rabbitmq.com/
   [ZeroMQ]:http://zeromq.org/
   [MetaQ]:http://gitlab.alibaba-inc.com/middleware/metaq3/wikis/what_is_metaq
   [Notify]:http://baike.corp.taobao.com/index.php/Notify

* 利用Tair实现全局TOP-N并发锁
  
  全局TOP-N并发锁是我自己想出来的一个名字，有点不明觉厉吧。实际业务中我们可能会遇到这样一种情况，在短时间内会有大量的并发来获取某种资源，但是我们这个资源又有数量限制。例如，抢火车票，在某一时刻将1000张火车票发出去，假如有大量的用户在同一时间来抢这些火车票就会形成并发，同时我们又有着很高的性能要求。以抢火车票为例，下面是我的思考过程。
  
  因为我需要控制并发，要告诉第1001个用户你没有抢到，那么我肯定需要一个计数器来保存火车票发售的实时情况，那么很容易就写出了以下伪代码：
  
  ```
	  	if(get(ticker_counter)<1000)) {
	  	    bool lockFlag=lock(key,60);
  			if(lockFlag) {
  				int counter=get(ticker_counter);
  				if(++counter<1000) {
  					set(ticker_counter=counter)
  				}
  				unlock(key);
  			}
  		}
  	```
  		
  首先获取当前计数器的值，如果>=1000则直接失败返回,表示已经被抢完了，但是如果<1000，表示还没被抢完，则尝试去获取全局锁，如果获取成功则增加计数器的值，注意此时是需要再获取一次计数器的。但是这样会有一个明显的问题，就是当A获取了锁，正在执行增加计数器操作时，B也去尝试获取锁，此时必然是失败的。但是我现在就应该告诉他你已经失败了吗，你没有机会获得这张火车票了吗？显然不是。因为我们允许获取的资源是一个范围，那么当没有明确地表示现在资源已经超出这个返回了或者没有资源了，那么现在所有尝试得到资源的线程或者用户都是有机会的。此时，书中的一个概念浮现出来——信号量。这种业务场景正好是信号量技术能够解决的。但是在分布式环境下如何解决这个问题呢。
  	
  我想到了Linux环境下编程时的很多技术。其中就有一个很这个业务场景非常相似的API，就是POSIX系列里面的`pthread_cond_wait()`和`pthread_cond_signal()`。前者会一直阻塞直到等待的资源变为可用，而后者会唤醒一个正在等待某个资源的线程。如果有有这两个语义的API存在的话就会变得非常简单，伪代码将变为：
  
  ```		
		 if(get(ticker_counter)<1000)) {
	  	    pthread_cond_wait();
  			int counter=get(ticker_counter);
  			if(++counter<1000) {
  				set(ticker_counter=counter)
  			}
  			pthread_cond_signal();
  			}
  		}
  ```
  	
  只可惜在分布式环境下没有这两个语义的API操作存在，那么久不得不转化思维。之所以我需要这两个语义的API存在是因为我希望在A线程完成工作以后，将这个状态/消息通知到其他在等待的线程，并且这些线程是分布式的。其实这里是可以使用到消息模型的。notify会选择集群中的一台服务器投递消息，这就可以作为唤醒操作。所有的worker一开始都去监听notify的消息，直到其中一个worker收到，然后去checkAndInc(counter)，最后再发出一个消息，如此循环就能达到目的。最后只需要增加一个Trigger,在最开始执行的时候直接去执行，而不用等待notify消息，就能完成完整的流程。但是，如此简单地一个功能，真的要实现的这么复杂吗？当然不行，什么时候都要坚持KISS原则。
  	
  其实，我最终的目的很简单，就是增加一个计数器的值，然后达到某一上线时希望能够得到一个错误返回。因为在做以前一个项目时使用到了Tair中计数器的功能，带着侥幸的心理重新去找Tair的API，居然发现了这个重要API:
  	
  	`Result<Integer> incr(int namespace, Serializable key, int value, int defaultValue, int expireTime, int lowBound, int upperBound)`
  	
  当我看到这个API的时候感悟良多，在此还是要感谢一下设计这个API的作者，因为这个API的设计就是为这种业务场景而生的。这个`incr()`操作可以指定一个范围段，如果value值不在这个范围段中就会报错。有个这个API那么伪代码就简化成以下：
  	
  ```	
  		result = tairManager.incr(NAMESPACE, key, 1, 0, 60, 0, 1000);
        return ResultCode.SUCCESS.equals(result.getRc())? true : false
  ```
     
   这个多么的简洁和优雅,而且又有着很高的性能。如果有着类似的业务场景，推荐大家不妨试一下这个API。
   
 * 一点思考
 
 现在分布式计算越来越受到重视，随着去IOE的深入，大型机的时代一去不复返。但是分布式计算的流行使得程序员思考问题的方式也在发生改变，以前在单机上运行很好地系统，拿到分布式环境下可能就会出现各种问题。虽然整体架构发生了很大的变化，但是单机时代的很多思想还是值得我们去借鉴的。就比如信号量计算，PV操作。以前这些技术靠操作系统去实现就好了，但是在分布式环境下就很难实现这些以前看似很自然的功能。从某种程度上，这又为我们的中间件技术指明了发展的道路。如果哪一天业务程序员能在分布式环境中像在单机环境里编程，那分布式技术的发展就达到了一个新的高度。
 
        
