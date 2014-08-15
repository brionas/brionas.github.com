---
layout: post
title: wxSocket 实现分析和使用总结
description: wxSocket 实现分析和使用总结
category: wxSocket
tags: wxSocket
refer_author: Peichao
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

本编文章分析了wxSocket在Linux操作系统中的实现，并总结了相关使用方法。 

wxSocket 属于wxWidgets的wxNet子module。wxSocket是对系统socket
API的简单封装，对外提供了wxEvent通知机制，屏蔽了操作系统相关的实现细节。用户可以通过wxSocketClient/wxSocketServer来很方便的使用。

Note: 此处使用的wxWidgets 库的版本是
v2.8.9，主要涉及到代码文件 src/common/sckstrm.cpp， src/common/socket.cpp
和 src/unix/gsocket.cpp。

# 1. wxSocket 实现 

wxSocket相关类UML图：

![Inheritance
graph](http://docs.wxwidgets.org/3.0/classwx_socket_base__inherit__graph.png)

在wx的[在线manual](http://docs.wxwidgets.org/3.0/group__group__class__net.html)里，可以看到wxNet相关的所有类。

### 1.1 主要操作接口 

wxSocket主要的操作如下：

1. basic IO

    - Close
    - Discard
    - Peek
    - Unread
    - Read
    - ReadMsg
    - Write
    - WriteMsg

1. Socket state

    2. Functions to retrieve current state and miscellaneous info.

        - Error
        - GetLocal/GetPeer
        - IsData
        - IsDisconnected/IsConnected
        - LastCount
        - LastError
        - IsOk
        - SaveState/RestoreState

    2. Functions that perform a timed wait on a certain IO condition.

        - InterruptWait
        - Wait
        - WaitForLost
        - WaitForRead/WaitForWrite
        - and also:
            - wxSocketServer::WaitForAccept
            - wxSocketClient::WaitOnConnect

    2. Functions that allow applications to customize socket IO as needed.

        - GetFlags/SetFlags
        - SetTimeout
        - SetLocal

1. Handling socket events 

    - Notify/SetNotify
    - GetClientData/SetClientData
    - SetEventHandler


用户可以方便的通过SetNotify来订阅想要通知的事件。基于此，可以完全把Read/Write操作事件化。

### 1.2 wxSocketFlags
wxSocketFlags用来控制socket的运行模式，主要的flag如下：
<table><tr><td>
wxSocketFlags</td>
<td>
含义
</td></tr><tr><td>
wxSOCKET_BLOCK</td> 
<td>
Block the GUI (do not yield) while reading/writing data. 
</td></tr><tr><td>
wxSOCKET_NOWAIT 
</td><td>
Read/write as much data as possible and return immediately. 
</td></tr><tr><td>
wxSOCKET_WAITALL 
</td><td>
Wait for all required data to be read/written unless an error occurs.
</td></tr></table>

a. wxSOCKET\_BLOCK表示在调用wxSocketBase::Read/Write的时候，不会触发yield的操作。如果wxSocket本身在main thread里面进行读写操作，程序此时会不响应UI消息。而且由于yield操作本身可能会导致socket相关的操作重入（reentrant），这很可能不是用户代码想要的。所以一般推荐在wxSocket\_BLOCK+thread环境下使用wxSocket，这样socket的读写操作就不会阻塞UI thread。

b. wxSOCKET\_WAITALL与wxSOCKET\_NOWAIT正好相反，分别用于同步和异步读写。

### 1.3 wxSocket实现 
wxSocket 对 系统socket函数进行封装，内部的socket fd采用非阻塞模式。

**src/unix/gsocket.cpp**
{% highlight cpp  %}
m_fd = socket(m_peer->m_realfamily,m_stream?SOCK_STREAM:SOCK_DGRAM,0);
if(m_fd == INVALID_SOCKET)
{
	m_error = GSOCKIOERR;
	return GSOCK_IOERR;
}
#ifdef SO_NOSIGPIPE
	setsockopt(m_fd,SOL_SOCKET,SO_NOSIGPIPE,(const char*)&arg,sizeof(arg));
#endif
#if defined(__EMX__)||defined(__VISAGECPP__)
	ioctl(m_fd,FIONBIO,(char*)&arg,sizeof(arg));
#else
	ioctl(m_fd,FIONBIO,&arg);
#endif
{% endhighlight %}

这里arg的值为1，通过ioctl来设置socket fd为非阻塞。

# 2. wxSocketBase::Read/Write 
wxSocket本质上是non-blocking
socket。我们可以通过wxSocketBase::Read/Write直接读写数据。同时wxNet也提供了wxSocket
stream （wxSocketInputStream, wxSocketOuputStream）来读写数据。

通过wxSocket stream来读写数据和直接通过wxSocket来读写基本一样。但是wxSocket stream
的作用在于，我们可以利用wx库提供的通用的[stream操作类](http://docs.wxwidgets.org/3.0/group__group__class__streams.html)来处理socket相关的读写。比如：
利用wxBufferedInputStream 和 wxBufferedOutputStream来进行带缓冲区的读写。

### 2.1 未设置wxSOCKET\_WAITALL flag时的wxSocketBase::Read 

**问题：** 假如直接调用wxSocketBase::Read或者wxSocketInputStream::Read读取一大块数据（e.g.
10MB），可能会发生什么情况？

**答案：** wxSocketInputStream::Read返回后，实际读取的字节数很随机，跟当前的系统状态有很大关系。

当然这样的行为和直接用系统函数recv(2)或read(2)从non-blocking socket
读取数据类似。


 这个通过查看wxSocketBase::\_Read的实现(src/common/socket.cpp)，很容易弄清楚。

{% highlight cpp  %}
     wxUint32 wxSocketBase::_Read(void* buffer, wxUint32 nbytes)
    {
      // ...
      int ret;
      if (m_flags & wxSOCKET_NOWAIT)
      {
	    m_socket->SetNonBlocking(1);
	    ret = m_socket->Read((char *)buffer, nbytes);
	    m_socket->SetNonBlocking(0);
	    if (ret > 0)
      		total += ret;
      }
      else
      {
	    bool more = true;
	    while (more)
	    {
	      if ( !(m_flags & wxSOCKET_BLOCK) && !WaitForRead() )
		    break;
	      ret = m_socket->Read((char *)buffer, nbytes);
      	  if (ret > 0)
	      {
		    total  += ret;
		    nbytes -= ret;
		    buffer  = (char *)buffer + ret;
	      }
	      // If we got here and wxSOCKET_WAITALL is not set, we can leave
	      // now. Otherwise, wait until we recv all the data or until there
	      // is an error.
	      //
	      more = (ret > 0 && nbytes > 0 && (m_flags & wxSOCKET_WAITALL));
    	}
      }
      return total;
    }
{% endhighlight %}
_Read函数内的while循环的启动条件 要求`“``ret > 0 && nbytes > 0 && (m_flags & wxSOCKET_WAITALL)”`，这意味着要求满足如下条件：

1.  设置 wxSOCKET\_WAITALL
2.  上次读到数据
3.  还有数据需要读

即使设置了wxSOCKET\_WAITALL， wxSocketBase::\_Read也不能完全保证返回时，请求的nbytes已经完全读出，这是因为
代码行 “ret = m\_socket-\>Read((char \*)buffer, nbytes);”的返回值不能保证 “ret \> 0”条件。

 

### 2.2 辅助调试类 

这里通过继承wxSocketInputStream来打印更多调试信息。

{% highlight cpp  %}
    class wxGDSSocketInputStream : public wxSocketInputStream
	{
	public:
	    wxGDSSocketInputStream (wxSocketBase& s)
	        : wxSocketInputStream (s)
    	{ }
	protected:
	    size_t OnSysRead(void *buffer, size_t bufsize);
	    DECLARE_NO_COPY_CLASS(wxGDSSocketInputStream)
	};
     
    static void PrintSocketError(wxSocketBase* sock)
    {
    	if (sock->LastError() == wxSOCKET_WOULDBLOCK)
	    {
  		    LOG4TY_DEBUG("socket error: wxSOCKET_WOULDBLOCK");
  	    }
    	else if (sock->LastError() == wxSOCKET_TIMEDOUT)
    	{
    		LOG4TY_DEBUG("socket error: wxSOCKET_TIMEDOUT");
    	}
    	else
    	{
    		LOG4TY_DEBUG("socket error: " << sock->LastError());
    	} 
    }
     
    size_t wxGDSSocketInputStream::OnSysRead(void *buffer, size_t size)
    {
    	size_t count = wxSocketInputStream::OnSysRead(buffer, size);
    	wxLogTrace(GDS_STREAM_MASK, "[OnSysRead] request %lu, receive %lu", size, count);
    	if (m_i_socket->Error()) ::PrintSocketError(m_i_socket);
    	return count;
    }
{% endhighlight %}

通过调试发现，wxSocketBase::Read在读取大数据块时，很可能会提前返回，而且此时wxSocket内可能发生了错误（具体错误及原因见2.3节）。
 

### 2.3 GSocket::Read分析

wxSocketBase::Read最终通过GSocket::Read实现读数据（src/unix/gsocket.cpp）。

{% highlight cpp  %}
int GSocket::Read(char *buffer, int size)
{
  int ret;
  assert(this);
  if (m_fd == INVALID_SOCKET || m_server)
  {
    m_error = GSOCK_INVSOCK;
    return -1;
  }
  /* Disable events during query of socket status */
  Disable(GSOCK_INPUT);
  /* If the socket is blocking, wait for data (with a timeout) */
  if (Input_Timeout() == GSOCK_TIMEDOUT) {
    m_error = GSOCK_TIMEDOUT;
    /* Don't return here immediately, otherwise socket events would not be
     * re-enabled! */
    ret = -1;
  }
  else
  {
    /* Read the data */
    if (m_stream)
      ret = Recv_Stream(buffer, size);
    else
      ret = Recv_Dgram(buffer, size);
    /*
     * If recv returned zero for a TCP socket (if m_stream == NULL, it's an UDP
     * socket and empty datagrams are possible), then the connection has been
     * gracefully closed.
     *
     * Otherwise, recv has returned an error (-1), in which case we have lost
     * the socket only if errno does _not_ indicate that there may be more data
     * to read.
     */
    if ((ret == 0) && m_stream)
    {
      /* Make sure wxSOCKET_LOST event gets sent and shut down the socket */
      m_detected = GSOCK_LOST_FLAG;
      Detected_Read();
      return 0;
    }
    else if (ret == -1)
    {
      if ((errno == EWOULDBLOCK) || (errno == EAGAIN))
        m_error = GSOCK_WOULDBLOCK;
      else
        m_error = GSOCK_IOERR;
    }
  }
  /* Enable events again now that we are done processing */
  Enable(GSOCK_INPUT);
  return ret;
}
{% endhighlight %}

GSocket::Read主要调用了两个函数：Input\_Timeout和Recv\_Stream。

GSocket::Recv\_Stream 通过调用system 函数 **recv(2)** 实现。

{% highlight cpp  %}
int GSocket::Recv_Stream(char *buffer, int size)
{
  int ret;
  do
  {
    ret = recv(m_fd, buffer, size, GSOCKET_MSG_NOSIGNAL);
  }
  while (ret == -1 && errno == EINTR); /* Loop until not interrupted */
  return ret;
}
{% endhighlight %}

可以通过如下方式来查看socket 默认缓冲区大小：

{% highlight cpp  %}
pwang@p03bc ~$ cat /proc/sys/net/ipv4/tcp_rmem
4096 87380(85K 340B) 174760 //第一个表示最小值，第二个表示默认值，第三个表示最大值。
pwang@p03bc ~$ cat /proc/sys/net/ipv4/tcp_wmem
4096 16384(16k) 131072
{% endhighlight %}

这里read缓冲区最小4KB，最大170KB，默认约85KB。所以recv函数绝对不可能一次读10MB数据。

GSocket::Input\_Timeout函数（src/unix/gsocket.cpp）通过**select(2)**函数来计时。

{% highlight cpp  %}
GSocketError GSocket::Input_Timeout()
{
  struct timeval tv;
  fd_set readfds;
  int ret;
  /* Linux select() will overwrite the struct on return */
  tv.tv_sec  = (m_timeout / 1000);
  tv.tv_usec = (m_timeout % 1000) * 1000;
  if (!m_non_blocking) // m_non_blocking默认为false
  {
    wxFD_ZERO(&readfds);
    wxFD_SET(m_fd, &readfds);
    ret = select(m_fd + 1, &readfds, NULL, NULL, &tv);
    if (ret == 0)
    {
      GSocket_Debug(( "GSocket_Input_Timeout, select returned 0\n" ));
      m_error = GSOCK_TIMEDOUT;
      return GSOCK_TIMEDOUT;
    }
    if (ret == -1)
    {
      GSocket_Debug(( "GSocket_Input_Timeout, select returned -1\n" ));
      if (errno == EBADF) { GSocket_Debug(( "Invalid file descriptor\n" )); }
      if (errno == EINTR) { GSocket_Debug(( "A non blocked signal was caught\n" )); }
      if (errno == EINVAL) { GSocket_Debug(( "The highest number descriptor is negative\n" )); }
      if (errno == ENOMEM) { GSocket_Debug(( "Not enough memory\n" )); }
      m_error = GSOCK_TIMEDOUT;
      return GSOCK_TIMEDOUT;
    }
  }
  return GSOCK_NOERROR;
}
{% endhighlight %}

m\_timeout的默认值是600秒，所以如果没有数据可读，等到timeout错误返回，要等10分钟。

如果设置了wxSOCKET\_NOWAIT flag，GSocket::Input\_Timeout 就会直接返回。

### 2.4 wxSocketBase::Read/Write可能发生的错误 
wxSocket 所有可能发生的错误如下：
<table><tr><td>
error</td><td>               Note</td></tr>
<tr><td>wxSOCKET_NOERROR</td><td>   No error happened.</td></tr> 
<tr><td>wxSOCKET_INVOP </td><td> Invalid operation.</td></tr>
<tr><td>wxSOCKET_IOERR </td><td> Input/Output error.</td></tr>
<tr><td>wxSOCKET_INVADDR</td><td>Invalid address passed to wxSocket.</td></tr>
<tr><td>wxSOCKET_INVSOCK</td><td>Invalid socket (uninitialized).</td></tr>
<tr><td>wxSOCKET_NOHOST</td><td>No corresponding host.</td></tr>
<tr><td>wxSOCKET_INVPORT</td><td>Invalid port.</td></tr>
<tr><td>wxSOCKET_WOULDBLOCK</td><td>The socket is non-blocking and the operation would block.</td></tr>
<tr><td>wxSOCKET_TIMEDOUT</td><td>The timeout for this operation expired.</td></tr>
<tr><td>wxSOCKET_MEMERR</td><td>Memory exhausted.</td></tr>
 </table>

调用wxSocketBase::Read可能会触发哪些错误呢？

根据GSocket::Read的实现，如果socket连接没有正常，则可能有如下错误：
<table><tr><td>
error </td><td> Note</td></tr>

<tr><td>wxSOCKET_WOULDBLOCK</td><td>   没有数据可读或者当前socket不可写（buffer已满）。

   Recv_Stream函数发生错误，recv(2)返回-1，错误码是 EWOULDBLOCK 或 EAGAIN。</td></tr>

<tr><td>wxSOCKET_IOERR </td><td>    Recv_Stream函数发生错误，recv(2)返回-1。</td></tr>

<tr><td>wxSOCKET_TIMEDOUT </td><td>一般由GSocket::Input\_Timeout触发。

但并不见得就是真的timeout了，比如系统调用select出错，提前返回。</td></tr>
</table>

### 2.5 wxSOCKET\_TIMEDOUT
通过SocketInputStream辅助类，当调用wxSocketBase::Read来读大块数据时，很可能发生wxSOCKET\_TIMEDOUT错误。但是wxSocket的默认timeout是600秒，而Read函数明显没有等待那么长时间，为什么呢？

下面给出了一个程序发生timeout error时的堆栈：

{% highlight cpp  %}
Breakpoint 2, GSocket::Input_Timeout (this=0x1599430)
    at ./src/unix/gsocket.cpp:1563
1563 m_error = GSOCK_TIMEDOUT;
(gdb) p errno
$10 = 4
(gdb) bt 10
#0 GSocket::Input_Timeout (this=0x1599430) at ./src/unix/gsocket.cpp:1563
#1 0x0000002aa65c7739 in GSocket::Read (this=0x1599430,
    buffer=0x2ab4bec290 "", size=4117729) at ./src/unix/gsocket.cpp:1164
#2 0x0000002aa65c21c5 in wxSocketBase::_Read (this=0x1436e30,
    buffer=0x2ab4bec290, nbytes=4117729) at ./src/common/socket.cpp:363
#3 0x0000002aa65c2068 in wxSocketBase::Read (this=0x1436e30,
    buffer=0x2ab4bec290, nbytes=4117729) at ./src/common/socket.cpp:308
#4 0x0000002aa65c10b5 in wxSocketInputStream::OnSysRead (this=0x7fbfffa8c0,
    buffer=0x2ab4bec290, size=4117729) at ./src/common/sckstrm.cpp:90
#5 0x0000002aa3f08eb1 in wxGDSSocketInputStream::OnSysRead (
    this=0x7fbfffa8c0, buffer=0x2ab4bec290, size=4117729)
    at GUI/libComm/src/wxGDSStream.cpp:54
#6 0x0000002aa655b9b1 in wxInputStream::Read (this=0x7fbfffa8c0,
    buf=0x2ab4ae9010, size=4117729) at ./src/common/stream.cpp:846
{% endhighlight %}


根据上面列出的GSocket::Input\_Timeout函数的代码，通过调试发现
select函数返回值有时是-1， 此时发生系统错误EINTR
（慢系统调用被中断）。在这种情况下，Input\_Timeout函数会提前返回。

这说明wxSocketBase::Read来读大块数据时，有可能发生wxSOCKET\_TIMEDOUT错误，但是这个wxSOCKET\_TIMEDOUT
error并不一定是真实的，很可能是由EINTR引起的。

### 2.6 wxSocketBase::LastCount函数 

wxSocketBase::LastCount 会返回前一个Read/Write操作中成功的字节数。

根据wx manual，函数Discard(), Peek(), Read(), ReadMsg(), Unread(),
Write(), WriteMsg() 都有可能修改LastCount的值。

LastCount函数内通过一个变量来记录所有上述操作中成功的字节数，所以LastCount不是线程安全的。

因此，禁止对于同一个socket：

1. 一个线程调用Read，另一个线程调用Write
2. 两个线程同时Read或者Write


wx 3.0提供了接口**wxSocketBase::LastReadCount** 和 **wxSocketBase::LastWriteCount** 来解决这个问题。

# 3. 使用wxBufferedInputStream 

### 3.1 同时使用wxBufferedInputStream 和wxSOCKET\_WAITALL flag 

假设我们按照下面的方式初始化wxSocket：

**initialize wxSocket**
{% highlight cpp  %}
m_socket = new wxSocketClient();
m_is = new wxGDSSocketInputStream(*m_socket);
m_buf = new wxStreamBuffer(*m_is, wxStreamBuffer::read);
m_buf->SetBufferIO(1000000);
m_buf_is = new wxBufferedInputStream(*m_is, m_buf);
m_socket->SetFlags(wxSOCKET_BLOCK|wxSOCKET_WAITALL);
m_socket->SetNotify(wxSOCKET_LOST_FLAG);
m_socket->Notify(true);
{% endhighlight %}


然后利用wxBufferedInputStream 读数据，可能会发生什么？

**read through wxBufferedInputStream** 
{% highlight cpp  %}
char ptr[5];
memset(ptr, 0x00, 5);
m_buf_is->Read(ptr,4); 
{% endhighlight %}

答案是程序很可能会阻塞在m\_buf\_is-\>Read函数里。



下面是tachyon GUI按照上面的方式设置，GUI hang在那里后，打印的堆栈信息。

**call stack**
 
{% highlight bash  %}
 (gdb) bt
 #0 0x00000034cefbef86 in select () from /lib64/tls/libc.so.6
 #1 0x0000002aa65e7233 in GSocket::Input_Timeout (this=0x15ba580) at ./src/unix/gsocket.cpp:1548
 #2 0x0000002aa65e6739 in GSocket::Read (this=0x15ba580, buffer=0x14c727f "", size=994257) at ./src/unix/gsocket.cpp:1164
 #3 0x0000002aa65e11c5 in wxSocketBase::_Read (this=0x14c5b40, buffer=0x14c727f, nbytes=994257) at ./src/common/socket.cpp:363 //请求的字节数减少5473
 #4 0x0000002aa65e1068 in wxSocketBase::Read (this=0x14c5b40, buffer=0x14c5c10, nbytes=1000000) at ./src/common/socket.cpp:308
 #5 0x0000002aa65e00b5 in wxSocketInputStream::OnSysRead (this=0x14c5a80, buffer=0x14c5c10, size=1000000) at ./src/common/sckstrm.cpp:90
 #6 0x0000002aa3f28eb1 in wxGDSSocketInputStream::OnSysRead (this=0x14c5a80, buffer=0x14c5c10, size=1000000) at GUI/libComm/src/wxGDSStream.cpp:54
 #7 0x0000002aa65793a9 in wxStreamBuffer::FillBuffer (this=0x14c5ac0) at ./src/common/stream.cpp:204
 #8 0x0000002aa6579539 in wxStreamBuffer::GetDataLeft (this=0x14c5ac0) at ./src/common/stream.cpp:241
 #9 0x0000002aa6579a02 in wxStreamBuffer::Read (this=0x14c5ac0, buffer=0x409fed50, size=4) at ./src/common/stream.cpp:398
 #10 0x0000002aa657bde6 in wxBufferedInputStream::Read (this=0x14c1e70, buf=0x409fed50, size=4) at ./src/common/stream.cpp:1230 //请求4Bytes
{% endhighlight %}

根据调用栈可知，在frame 10的位置，我们调用了
**wxBufferedInputStream::Read (buf, 4)**，
但是此调用触发了wxStreamBuffer::FillBuffer操作。于是，**wxSocketInputStream::OnSysRead** 比较野蛮的要求从socket读整个buffer大小的内容。

实际上，socket端没有这么多数据，由于设置了 **wxSOCKET\_WAITALL**
flag，wxSocketBase::\_Read
函数会一直等待，直到读完要求的字节，或者产生timeout错误后返回。

### 3.2 安全使用socket Read 

可见，wxBufferedInputStream和wxSOCKET\_WAITALL不能同时使用，那我们的代码该怎么写呢？

可行的办法是在wxInputStream外封一个函数，就像下面的代码：
{% highlight cpp  %}
Uint32 wxGDSStream::Read( void *buffer, Uint32 size )
{
    wxLogTrace(GDS_STREAM_MASK, "[wxGDSStream::Read] request %u", size);
    m_stream_impl.Read(buffer,size);
    size_t read_size = m_stream_impl.LastRead();
    while( read_size < size)
    {
        m_stream_impl.Read((char*)buffer + read_size, size - read_size);
        read_size +=  m_stream_impl.LastRead();
        wxLogTrace(GDS_STREAM_MASK, "[wxGDSStream::Read] continue read %lu, totally receive %lu",
            m_stream_impl.LastRead(), read_size);
        LOG4TY_DEBUG("continue read:" << m_stream_impl.LastRead() << ", totally receive " << read_size);
        if (!m_sock->IsConnected()) break;
    }
    return read_size;
} 
{% endhighlight %}

此函数通过一个while循环来实现读取要求大小的数据，同时要考虑socket出错的情况。只要socket没有断开，我们就可以继续循环读取数据。

### 3.3 安全使用socket Write 
写操作也类似，但略有不同：
1.  对于写操作来说，当前协议要发送的数据的大小是已知的，读操作则一般是在读取的过程中才知道当前协议剩余的数据还有多少。
2.  我们调用Write的时候，一般都会把所有要发送的数据先按照协议组织好，然后再发送。这样就可以做到尽量少调用Write，进而少进行系统调用。

为了提高socket读写的效率，对于Read操作，我们一般倾向于使用带缓冲区的方式。对于Write操作，则不会使用缓冲区，尽量让数据尽早发送出去，在外部代码里面来控制尽量少调用Write。

在Write时不使用缓冲区，所以相对安全的Write操作有两种实现：

*1*. 利用wxSOCKET\_WAITALL flag.
{% highlight cpp  %}
int wxGDSSocket::Write( const void * buffer, Uint32 nbytes)
{   
    wxSocketFlags old_flag = m_socket->GetFlags();
    m_socket->SetFlags(old_flag | wxSOCKET_WAITALL);
    while (nbytes > 0)
    {
        int length = nbytes > m_max_write_length ? m_max_write_length : nbytes;
        m_socket->Write((char*)buffer + write_size, length);
        write_size += length;
        nbytes -= length;
    } // while
    m_socket->SetFlags(old_flag);
}{% endhighlight %}
 根据2.1节的结论，这个Write的实现其实也是有潜在问题的（wxSocketBase::Write函数返回值不能完全保证length长度的数据写成功），虽然可能很少发生
*2*. 不使用wxSOCKET\_WAITALL
{% highlight cpp  %}
int wxGDSSocket::Write( const void * buffer, Uint32 nbytes)
{
    //...
     int write_size = 0;
    while (nbytes > 0)
    {
        int length = nbytes > m_max_write_length ? m_max_write_length : nbytes;
        m_socket->Write((char*)buffer + write_size, length);
        length = m_socket->LastCount();
        write_size += length;
        nbytes -= length;
        if (m_socket->Error()) ::PrintSocketError(m_socket);
        if (!m_socket->IsConnected()) break;
    } // while
    return write_size;
} 
{% endhighlight %}


# 4. reference 

1.  [wx 3.0 wxSocketBase Class
    Reference](http://docs.wxwidgets.org/3.0/classwx_socket_base.html)
2.  [wx 2.8.9 源码](http://biolpc22.york.ac.uk/pub/2.8.9/)


