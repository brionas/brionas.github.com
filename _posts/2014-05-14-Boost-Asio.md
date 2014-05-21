---
layout: post
title: Boost Asio介绍 
description: Boost Asio介绍 
category: C++ Boost
tags: Boost Asio介绍
refer_author: qsun
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}


###一  简介 

        Boost Asio ( asynchronous input and
output)关注异步输入输出。Boost
Asio库提供了平台无关性的异步数据处理能力（当然它也支持同步数据处理）。一般的数据传输过程需要通过函数的返回值来判断数据传输是否成功。Boost Asio将数据传输分为两个独立的步骤：


   1.  采用异步任务的方式开始数据传输

   2.  将传输结果通知调用端

       与传统方式相比，Boost Asio优点在于程序在数据传输期间不会阻塞。

###二 I/O services and I/O objects

        应用程序采用Boost.Asio进行异步数据处理主要基于 I/O services 和
I/O objects。I/O
services抽象系统I/O接口,提供异步数据传输的能力，它是应用程序和系统I/O接口的桥梁。I/O
objects 用来初始化某些特定操作，如TCP
socket，提供TCP方面可靠数据传输的能力。Boost.Asio只提供一个类实现 I/O
services, boost::asio::io\_service。提供多个I/O
objects对象，如boost::asio::ip::tcp::socket（用来收发数据）和
boost::asio::deadline\_timer（用来提供计时器的功能，计时器可以在某个时间点或经历某个时间段后生效）。由于计时器不涉及到
太多的网络方面的内容，用其举例说明一下Boost.Asio的初步用法：

	#include <boost/asio.hpp>
	#include <iostream>
	 
	void handler(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	int main()
	{
	  boost::asio::io_service io_service;
	  boost::asio::deadline_timer timer(io_service, boost::posix_time::seconds(5));
	  timer.async_wait(handler);
	  io_service.run();
	}

        首先定义一个boost::asio::io\_service来初始化I/O
objects实例timer. 由于I/O objects对象构造函数的第一个参数都是一个I/O
service对象，timer也不例外，timer的第二个参数可以是时间点或时间段，上例中是一个时间段。

   async\_wait()表示时间到后调用handler，应用程序可以执行其它操作而不会阻塞在
sync\_wait处。async\_wait()是一种非阻塞的函数，timer也提供阻塞的函数wait()，由于它在调用结束后返回，因而不需要
handler作为其参数。

       可以发现，在调用 async\_wait()之后，I/O
service调用了run方法。这是必须的，因为我们必须把控制权交给操作系统，以便在5s之后调用handler方法。也就是
说，async\_wait()在调用后立即返回，run()调用后实际阻塞了。许多操作系统都是通过一个阻塞的函数来实现异步的操作。这令人费解，但是，
看了下面的例子，你也许就会发现这其实很好理解。

   

	#include <boost/asio.hpp>
	#include <iostream>
	 
	void handler1(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	void handler2(const boost::system::error_code &ec)
	{
	  std::cout << "10 s." << std::endl;
	}
	 
	int main()
	{
	  boost::asio::io_service io_service;
	  boost::asio::deadline_timer timer1(io_service, boost::posix_time::seconds(5));
	  timer1.async_wait(handler1);
	  boost::asio::deadline_timer timer2(io_service, boost::posix_time::seconds(10));
	  timer2.async_wait(handler2);
	  io_service.run();
	}

        从上面我们可以发现handler2的调用是在handler1调用5s后进行的，这
也正是异步的精髓所在：timer2没有等到timer1计时5s后才启动。之所以异步操作看上去需要阻塞的run()方法，是因为我们必须阻止程序终
结：如果run()不阻塞，main()就会马上退出了。由于run()会阻塞进程，如果不想阻塞当前进程，我们可以在另外一个进程中调用run().

###三 可扩展性和多线程

     
采用Boost.Asio开发应用程序和通常的C++风格不同：在Boost.Asio中需要较长时间才返回的functions的调用不是有序的，这是
因为对于阻塞的方法，Boost.Asio
采用异步操作。通过上面所示的handler形式来实现当某个操作完成后就必须被调用的方法。采用这种方法的缺点是顺序执行函数的人为分割，使得相应的代
码更难理解。

     
采用Boost.Asio库的主要目的是为了实现程序的高效，不用等待某个function结束，应用程序可以在这期间进行其它任务的运行，例如，开始某个可能需要花费一段时间才能完成的操作。

     
扩展性是指程序有效的利用计算机的其它资源。推荐采用Boost.Asio的原因之一是持续时间长的操作不会阻塞其它操作，另外，由于现有计算机一般都是多核的，采用多线程可以有效的提升程序的可扩展性。

      
在上面的程序中，采用boost::asio::io\_service调用run()方法，和boost::asio::io\_service相关联的
handler将会在同一线程内触发。通过采用多线程，应用程序可以同时调用多个run()方法。一旦某个异步操作完成，对应的I/O
service将会执行某个线程中的handler方法。如果第二个异步操作在第一个结束后很快完成，I/O
service 可以立刻执行其对应的handler,而不用等待第一个handler执行完毕。

	#include <boost/asio.hpp>
	#include <boost/thread.hpp>
	#include <iostream>
	 
	void handler1(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	void handler2(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	boost::asio::io_service io_service;
	 
	void run()
	{
	  io_service.run();
	}
	 
	int main()
	{
	  boost::asio::deadline_timer timer1(io_service, boost::posix_time::seconds(5));
	  timer1.async_wait(handler1);
	  boost::asio::deadline_timer timer2(io_service, boost::posix_time::seconds(5));
	  timer2.async_wait(handler2);
	  boost::thread thread1(run);
	  boost::thread thread2(run);
	  thread1.join();
	  thread2.join();
	}

        上一小节中的程序现在被转换为了一个多线程的程序。通过使用定义在
boost/thread.hpp中的boost::thread类，在main()中创建了两个线程。这两个线程为同一个I/O
service调用run()。这样做的好处是，一旦独立的异步操作完成，I/O
service可以有效利用两个线程来执行handler方法。

       
上面例子中的两个timer都是让时间停顿5s。由于有两个线程，handler1和handler2可以同时执行。如果timer2在停顿期
间，timer1对应的handler1仍然在执行，那么handler2将会在第二个线程内执行。如果handler1已经结束了，那么I/O
service将会自由选择线程来执行handler2。

       
线程可以增加程序的执行效率。因为线程跑在CPU的核上，创建比CPU核数还多的线程是没有意义的。这可以保证每个线程跑在各自的核上，而不会出现多个线
程为抢占某个核的”核战争”。

       
应该注意到，采用线程也不是总是合理的。执行上面的代码可能会导致各自信息在标准输出流上产生混合的输出，这是因为两个hander方法可能会并行的执行
到，而他们访问的是一个共享的标准输出流std::cout。对共享资源的访问需要进行同步，从而保证每条消息完全输出后，另外一个线程才能够向标准输出
写入另外一条消息。如果线程各自的handler不能独立的并行执行(handler1的输出可能影响到handler2)，在这种场景下使用线程不会带
来什么好处。

        基于Boost.Asio来提高程序的可扩展性推荐的方法是：采用单个I/O
service多次调用run()方法。当然，也有另外的方法可以选择：可以创建多个I/O
service，而不是将所有的线程都绑定到一个I/O service上。每个I/O
service对应于一个线程。如果I/O
service的个数和计算机的核数相匹配，异步操作将会在各自对应的核上运行。下面是一个这样的例子：

	#include <boost/asio.hpp>
	#include <boost/thread.hpp>
	#include <iostream>
	 
	void handler1(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	void handler2(const boost::system::error_code &ec)
	{
	  std::cout << "5 s." << std::endl;
	}
	 
	boost::asio::io_service io_service1;
	boost::asio::io_service io_service2;
	 
	void run1()
	{
	  io_service1.run();
	}
	 
	void run2()
	{
	  io_service2.run();
	}
	 
	int main()
	{
	  boost::asio::deadline_timer timer1(io_service1, boost::posix_time::seconds(5));
	  timer1.async_wait(handler1);
	  boost::asio::deadline_timer timer2(io_service2, boost::posix_time::seconds(5));
	  timer2.async_wait(handler2);
	  boost::thread thread1(run1);
	  boost::thread thread2(run2);
	  thread1.join();
	  thread2.join();
	}

        上面采用一个I/O service的程序被重写成了采用两个I/O
service的程序。程序还是有两个线程，只不过每个线程现在对应的是不同的I/O
service，同时，timer和timer2也和不同的I/O service相对应。

        程序的功能和以前的相同。拥有多个I/O
service在某种情况下是有益的，理想情况下，每个I/O
service拥有自己的线程，跑在各自的核上，这样，不同的异步操作以及其对应的handler方法可以在局部执行。这样就不会出现上面两个线程共享同
一个标准输出的情况了。如果没有访问外部数据和方法的需要，每个I/O
service就等同于一个独立的程序。在制定优化策略前，由于需要了解硬件，操作系统，编译器以及潜在的瓶颈等相关知识，采用多个I/O
service只有在确实可以从中获益的情况下才使用。   

###四 网络编程

       
尽管Boost.Asio是一个可以进行异步数据处理的库，它主要用在网络编程上。这是因为Boost.Asio在添加其它I/O
objects前，很早就支持网络功能了。网络功能是一个完美的异步处理的例子，因为数据在网络上的传输可能需要花费更多的时间，相应的应答或出错情况往
往不是直接可以获得的。

        Boost.Asio提供很多I/O
objects来开发网络程序。下面的例子采用boost::asio::ip::tcp::socket来建立和不同PC间的连接，同时下载
brioncd首页--就像浏览器访问brioncd.briontech.com时做的一样。

	#include <boost/asio.hpp>
	#include <boost/array.hpp>
	#include <iostream>
	#include <string>
	 
	boost::asio::io_service io_service;
	boost::asio::ip::tcp::resolver resolver(io_service);
	boost::asio::ip::tcp::socket sock(io_service);
	boost::array<char, 4096> buffer;
	void read_handler(const boost::system::error_code &ec, std::size_t bytes_transferred)
	{
	    if (!ec)
	    {
	        std::cout << std::string(buffer.data(), bytes_transferred) << std::endl;
	        sock.async_read_some(boost::asio::buffer(buffer), read_handler);
	    }
	}
	void connect_handler(const boost::system::error_code &ec)
	{
	    if (!ec)
	    {
	        boost::asio::write(sock, boost::asio::buffer("GET / HTTP 1.1\r\nHost: brioncd.briontech.com\r\n\r\n"));
	        sock.async_read_some(boost::asio::buffer(buffer), read_handler);
	    }
	}
	 
	void resolve_handler(const boost::system::error_code &ec, boost::asio::ip::tcp::resolver::iterator it)
	{
	    if (!ec)
	    {
	        sock.async_connect(*it, connect_handler);
	    }
	}
	 
	int main()
	{
	    boost::asio::ip::tcp::resolver::query query("brioncd.briontech.com", "80");
	    resolver.async_resolve(query, resolve_handler);
	    io_service.run();
	}

        

       
上面例子中，最值得关注的是三个handler方法：一旦建立连接以及接收到数据，将会分别调用connect\_handler()和
read\_handler()，那么为什么需要resolve\_handler()？

       
因特网采用IP地址来标识不同的计算机。IP地址实质上是一连串不好记的数字，记住域名比记住数字好的多。为了用域名访问计算机，必须将域名转换为对应的
IP地址，这个过程也就是名称解析。在Boost.Asio中，用boost::asio::ip::tcp::resolver来实现名称解析。

       
名称解析需要联网才能完成。一些特定的PC(DNS服务器),负责将域名转换为IP地址。boost::asio::ip::tcp::resolver
I/O
object所完成的事情就是连接外网获取域名对应的IP，由于名称解析不是发生在本地，因而它也是作为一个异步操作实现的。一旦名称解析完成（不管成功
还是返回失败），就会调用resolve\_handler()。

       
由于获取数据的前提是成功建立链接，而成功建立链接的前提又是成功进行名称解析，这样不同的异步操作将会在不同的handler内部进行。
resolve\_handler()利用由it提供的获取到的ip地址，通过I/O object
sock来建立连接。在connect\_handler()内，采用sock来发送HTTP请求来初始化数据接收。由于所有的操作都是异步的，各自的处理
方法的函数名是通过参数的形式进行传递的。由于handler的不同，需要不同的参数，例如迭代器it,指向获取到的IP地址；缓存buffer,储存接
收到的数据。

       
程序开始运行时就会创建一个query对象并用域名和端口号对齐进行初始化，接着query对象传递给async\_resolve()方法去进行名字解
析。最后，main()方法调用I/O
service的run()方法来将异步操作的控制权交给操作系统。

       
一旦名称解析完成，resolve\_handler()将会被调用，首先它将检查名称解析是否成功，如果成功，包含各种错误情形的对象object
ec,将会被设置为0。只有在这种情况下，程序才会访问sock来创建一个连接。链接需要的IP地址由第二个参数it提供。

       
调用完async\_connect后，connect\_handler()又会自动被调用。在connect\_handler()内部，同样通过
object
ec对象来判断连接是否成功建立。如果连接成功建立，会调用async\_read\_some()方法来初始化对应socket上的read操作。数据存储
在第一个参数表明的buffer内部。在上面的例子中，buffer是boost::array类型的，定义在boost/array.hpp中。

       
read\_handler()方法在有数据接收并储存到buffer中后就立刻被调用。接收到的数据大小通过参数bytes\_transferred可以
得到。同样的，通过object
ec对象来判断接收过程中是否出错。如果接收成功，数据将会重定向到标准输出。

        一旦数据写到标准输出
read\_handler()将会再次调用async\_read\_some()，这是因为数据不会一次读完。

        上面的例子用来获取网页内容，下面的例子则是实现了一个简单的web
server.最重要的区别是，程序不会连接到别的服务器，而是等待别人向其发起连接，如本机IP是192.168.100.100，我们在浏览器中输入
[http://192.168.100.100](http://192.168.100.100/),将会出现Hello,world!。

	#include <boost/asio.hpp>
	#include <string>
	 
	boost::asio::io_service io_service;
	boost::asio::ip::tcp::endpoint endpoint(boost::asio::ip::tcp::v4(), 80);
	boost::asio::ip::tcp::acceptor acceptor(io_service, endpoint);
	boost::asio::ip::tcp::socket sock(io_service);
	std::string data = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, world!";
	 
	void write_handler(const boost::system::error_code &ec, std::size_t bytes_transferred)
	{
	}
	 
	void accept_handler(const boost::system::error_code &ec)
	{
	  if (!ec)
	  {
	    boost::asio::async_write(sock, boost::asio::buffer(data), write_handler);
	  }
	}
	 
	int main()
	{
	  acceptor.listen();
	  acceptor.async_accept(sock, accept_handler);
	  io_service.run();
	}

        上面用带有协议和端口号的endpoint对象初始化了的I/O object
acceptor，acceptor主要用来等待从其它PC过来的连接。在初始化acceptor后，main()函数中首先调用listen()方法来
将acceptor设置为接收模式，然后调用async\_accept来等待连接。用来接收和发送数据的socket在第一个参数中进行了指定。

        
一旦有计算机试图建立连接，accept\_handler将会自动被调用。如果连接请求成功，可以独立运行的函数
boost::asio::async\_write将会被调用，它将存储在data中的数据通过socket发送出去，boost也提供了
async\_write\_some方法，只要有数据发送出去，该方法将会触发相应的handler方法，handler方法需要计算已经发送了多少数据，
同时再次触发async\_write\_some，直到数据都被传送出去。采用async\_write方法可以避免上面分析中的复杂过程，因为
async\_write的异步操作只有在所有的数据都发送出去后才会停止。

       
在上面的例子中，一旦所有的数据都发送出去了，空方法write\_handler将会被调用。由于所有的异步操作都结束了，整个应用程序就结束了。建立的连接也相应的关闭了。

#######注：本文为译文，原文链接
 [http://en.highscore.de/cpp/boost/index.htm](http://en.highscore.de/cpp/boost/index.htm)
chapter7


