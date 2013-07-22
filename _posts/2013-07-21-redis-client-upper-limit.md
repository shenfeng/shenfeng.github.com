---
author: feng
date: '2013-7-21 08-21-00'
layout: post
status: publish
title: 设计和实现一个高性能的redis go client 之 实际性能上限
categories: ['performance', 'redis']
---

这几天有打算写一个redis的go的客户端：

1. redis是一个非常好用的k/v存储，广泛应用
2. 在负责的一个项目中，使用了redis做缓存，虽然redis本身很快，但client可能比较慢，看了几个开源的实现，发现有改进的空间
4. redis协议比较简单，实现一个客户端时间可控：可能一个周末可实现一个可用版本
5. 可以促进对redis的了解

我比较偏好把程序写得简单，高效。为了高效，需要知道一个天花板：最快可以做到多快，所以写了下面的go代码：

{% highlight go %}
func BenchmarkRawUpperLimit(b *testing.B) {
	c, err := net.Dial("tcp", "localhost:6379")
	if err != nil {
		b.Error(err)
	} else {
		buffer := make([]byte, 1024)
		ping := []byte("*1\r\n$4\r\nPING\r\n")
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			c.Write(ping)
			c.Read(buffer)  // receive PONG
		}
	}
	c.Close()
}
{% endhighlight %}


{% highlight bash %}
go test -bench .

BenchmarkRawUpperLimit	  200000	     13313 ns/op
{% endhighlight %}

发送ping，等待redis返回，需要13313ns，也就是每秒75k次 （测试机器为Intel(R) Core(TM) i7-2600 CPU @ 3.40GHz)

下面是相应的python版本, 作为一个简单的计较
{% highlight python %}
import socket, time

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("localhost", 6379))

buffer = bytearray(1024)
loop = 0
start = time.time()
LOOP = 1000000
while loop < LOOP:
    sock.send("*1\r\n$4\r\nPING\r\n")
    sock.recv_into(buffer)
    #  sock.recv(1024)  76k/s, a little slower than above, which is 78k per second
    loop += 1

t = time.time() - start
# about 78k per second on my computer
print t, 1 / (t / LOOP)
{% endhighlight %}

python版本的稍微快一点，一个可能的解释是，go语言的socket是在epoll等的基础上封装的，由runtime调度。但它与python的性能差别不大，可见go语言的runtime非常的高效
