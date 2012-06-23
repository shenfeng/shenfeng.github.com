---
author: feng
date: '2012-2-01 20-21-00'
layout: post
status: publish
desc: Online English dictionary written in c with epoll, it's super fast, 1600k concurrent connection, 57.3k req/s
title: How far epoll can push concurrent socket connection
categories:
- performance
---

I wrote an [online dictionary](http://shenfeng.me:9090/) in pure C in
Spring festival.

The dictionary data(about `8.2M`, file `dbdata`) is compressed and loaded into
memory using `mmap`, an index is build on top of it for fast lookup
using `binary search`. The RES is about `9M` when concurrent connection
is not high, say, blow 1k.

I handcrafted the web server in pure C with epoll. It serves static
file and word lookup request. The performance is amazing,
`57.3k req/s, when 1600k socket connections are kept`

## Test Machine

Server and test app are run on the same computer.
* `Mem`: 16G
* `CPU`: Intel(R) Core(TM) i7-2600 CPU @ 3.40GHz
* `OS` : GUN/Linux Debian 3.1.0-1-amd64

### Several config for Linux:

{% highlight sh %}

# set up virtual network interface,
# test client bind to these IP, then connect
for i in `seq 21 87`; do sudo ifconfig eth0:$i 192.168.1.$i up ; done

# more ports for testing
sudo sysctl -w net.ipv4.ip_local_port_range="1025 65535"
# tcp read buffer, min, default, maximum
sudo sysctl -w net.ipv4.tcp_rmem="4096 4096 16777216" 
# tcp write buffer, min, default, maximum
sudo sysctl -w net.ipv4.tcp_wmem="4096 4096 16777216" 
echo 9999999 | sudo tee /proc/sys/fs/nr_open
echo 9999999 | sudo tee /proc/sys/fs/file-max

# edit /etc/security/limits.conf, add line
# * - nofile 9999999

{% endhighlight %}

### Command to show status
{% highlight sh %}

cat /proc/net/sockstat

{% endhighlight %}

## 1600K concurrent connection. C1600k.
Test code, written in JAVA

{% highlight java %}
public class MakeupIdelConnection {
    final static int STEPS = 10;
    final static int connectionPerIP = 50000;
    public static void main(String[] args) throws IOException {
        final Selector selector = Selector.open();
        InetSocketAddress locals[] = new InetSocketAddress[32];
        for (int i = 0; i < locals.length; i++) {
            locals[i] = new InetSocketAddress("192.168.1." + (21 + i), 9090);
        }
        long start = System.currentTimeMillis();
        int connected = 0;
        int currentConnectionPerIP = 0;
        while (true) {
            if (System.currentTimeMillis() - start > 1000 * 60 * 10) {
                break;
            }
            for (int i = 0; i < connectionPerIP / STEPS && currentConnectionPerIP < connectionPerIP; ++i, ++currentConnectionPerIP) {
                for (InetSocketAddress addr : locals) {
                    SocketChannel ch = SocketChannel.open();
                    ch.configureBlocking(false);
                    Socket s = ch.socket();
                    s.setReuseAddress(true);
                    ch.register(selector, SelectionKey.OP_CONNECT);
                    ch.connect(addr);
                }
            }

            int select = selector.select(1000 * 10); // 10s
            if (select > 0) {
                System.out.println("select return: " + select + " events ; current connection per ip: " + currentConnectionPerIP);
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectedKeys.iterator();

                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    if (key.isConnectable()) {
                        SocketChannel ch = (SocketChannel) key.channel();
                        if (ch.finishConnect()) {
                            ++connected;
                            if (connected % (connectionPerIP * locals.length / 10) == 0) {
                                System.out.println("connected: " + connected);
                            }
                            key.interestOps(SelectionKey.OP_READ);
                        }
                    }
                }
                selectedKeys.clear();
            }
        }
    }
}
{% endhighlight %}

## 57.3k req/s

When 1600K connections are kept.

{% highlight java %}
class SelectAttachment {
    private static final Random r = new Random();
    private static final String[] urls = { "/d/aarp", "/d/about", "/d/zoo", "/d/throw", "/d/new", "/tmpls.js", "/mustache.js" };
    String uri;
    ByteBuffer request;
    int response_length = -1;
    int response_cnt = -1;
    public SelectAttachment(String uri) {
        this.uri = uri;
        request = ByteBuffer.wrap(("GET " + uri + " HTTP/1.1\r\n\r\n").getBytes());
    }
    public static SelectAttachment next() {
        return new SelectAttachment(urls[r.nextInt(urls.length)]);
    }
}

public class PerformanceBench {
    static final byte CR = 13;
    static final byte LF = 10;
    static final String CL = "content-length: ";

    public static String readLine(ByteBuffer buffer) {
        StringBuilder sb = new StringBuilder(64);
        char b;
        loop: for (;;) {
            b = (char) buffer.get();
            switch (b) {
            case CR:
                if (buffer.get() == LF)
                    break loop;
                break;
            case LF:
                break loop;
            }
            sb.append(b);
        }
        return sb.toString();
    }

    public static void main(String[] args) throws IOException {
        int concurrency = 1024 * 3;
        long totalByteReceive = 0;
        int total = 200000;
        int remaining = total;
        InetSocketAddress addr = new InetSocketAddress("127.0.0.1", 9090);
        ByteBuffer readBuffer = ByteBuffer.allocateDirect(1024 * 64);
        Selector selector = Selector.open();
        SelectAttachment att;
        SocketChannel ch;
        long start = System.currentTimeMillis();
        for (int i = 0; i < concurrency; ++i) {
            ch = SocketChannel.open();
            ch.socket().setReuseAddress(true);
            ch.configureBlocking(false);
            ch.register(selector, SelectionKey.OP_CONNECT, SelectAttachment.next());
            ch.connect(addr);
        }
        loop: while (true) {
            int select = selector.select();
            if (select > 0) {
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectedKeys.iterator();
                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    if (key.isConnectable()) {
                        ch = (SocketChannel) key.channel();
                        if (ch.finishConnect()) {
                            key.interestOps(SelectionKey.OP_WRITE);
                        }
                    } else if (key.isWritable()) {
                        ch = (SocketChannel) key.channel();
                        att = (SelectAttachment) key.attachment();
                        ByteBuffer buffer = att.request;
                        ch.write(buffer);
                        if (!buffer.hasRemaining()) {
                            key.interestOps(SelectionKey.OP_READ);
                        }
                    } else if (key.isReadable()) {
                        ch = (SocketChannel) key.channel();
                        att = (SelectAttachment) key.attachment();
                        readBuffer.clear();
                        int read = ch.read(readBuffer);
                        totalByteReceive += read;
                        if (att.response_length == -1) {
                            readBuffer.flip();
                            String line = readLine(readBuffer);
                            while (line.length() > 0) {
                                line = line.toLowerCase();
                                if (line.startsWith(CL)) {
                                    String length = line.substring(CL.length());
                                    att.response_length = Integer.valueOf(length);
                                    att.response_cnt = att.response_length;
                                }
                                line = readLine(readBuffer);
                            }
                            att.response_cnt -= readBuffer.remaining();
                        } else {
                            att.response_cnt -= read;
                        }
                        if (att.response_cnt == 0) {
                            remaining--;
                            if (remaining > 0) {
                                if (remaining % (total / 10) == 0) {
                                    System.out.println("remaining\t" + remaining);
                                }
                                key.attach(SelectAttachment.next());
                                key.interestOps(SelectionKey.OP_WRITE);
                            } else {
                                break loop;
                            }
                        }
                    }
                }
                selectedKeys.clear();
            }
        }
        long time = (System.currentTimeMillis() - start);
        long receiveM = totalByteReceive / 1024 / 1024;
        double reqs = (double) total / time * 1000;
        double ms = (double) receiveM / time * 1000;
        System.out.printf("total time: %dms; %.2f req/s; receive: %dM data; %.2f M/s\n", time, reqs, receiveM, ms);
    }
}
{% endhighlight %}


## Source code
It's on github,
[https://github.com/shenfeng/dictionary](https://github.com/shenfeng/dictionary)

 1. `/server` Server side code, in C.
 2. `/client` Javascript/HTML/CSS
 3. `/test/java`  Unit test and performance test code
 4. `/src`  Clojure and java code to generate the `dbdata` file
