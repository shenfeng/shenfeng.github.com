---
author: feng
date: '2014-11-23 08-21-00'
layout: post
status: publish
title:  生成Java JDBC访问代码，从SQL文件
---

### JDBC访问数据库

通过JDBC访问数据库，相信不少人都用过。比较辛苦，有很多的boilerplate。很多聪明的程序员也发现了这个问题，通过各种方式来解决这个问题（当然，他们也为了解决另外的问题， boilerplate是其中之一），比如Hibernate，iBATIS，JdbcTemplate(Spring)，等等。这几种，各有各的优势，也有很多用户。

最近在项目中，受Thrift，以及其它一些项目启发，试着写了一个程序，来自动生成JDBC的访问代码。

### SQL，简洁，表达力强

SQL，表达能力强，没有多少boilerplate:

{% highlight python %}
SELECT isbn, title, price, price * 0.06 AS sales_tax
FROM  Book WHERE price > 100.00 ORDER BY title limit 10;
{% endhighlight %}

### 程序：函数，参数，返回值

我们在写程序时，可能价格是个参数，还需要支持分页(limit, offset)，返回的是List<Book>, 于是就是:

{% highlight python %}

func list<Book> getBooksByPrice(float price, i32 limit, i32 offset) {
  SELECT isbn, title, price, price * 0.06 AS sales_tax
  FROM  Book WHERE price > :price ORDER BY title limit :limit, :offset;
}

{% endhighlight %}

这段话，表达是清楚的，并且没有额外的“客套话”。 如果有一个程序，输入是上面的code，生成的code是可以直接被调用的函数，将是理想情况。

### java-jdbc

这里有个问题，`Book` 未定义。当然，我们可以解析SQL语句，找到它来自 `Book` 表，继而contact数据库，找到Book表的metadata，能找到各个字段的数据类型, 就能定义Book了。如果这里面有Join的情况，会多联系几张表。
这样引入了联系数据库的依赖，增加依赖一般是需要再三考虑的。另外，解析SQL，找出字段的来源去脉，推断类型，不是一件简单的工作。

但有个折中的办法，引入一些冗余(为什么是冗余呢？因为这个Book定义的信息是可以推导出来的)

{% highlight python %}
struct Book {
    string isbn
    string title
    float price
    float salesTax
}
{% endhighlight %}

虽加入了一些冗余，貌似还可以接受。


我试着用Python写了这个程序 [java-jdbc](https://github.com/shenfeng/java-jdbc)，输入是上面的代码，输出是Java代码（函数）。

### 示例

运行命令
{% highlight python %}
python java_jdbc.py --input example.sf --out gen-java
{% endhighlight %}

就会在 gen-java目录生成访问数据库的代码。其中输入文件（example.sf）：

{% highlight python %}
namespace java me.shenfeng

struct Item {
    i32 id
    string name
}

// select
func Item getItemById(i32 id) {
    select * form item where id = :id
}
func list<Item> getItems(int limit, int offset) {
    select * from item limit :limit, :offset
}
func list<Item> getItems(list<i32> ids) {
    select * from item where id in (:ids) order by FIELD(id, :ids)
}

// insert, return generated primary key
func i32 saveItem(string name) {
    insert into item (name) value (:name)
}

// update
func void updateItem(i32 id, string newName) {
    update item set name = :newName where id = :id
}

// delete
func void deleteItemById(i32 id) {
    delete from item where id = :id
}

{% endhighlight %}


生成的API：

![2](imgs/jdbc/api.png)


### 相比其它方案的优势，缺点，以及以后的方向


相比Hibernate，iBATIS，JdbcTemplate(Spring)，java-jdbc优点：

1. 理解成本低，几乎是self-explained。
2. runtime零依赖，生成的code不依赖任何第三方库
3. 对join等支持良好，对各个数据库的“特殊” 特性，“原生”支持。
4. 维护方便。维护SQL，几乎是最简单的，SQL的文档也很多。
5. 实现简单 (300多行Python code)

缺点：

1. 有的同学不是很喜欢SQL
2. 功能单一，有些挺实用的功能，比如cache机制，读写分离机制，并没有支持

后面的方向嘛，由于这相当于定义了新的语言，可以通过扩展语言的方式，来扩展功能，比如

{% highlight java %}

// 最多cache 10000，通过lru策略淘汰，每个cache时间，最多3600s。 TODO
@lru(size=10000, expire=3600s)
func list<Book> getBooksByPrice(float price, i32 limit, i32 offset) {
  SELECT isbn, title, price, price * 0.06 AS sales_tax
  FROM  Book WHERE price > :price ORDER BY title limit :limit, :offset;
}

{% endhighlight %}
