---
layout: post
title: C#网络编程初步之TCP
date: '2011-10-03 10:22:40'
tags: C# tcp udp network
categories: C#
---

 
**阅读背景：本文针对有C#的初学者而写的，主要讲解如何利用C#进行网络编程。
如果你已经有一些网络编程的经验（只需要懂得网络编程的基本常识即可），并且理解C#的基本语法，那么这篇文章可以很快地带你进入C#网络编程的世界。
如果你的基础不好，也不要紧，我相信这篇文章也会有你需要的内容。**
 
网络编程基础复习：

   ![TCP-Module](http://hi.csdn.net/attachment/201110/2/0_1317580064Gkl4.gif)
      
 图1. TCP编程基本模型
 
 相信很多人看到图1应该不会陌生，这是一个利用TCP进行通信的经典模型图。我想大家都应该把这张图记在心中。
 在此我就不讲述上图中每个API的意思了，百度一下，你就知道。我想说的是，难道你不觉得这么编程很累吗? 
 我们需要去调用每个API函数，然后每个判断返回值是多少，如果你忘记了哪个API的参数形式还得去查MSDN，这种时间花费是巨大的，尤其当你做应用层的快速开发时。
 
 
 图2是利用UDP通信时的编程基本模型，这个模型较为简单，但是应用极为广泛，相比TCP而言，我本人觉得利用UDP通信是一门更为高深的技术，因为它是无连接的，
 换言之，它的效率与灵活度就更高些。

   ![UDP-Module](http://hi.csdn.net/attachment/201110/2/0_131757970540yA.gif)
   
图2. UDP编程基本模型
 
   在此我补充一点，关于何时利用TCP通信、何时利用UDP通信的问题。他们的特性其实已经决定了他们的适用范围。
   在进行大数据量、持续连接时，我们使用TCP，例如FTP协议；而在进行小规模数据、突发性高的通信时，我们使用UDP，例如聊天程序。
   但是，这并不是绝对的事情。例如流媒体通信，它是大数量、持续的通信，但是使用的是UDP协议，为什么呢？
   ——因为我们不关心丢失的帧，人的肉眼是无法识别出少量的帧丢失的。那么使用UDP通信就可以大幅度提高效率，降低网络负载。
 
 
### C#之TCP编程

**如何创建一个套接字?**

我们先来看看利用Winsock2是如何建立一个套接字的：

首先，我们要加载套接字库，然后再建立套接字。大致代码如下：

    WORD wVersion=MAKEWORD(2,2);
    WSADATA wsaData;
    if(WSAStartup(wVersion,&wsaData))
    {
    WSACleanup();
    returnFALSE;
    }
     
    m_sock=WSASocket(AF_INET,SOCK_DGRAM,IPPROTO_UDP,NULL,0,0);
    if(m_sock==INVALID_SOCKET)
    {
            MessageBox("创建套接字失败！");
            return FALSE;
    }

  难道你不觉得利用Winsock2创建一个套接字很费劲吗？如果你在Linux环境中变成倒是可以省掉加载套接字的部分，
  但是却只能反复的调用API，这样也是很费时的事情。那我们再看看看利用C#是如何帮你简化工作的。这里我会介绍TCPClient类。

  ![msdn](http://hi.csdn.net/attachment/201110/2/0_131758022652u1.gif)
  
  以上是从MSDN上截取的一段话，可见我们利用TCPClient还处理与TCP通信相关的操作。TCPClient有四个构造函数，每个构造函数的用法是有不同的。这里我补充一个知识，那就是端地址在C#中描述。
  我们知道，我们用一个IP地址和一个端口号就可以表示一个端地址。在C#中我们利用IPEndPoint类来表示一个端地址，本人经常利用如下的构造函数来创建一个IPEndPoint类。
 
    IPEndPoint localEP = new IPEndPoint(IPAddress.Parse("127.0.0.1"),6666);

这样来表示一个端地址是不是比创建一个struct sockaddr_in的结构体来的快呢？
 
**如何绑定一个端地址？**

  我们已经创建了一个端地址，也构造了套接字（TCPClient类），那么如何将二者绑定起来呢?也许你已经发现了，在建立TCPClient的时候我们其实就可以绑定端地址了。
  如果你使用的TCPClient tcp_Client=new TCPClient()的构造函数来创建的TCPClient,那么系统会认为你没有人为的制定端地址，而会自动帮你制定端地址，在创建客户端的TCPClient时我们常常这样做，
  因为我们不关心客户端的端地址。如果是服务器监听呢？在服务器监听时我们会使用例外一个类，叫做TCPListener，接下来我会讲到。我们可以利用TCPClient(IPEndPoint)来构造一个绑定到固定端地址的TCPClient类。例如：
        
        TcpClient tcp_Client = new TcpClient(localEP);

**如何监听套接字？**

   到现在为此我们还没讨论如何监听一个套接字。在传统的socket编程中，我们创建一个套接字，然后把它绑定到一个端地址，而后调用Listen()来监听套接字。而在C#中，我们利用TCPListener来帮我们完成这些工作。让我们先来看看如何在C#监听套接字。
    
    IPEndPointlocalEP = new IPEndPoint(IPAddress.Parse("127.0.0.1"),6666);
    TcpListenerListener = new TcpListener(localEP);
    Listener.Start(10);
 
  我们首先创建需要绑定的端地址，而后创建监听类，并利用其构造函数将其绑定到端地址，然后调用Start(int number)方法来真正实施监听。这与我们传统的socket编程不同。以前我们都是先创建一个socket，然后再创建一个sockaddr_in的结构体。我想你应该开始感受到了C#的优势了，它帮我们省去了很多低级、繁琐的工作，让我们能够真正专注于我们的软件架构和设计思想。

**如何接受客户端连接？**

  接听套接字后面自然就是接受TCP连接了。我们利用下面一句话来完成此工作：
  
    TcpClient remoteClient =Listener.AcceptTcpClient();
  
  类似于accept函数来返回一个socket,利用TCPListener类的AcceptTcpClient方法我们可以得到一个与客户端建立了连接的TCPClient类，
  而由TCPClient类来处理以后与客户端的通信工作。我想你应该开始理解为什么会存在TCPClient和TCPListener两个类了。
  这两个类的存在有着更加明细的区分，让监听和后续的通信真正分开，让程序员也更加容易理解和使用了。
 
这里我还得补充一点：监听是一个非阻塞的操作`Listener.Start()`，而接受连接是一个阻塞操作`Listener.AcceptTcpClient`。
 
 
说了这么多，还不如来个实例来的明确。接下来，我会通过一个简单的控制台聊天程序来如何使用这些。先贴代码吧！

服务器端：

    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Net;
    using System.Net.Sockets;
     
    namespace Demo
    {
        class Program
        {
            static void Main(string[]args)
            {
                byte[]SendBuf = Encoding.UTF8.GetBytes("Hello,Client!");    //发给客户端的消息；
                IPEndPointlocalEP = new IPEndPoint(IPAddress.Parse("127.0.0.1"),6666);        //本地端地址
                TcpListenerListener = new TcpListener(localEP);            //建立监听类，并绑定到指定的端地址
                Listener.Start(10);           //开始监听                                                            
                Console.WriteLine("Server is listening...");                           
                TcpClientremoteClient = Listener.AcceptTcpClient();  //等待连接（阻塞）
                Console.WriteLine("Client:{0} connected!",remoteClient.Client.RemoteEndPoint.ToString()) ;     //打印客户端连接信息；
                remoteClient.Client.Send(SendBuf);     //发送欢迎信息；
                remoteClient.Close();                  //关闭连接；
            }
        }
    }

 
客户端:

    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Net;
    using System.Net.Sockets;
    
    namespace Demo_Client
    {
        class Program
        {
            static void Main(string[] args)
            {
                byte[] RecvBuf=new byte[1024];                    //申请接收缓存；
                int RecvBytes = 0;                                            //接收字节数；
                string recvmsg=null;                                      //接收消息；
    
                IPEndPoint remoteEP = new IPEndPoint(IPAddress.Parse("127.0.0.1"), 6666);      //远程服务器端地址；
                TcpClient remoteServer = new TcpClient();                   //创建TCPClient类来与服务器通信；
                remoteServer.Connect(remoteEP);                                  //调用connect方法连接远端服务器；
                Console.WriteLine("I'm using {0}.", remoteServer.Client.LocalEndPoint);          //打印自己使用的端地址；
                RecvBytes=remoteServer.Client.Receive(RecvBuf);                           //接受服务器发送过来的消息；
                recvmsg=Encoding.UTF8.GetString(RecvBuf,0,RecvBytes);          //将接受到的字节码转化为string类型；
                Console.WriteLine("Server says:{0}.", recvmsg);              //打印欢迎信息；
            }
        }
    }

 
   在C#网络编程中，我们要用到两个名空间，分别是System.Net和System.Net.Socket。
   可能有人会有这样的疑惑，干嘛要申请一个Byte数组。我们知道，在传统socket编程中，我们都是用char*来发送或者接受消息的，
   其实char*和Byte[]是同源的。他们都是一个Byte，而使用Byte[]能更易于人们理解和转化为其他类型。我们知道网络间传输的字节流，而Byte[]刚好符合了这个思想。
   如果对以上类的用法不理解或者不熟悉的话，建议查看MSDN，上面讲解的很详细。
 
现在看看运行效果：

![result](http://hi.csdn.net/attachment/201110/2/0_1317579792B9CZ.gif)

图3 运行效果（左为服务器，右为客户端）
 
  好啦，到这里我们C#网络编程初步之TCP基本上算告一段落了，我只讲解了最为基础的部分，仅做抛砖引玉的作用。每个类的使用千变万化，希望你能找到最适合自己使用方法。现在你可以对比以前类似程序的代码了，看看我前面有没有说错。而且，越到后来你会越来越体会到C#人性化的一面。
 
  后期的博文中，我会更新C#网络编程初步之UDP.本人更喜欢利用UDP来进行通信，至于为什么我已经说过了。以后，我会逐步写一些网络编程的高级内容，例如异步通信、多线程编程，并关注程序员经常遇到的一些棘手问题，比如TCP边界的确定等等。有机会，我也会同大家讨论网络编程中常用的软件设计思想与架构。

（本文图1、图2来自互联网，有部分信息来自MSDN。如需转载本文，请注明出处！谢谢）

