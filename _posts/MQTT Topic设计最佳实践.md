# 前言
MQTT broker使用Topic来决定哪个Client收到哪条消息，值得注意的是SYS-topics，它是一个特殊的Topic，用于展示broker的状态信息。

# Topics
在MQTT中，Topic是一个UTF-8编码的字符串，broker使用它来为每一个连接的客户端筛选消息。Topic由一到多个Topic层级组成，每个Topic层级由斜杠（'/'）分隔开来。
![Aaron Swartz](https://www.hivemq.com/img/blog/topic_basics.png)

比起消息队列中的Topic，MQTT的Topic非常轻量级：客户端不需要预先创建Topic，直接Pub/Sub就可以了，broker接受任意合法的Topic而不需要事先创建它们。

需要注意的是：
* 每个Topic必须包含至少一个字符
* Topic中允许包含空格
* Topic是大小写敏感的
* 斜杠（'/'） 是一个合法的Topic

# 通配符
* 客户端可以用通配符来达到一次订阅多个Topic的目的；
* 通配符只能用于subscribe，不能用于publish；
* 通配符有两种：单层通配符（+）和多层通配符（#）

## 单层通配符：+
顾名思义，单层通配符替换一个Topic层级，加号在Topic中代表一个单层通配符
![Aaron Swartz](https://www.hivemq.com/img/blog/topic_wildcard_plus.png)

一个包含单层通配符的Topic，可以和使用任意字符串替换该通配符的Topic相匹配，
如_myhome/groundfloor/+/temperature和以下Topic匹配结果：

![Aaron Swartz](https://www.hivemq.com/img/blog/topic_wildcard_plus_example.png)


## 多层通配符
多层通配符包含多个Topic层级，这个哈希符号表示在Topic中多层通配，为了使broker能够判断匹配哪些Topic，#必须是Topic中的最后一个字符，并且它前面必须是斜杠（'/'）。
![Aaron Swartz](https://www.hivemq.com/img/blog/topic_wildcard_hash.png)
![Aaron Swartz](https://www.hivemq.com/img/blog/topic_wildcard_hash_example.png)

当客户端使用多层通配符订阅主题时，它会收到所有以通配符前边部分开头的Topic的消息，无论这个Topic多长，层级多深。如果订阅"#"这个Topic，那么这个客户端会收到所有发送到这个broker的消息。如果你的业务需要高吞吐量，那么就不应该订阅#这个Topic。

## 以$开头的Topic
通常，你可以任意命名你的Topic，但是有一个例外：以$开头的Topic有着不同的用途，当你订阅#这个Topic的时候，以$开头的Topic并不包含在内。$开头的Topic作为保留Topic仅供broker内部数据统计使用，客户端不可以向$开头的Topic发送消息。

# 总结
以上就是MQTT消息Topic的一些基本内容。可以看出，MQTT Topic提供了很大的灵活性，但在实际工作中使用通配符时，你将面临不少的挑战，以下是一些最佳实践：

* **不要在Topic最前面加上斜杠**
虽然可以这么做（例如，`/myhome/groundfloor/livingroom`），但这样就平白多了一层没有任何内容的Topic层级，没有带来任何好处，有时候还会引起歧义

* **不要在Topic中使用空格**
空格是每个程序员的天敌，当程序出现问题时候，空格使得调试格外的困难。同上一条一样，我们不能仅仅因为允许这样做就真的这样做：UTF-8编码包含许多种不同的空格，这些生僻的字符应当尽量避免。

* **让Topic尽量简短**
每个Topic都包含在使用它的消息体里面，当这些消息进入小设备时，每个字节都很重要，Topic的长度影响重大。

* **仅使用ASCII字符，避免不可打印的字符**
由于非ASCII的UTF-8字符通常无法正确显示，我们很难发现其中的拼写错误。除非相当有必要，否则应当尽量避免非ASCII字符出现在Topic里。

* **将唯一ID或ClientId加入Topic中**
将发送方的唯一ID嵌入Topic中非常有用，Topic中的唯一ID能够帮助我们识别谁发送了这条消息。这个内嵌的ID还能够用于认证：只有ClientId与Topic中ID一致的情况下才能够发送消息，例如_client1可以向_client1/status发送消息，但不能向_client2/status发送消息。

* **不要订阅#**
不用使用客户端订阅broker上的所有消息，客户端是没有能力处理这么多消息的（特别是当系统吞吐量很大的时候），如果需要获取broker上的所有消息（例如持久化存储），建议开发一个专门的broker插件来完成这项任务。

* **记住可扩展性**
Topic是一个很灵活的概念，无论何时，我们不应该预先分配它们。不过消息的发送者和接收者都应关注Topic。考虑如何扩展Topic来适应新功能或新产品是很重要的，例如向智能家庭解决方案中加入一组新的传感器，应当在不改变现有Topic层级的前提下，添加一组新的Topic。

* **使用专有的Topic，而不是通用的**
Topic的名称要尽量专有，例如，客厅中有三个传感器，我们应当分别为它们创建不同的Topic：_myhome/livingroom/temperature, _myhome/livingroom/brightness和_myhome/livingroom/humidity，而不是全部都发送到_myhome/livingroom。专有Topic能够让我们充分利用MQTT的一些特性，例如保留消息（retained message）。