---
author: feng
date: '2013-11-15 08-21-00'
layout: post
status: publish
title: Apache Thrift的一另类用法 - dump/load数据文件
---

[Apache Thrift](http://thrift.apache.org/)
一般被用做跨语言的服务的开发。它在这方面很好用，高效且方便，我现在服务的美团大量的使用了它。

最近在做Deal的推荐系统，需要加载Deal的详细信息到内存。修改代码到程序跑起来的时间长短影响着开发效率，当然是越快越好，不希望每次修改代码后，编译，重启都需要去向数据库要一遍所有Deal的信息，毕竟C++的编译已经很耗时(*这是我喜欢go的一个原因，它编译迅速。但go对需要加载几百万用户的上亿行为到内存，20ms左右算出推荐结果的场景有些力不从心*)。一个可行的办法是把Deal信息dump到本地文件，重启时，快速的load这个文件。

Thrift的一个功能是把数据按照预定义的protocol，dump成与程序语言无关bytes，通过网络传输给另外的进程，对方以同样的protocol，load这些bytes，还原为原来的数据。通过网络这部分可以换成文件。

### 简单示例：定义数据格式，生成code

{% highlight c++ %}

// Thrift 定义文件 data.thrift
struct DealTiny {
    1: required i32 dealid,
    2: required i32 classid,
    3: required i32 mttypeid,
    4: required i32 bizacctid,
    5: required bool isonline,
    6: required i32 geocnt,
}

struct DealsTiny {
    1: required list<DealTiny> deals
}

{% endhighlight %}


通过下面的命令，生成需要的c++，和py code

{% highlight sh %}
# 生成py的code。python从数据库load数据，并保持为文件
thrift -gen py data.thrift

# 生成c++的code。线上服务是c++写的，它需要load py 生成的数据文件
thrift -gen cpp data.thrift
{% endhighlight %}

### dump 数据文件的python code 片段

{% highlight python %}
def dump_deals():
    deals = DealsTiny() 
    # 从db load数据

    # 用Thrift dump deals为bytes
    itransport = TTransport.TMemoryBuffer()
    prof = TBinaryProtocol.TBinaryProtocolAcceleratedFactory()
    ipro = prof.getProtocol(itransport)
    deals.write(ipro)

    # 写入文件
    buf = itransport.getvalue()
    with open("deals_info.bin", 'w') as f:
        f.write(buf)


{% endhighlight %}

### load 数据文件的C++ code 片段

{% highlight cpp %}
int load_deals(std::string file, DealsTiny &deals) {
    // mmap文件到内存
    int fd = open(file.c_str(), O_RDONLY);
    if (fd < 0) {
        perror(file.c_str());
        return fd;
    }
    const long size = get_file_size(file);
    unsigned char *buffer = (unsigned char*)mmap(NULL, size, PROT_READ, MAP_PRIVATE | MAP_POPULATE, fd, 0);
    close(fd);
    if (buffer == MAP_FAILED) {
        perror("mmap");
        return -1;
    }

    // 用Thrift load数据文件。
    shared_ptr<TTransport> itransport(new TMemoryBuffer(buffer, size));
    TBinaryProtocol ipro(itransport);
    deals.read(&ipro);

    munmap(buffer, size);
    return 1;
}


{% endhighlight %}

Thrift的BinaryProtocol很高效，用这种方式load数据文件方便，且快，很喜欢。
