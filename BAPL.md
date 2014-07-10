Linux下Boost.Asio Proactor模式实现分析
===========================================================================


**背景：**

epoll的实现是基于回调的，如果fd有期望的事件发生就通过回调函数将其加入epoll就绪队列中，用户针对该队列中的文件句柄发起相应操作，如read等，此时数据真正才会开始从内核buffer写入应用buffer中，整个过程是一种同步IO。而Boost.Asio的说明文档中明确其采用Proactor模式实现了异步IO，也就是说用户在发起async\_read后，可以去进行其它操作，数据将会从内核buffer写入应用buffer，数据拷贝完毕会调用用户提供的回调函数。

**问题：**

Boost.Asio在Linux下封装epoll这种同步接口是如何做到异步IO的呢？通过下面的分析，我们会发现，Boost.Asio在应用层上对epoll返回的就绪队列做了一层封装，实现数据拷贝，从而完成了异步IO操作。下面结合boost.Asio
1.55源码，进行分析。

**分析：**

Boost.Asio中最重要的一个类是io\_service，io\_service抽象系统I/O接口,提供异步数据传输的能力，它是应用程序和系统I/O接口的桥梁。Boost.Asio主要采用它实现了Proactor模式。io\_service有一个重要的成员，io\_service\_impl，它在不同的系统下有不同的实现。在Windows下是基于IOCP的，在Linux下是基于task\_io\_service，这主要是通过预处理进行区分的：
<pre><code>// /asio/io_service.hpp
namespace detail {
#if defined(BOOST_ASIO_HAS_IOCP)
  typedef class win_iocp_io_service io_service_impl;
  class win_iocp_overlapped_ptr;
#else
  typedef class task_io_service io_service_impl;
#endif
  class service_registry;
} // namespace detail
</code></pre>

io\_service类如下：
<pre><code>// /asio/io_service.hpp
 class io_service: private noncopyable
{
private:
  typedef detail::io_service_impl impl_type;
  ...
  // The service registry.
  boost::asio::detail::service_registry* service_registry_;

  // The implementation.
  impl_type& impl_;
public:
   ...
   // Run the io_service object's event processing loop.
   BOOST_ASIO_DECL std::size_t run();
   // Run the io_service object's event processing loop to execute ready handlers.
   BOOST_ASIO_DECL std::size_t poll();
 };</code></pre>

 其中run()函数的具体实现如下：
<pre><code>{
  boost::system::error_code ec;
  std::size_t s = impl_.run(ec);
  boost::asio::detail::throw_error(ec);
  return s;
}</code></pre>

io\_service的run()函数最终是调用了impl\_(task\_io\_service)的run函数，对于poll也类似。

 下面来分析下task\_io\_service：

 task\_io\_service作为proactor模式在Linux下的具体实现，主要功能有两个：

 1. 对IO是否就绪的进行扫描

 2. 事件到达后对线程池的统一调度

task\_io\_service如下：
<pre><code>// /asio/detail/task_io_service.hpp
class task_io_service : public boost::asio::detail::service_base<task_io_service>
{
public:
   ...
   typedef task_io_service_operation operation;
  // Run the event loop until interrupted or no more work.
  BOOST_ASIO_DECL std::size_t run(boost::system::error_code& ec);
private:
   ...
  // The task to be run by this service.
  reactor* task_;
  // The count of unfinished work.
  atomic_count outstanding_work_;
  // The queue of handlers that are ready to be delivered.
  op_queue<operation> op_queue_;
} </code></pre>
 它有几个比较重要的成员：

1. reactor：这是一个typedef定义的同义词，它在不同平台有不同的实现：
<pre><code>// asio/detail/reactor_fwd.hpp
#if defined(BOOST_ASIO_WINDOWS_RUNTIME)
typedef class null_reactor reactor;
#elif defined(BOOST_ASIO_HAS_IOCP)
typedef class select_reactor reactor;
#elif defined(BOOST_ASIO_HAS_EPOLL)
typedef class epoll_reactor reactor;
#elif defined(BOOST_ASIO_HAS_KQUEUE)
typedef class kqueue_reactor reactor;
#elif defined(BOOST_ASIO_HAS_DEV_POLL)
typedef class dev_poll_reactor reactor;
#else
typedef class select_reactor reactor;
#endif</code></pre>

平台使用的reactor类型可以通过下面的方法得到：
<pre><code>#include <iostream>
#include <string>
#include <boost/asio.hpp>
int main()
{
std::string output;
#if defined(BOOST_ASIO_HAS_IOCP)
  output = "iocp" ;
#elif defined(BOOST_ASIO_HAS_EPOLL)
  output = "epoll" ;
#elif defined(BOOST_ASIO_HAS_KQUEUE)
  output = "kqueue" ;
#elif defined(BOOST_ASIO_HAS_DEV_POLL)
  output = "/dev/poll" ;
#else
  output = "select" ;
#endif
    std::cout << output << std::endl;
}</code></pre>

在我的环境中使用的是epoll。通常，Linux下主要采用epoll，对应的实现类是epoll\_reactor

2.
op\_queue\<operation\>：回调函数对象列表，这里面的每一个operation都会在调用了run函数的用户线程里面执行，每个操作选择一个空闲的线程，关于这点可以看下面的程序：
<pre><code>#include <boost/asio.hpp>   
#include <boost/thread.hpp>   
#include <iostream>   

void handler1(const boost::system::error_code &ec)   
{   
    std::cout <<  boost::this_thread::get_id() << " handler1." << std::endl;
    //sleep(3);
}   

void handler2(const boost::system::error_code &ec)   
{   
    std::cout <<  boost::this_thread::get_id() << " handler2." << std::endl; 
    sleep(3);
}   

boost::asio::io_service io_service;   

void run()   
{   
    io_service.run();   
}   

int main()
{   
    boost::asio::deadline_timer timer1(io_service, boost::posix_time::seconds(2));   
    timer1.async_wait(handler1);   
    boost::asio::deadline_timer timer2(io_service, boost::posix_time::seconds(2));   
    timer2.async_wait(handler2);   
    boost::thread thread1(run);   
    boost::thread thread2(run);   
    thread1.join();   
    thread2.join();   
}</code></pre>
        
通过使用定义在boost/thread.hpp中的boost::thread类，在main()中创建了两个线程。这两个线程为同一个I/O
service调用run()。这样做的好处是，一旦独立的异步操作完成，I/O
service可以有效利用两个线程来执行handler方法
上面的两个timer都是让时间停顿2s。由于有两个线程，handler1和handler2可以同时执行。如果timer2在停顿期间，timer1对应的handler1仍然在执行，那么handler2将会在第二个线程内执行。如果handler1已经结束了，那么I/O
service将会自由选择线程来执行handler2。这可以通过注释掉handler1中的sleep(3)来验证：不注释，handler1和handler2总是不同的线程中执行；注释后，handler1和handler2可能在同一线程中执行，也可能不再，这要取决于整个系统当时的线程调度情况。通过调整timer的时间也可以观察到同样的现象。

task\_io\_service的run函数最终调用的是reactor的run函数，在Linux下是epoll\_reactor的run函数，调用层次为：
<pre><code>// asio/detail/impl/Task_io_service.ipp

std::size_t task_io_service::run(boost::system::error_code& ec)
{
 ... 
    mutex::scoped_lock lock(mutex_); 
    std::size_t n = 0; 
    for (; do_run_one(lock, this_thread, ec); lock.lock()) 
    if (n != (std::numeric_limits<std::size_t>::max)())
        ++n; 
    return n;
}</code></pre>
<pre><code>
// asio/detail/impl/Task_io_service.ipp
std::size_t task_io_service::do_run_one(mutex::scoped_lock& lock,
    task_io_service::thread_info& this_thread,
    const boost::system::error_code& ec)
{
  while (!stopped_)
  {
    if (!op_queue_.empty())
    {
      // Prepare to execute first handler from queue.
      operation* o = op_queue_.front();
      op_queue_.pop();
      bool more_handlers = (!op_queue_.empty());

      if (o == &task_operation_)
      {
       ...
        task_->run(!more_handlers, this_thread.private_op_queue);
      }
      else
      {
     ...

        // Complete the operation. May throw an exception. Deletes the object.
        o->complete(*this, ec, task_result);
        return 1;
      }
    }
    else
    {
      // Nothing to run right now, so just wait for work to do.
      this_thread.next = first_idle_thread_;
      first_idle_thread_ = &this_thread;
      this_thread.wakeup_event->clear(lock);
      this_thread.wakeup_event->wait(lock);
    }
  }
  return 0;
}</code></pre>
epoll\_reactor的run()方法最终调用的是epoll的epoll\_wait的，通过epoll\_wait，将就绪的事件放入ops，等待处理。
<pre><code>//asio/detail/impl/epoll_reactor.ipp
void epoll_reactor::run(bool block, op_queue<operation>& ops)
{ 
    ... 
    // Block on the epoll descriptor. 
    epoll_event events[128]; 
    int num_events = epoll_wait(epoll_fd_, events, 128, timeout); 
    ...
   descriptor_state* descriptor_data = static_cast<descriptor_state*>(ptr); 
   descriptor_data->set_ready_events(events[i].events);  
   ops.push(descriptor_data);
}</code></pre>
epoll\_reactor类的定义如下：
<pre><code>
// detail/epoll_reactor.hpp
class epoll_reactor : public boost::asio::detail::service_base<epoll_reactor>
{
  ...
private:
  ...
  BOOST_ASIO_DECL static int do_epoll_create();
  ...
  // The epoll file descriptor.
  int epoll_fd_;
  // The io_service implementation used to post completions.
  io_service_impl& io_service_;
  ...
 
 };</code></pre>

整个调用过程如下图所示：

![](http://img.blog.csdn.net/20140424102929062)

Boost.Asio是这样使用epoll来进行事件分发的，实际的IO操作是如何和epoll联系起来的呢？继续...



以TCP为例，Boost.Asio中对TCP类的封装如下：
<pre><code>//asio/ip/Tcp.hpp
class tcp
{
public:
  /// The type of a TCP endpoint.
  typedef basic_endpoint<tcp> endpoint;

  /// Construct to represent the IPv4 TCP protocol.
  static tcp v4()
  {
    return tcp(BOOST_ASIO_OS_DEF(AF_INET));
  }
...
  /// The TCP socket type.
  typedef basic_stream_socket<tcp> socket;

  /// The TCP acceptor type.
  typedef basic_socket_acceptor<tcp> acceptor;

  /// The TCP resolver type.
  typedef basic_resolver<tcp> resolver;

#if !defined(BOOST_ASIO_NO_IOSTREAM)
  /// The TCP iostream type.
  typedef basic_socket_iostream<tcp> iostream;
#endif // !defined(BOOST_ASIO_NO_IOSTREAM)
...
private:
  // Construct with a specific family.
  explicit tcp(int protocol_family)
    : family_(protocol_family)
  {
  }

  int family_;
};</code></pre>
basic\_stream\_socket\<tcp\>类似于TCP中的socket，basic\_socket\_acceptor\<tcp\>用于监听套接字，它们均有许多异步方法，如async\_receive，async\_send等。basic\_stream\_socket和basic\_socket\_acceptor都是模板类，以basic\_stream\_socket为例，其async\_receive方法如下：
<pre><code>// asio/Basic_stream_socket.hpp
template <typename MutableBufferSequence, typename ReadHandler>
  BOOST_ASIO_INITFN_RESULT_TYPE(ReadHandler,
      void (boost::system::error_code, std::size_t))
  async_receive(const MutableBufferSequence& buffers,
      socket_base::message_flags flags,
      BOOST_ASIO_MOVE_ARG(ReadHandler) handler)
  {
    // If you get an error on the following line it means that your handler does
    // not meet the documented type requirements for a ReadHandler.
    BOOST_ASIO_READ_HANDLER_CHECK(ReadHandler, handler) type_check;

    return this->get_service().async_receive(this->get_implementation(),
        buffers, flags, BOOST_ASIO_MOVE_CAST(ReadHandler)(handler));
  }</code></pre>

get-\>service()的原型是什么呢？basic\_stream\_socket继承于basic\_socket\<Protocol,
stream\_socket\_service\>，而stream\_socket\_service类为：

<pre><code>// boost/asio/stream_socket_service.hpp
class stream_socket_service
#if defined(GENERATING_DOCUMENTATION)
  : public boost::asio::io_service::service
#else
  : public boost::asio::detail::service_base<stream_socket_service<Protocol> >
#endif
{
public:
#if defined(GENERATING_DOCUMENTATION)
  /// The unique service identifier.
  static boost::asio::io_service::id id;
#endif

  /// The protocol type.
  typedef Protocol protocol_type;

  /// The endpoint type.
  typedef typename Protocol::endpoint endpoint_type;

private:
  // The type of the platform-specific implementation.
#if defined(BOOST_ASIO_WINDOWS_RUNTIME)
  typedef detail::winrt_ssocket_service<Protocol> service_impl_type;
#elif defined(BOOST_ASIO_HAS_IOCP)
  typedef detail::win_iocp_socket_service<Protocol> service_impl_type;
#else
  typedef detail::reactive_socket_service<Protocol> service_impl_type;
#endif

}</code></pre>
从上面可以看出，Linux下，真正干活的类是reactive\_socket\_service，reactive\_socket\_service又继承自reactive\_socket\_service\_base，该类如下：
<pre><code>// boost/asio/detail/Reactive_socket_service_base.hpp
class reactive_socket_service_base
{
public:
  // The native type of a socket.
  typedef socket_type native_handle_type;

  // The implementation type of the socket.
  struct base_implementation_type
  {
    // The native socket representation.
    socket_type socket_;

    // The current state of the socket.
    socket_ops::state_type state_;

    // Per-descriptor data used by the reactor.
    reactor::per_descriptor_data reactor_data_;
  };
protected:
  // The selector that performs event demultiplexing for the service.
  reactor& reactor_;
}</code></pre>
基本的继承关系图如下：

![](http://img.blog.csdn.net/20140424101426140)

通过reactor，将epoll与事件处理联系起来了，在reactive\_socket\_service\_base中，async\_receive的实现为：
<pre><code>
// boost/asio/detail/impl/Reactive_socket_service_base.ipp 

// Start an asynchronous receive. The buffer for the data being received
  // must be valid for the lifetime of the asynchronous operation.
  template <typename MutableBufferSequence, typename Handler>
  void async_receive(base_implementation_type& impl,
      const MutableBufferSequence& buffers,
      socket_base::message_flags flags, Handler& handler)
  {
    bool is_continuation =
      boost_asio_handler_cont_helpers::is_continuation(handler);

    // Allocate and construct an operation to wrap the handler.
    typedef reactive_socket_recv_op<MutableBufferSequence, Handler> op;
    typename op::ptr p = { boost::asio::detail::addressof(handler),
      boost_asio_handler_alloc_helpers::allocate(
        sizeof(op), handler), 0 };
    p.p = new (p.v) op(impl.socket_, impl.state_, buffers, flags, handler);

    BOOST_ASIO_HANDLER_CREATION((p.p, "socket", &impl, "async_receive"));

    start_op(impl,
        (flags & socket_base::message_out_of_band)
          ? reactor::except_op : reactor::read_op,
        p.p, is_continuation,
        (flags & socket_base::message_out_of_band) == 0,
        ((impl.state_ & socket_ops::stream_oriented)
          && buffer_sequence_adapter<boost::asio::mutable_buffer,
            MutableBufferSequence>::all_empty(buffers)));
    p.v = p.p = 0;
  }</code></pre>

reactive\_socket\_recv\_op对象op封装用户回调函数，设置事件状态；start\_op调用epoll\_reactor的start\_op将read\_op操作注册到epoll的文件描述符中。在这两个过程中，op起到了桥梁作用，一方面通过epoll检查对应描述符事件是否就绪，另一方面在就绪后进行数据IO操作，并触发用户注册的回调函数。这样就完成了整个异步IO过程。

**结论：**

Boost.Asio通过对用户操作、回调函数、epoll的封装，完成了异步IO，从而实现了Proactor模式。

【参考】

1.http://www.cnblogs.com/zhiranok/archive/2011/10/07/boost-asio-proactor.html\

2.http://www.cnblogs.com/hello-leo/archive/2011/04/12/2013958.html\

