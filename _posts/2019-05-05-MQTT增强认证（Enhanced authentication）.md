---
layout: splash
title:  "MQTT v5增强认证（Enhanced authentication）"
date:   2019-05-05 19:47:44 +0800
categories: MQTT
header:
  overlay_image: /assets/images/IMGP8176.jpg
excerpt: >
     
tags: 
  - MQTT
---
# 增强认证（Enhanced authentication）
MQTT `CONNECT`报文使用“用户名”和“密码”字段支持网络连接的基本身份验证。虽然这些字段以简单的密码身份验证命名，但它们可用于执行其他形式的身份验证，例如将令牌（token）作为密码传递。

增强认证扩展了此基本身份验证，包含了挑战/应答方式认证。身它可能涉及在`CONNECT`之后和`CONNACK`报文之前在客户端和服务器之间交换AUTH报文。它可能在`CONNECT`之后和`CONNACK`之前在客户端和服务器之间交换AUTH报文。

要开始增强认证，客户端在`CONNECT`报文中添加了认证方式（Authentication Method），这指定了要使用的身份验证方法。如果服务器不支持客户端提供的认证方式，它可以发送一个Reason Code为`0x8C`(不良身份验证方法)或0x87（未授权）的`CONNACK`报文，这种情况下必须关闭MQTT连接。

身份验证方法是客户端和服务器之间关于身份验证数据（Authentication Data）中发送的数据的含义以及`CONNECT`中的任何其他字段的协议，以及客户端和服务器完成身份验证所需的交换和处理。


*非规范评注*

    身份验证方法通常是SASL机制，使用这样的注册名称有助于交换。但是，身份验证方法不限于使用已注册的SASL机制。

如果客户端选择的身份验证方法指定客户端首先发送数据，则客户端应该在`CONNECT`报文中包含身份验证数据属性。此属性可用于提供身份验证方法指定的数据。认证数据的内容由认证方法定义。

如果服务器需要其他信息来完成身份验证，则可以向客户端发送AUTH报文。该报文必须包含的Reason Code为`0x18`（继续认证）。如果身份验证方法要求服务器将身份验证数据发送到客户端，则会在身份验证数据中发送。

客户端通过发送另一个AUTH报文来响应来自服务器的AUTH报文。该报文必须包含原因代码0x18（继续认证）。如果身份验证方法要求客户端为服务器发送身份验证数据，则会在身份验证数据中发送。

客户端和服务器根据需要交换AUTH报文，直到服务器通过发送Reason Code为`0`的`CONNACK`报文接受身份验证。如果接受身份验证需要将数据发送到客户端，则会在身份验证数据中发送。

客户端可以在此过程中的任何位置关闭连接。它可以在这之前发送`DISCONNECT`报文。服务器可以在此过程中的任何时候拒绝身份验证。它可以发送Reason Code为`0x80`或更高的`CONNACK`，并且必须关闭网络连接。

对于客户端和服务器，增强身份验证的实现是可选（OPTIONAL）的。如果客户端在`CONNECT`中不包含身份验证方法，则服务器不得发送`AUTH`报文，并且不得在`CONNACK`报文中发送身份验证方法。如果客户端在`CONNECT`中不包含身份验证方法，则客户端不得向服务器发送`AUTH`报文。

如果客户端在CONNECT报文中不包含身份验证方法，则服务器应该使用`CONNECT`报文，TLS会话和网络连接中的部分或全部信息进行身份验证。

*非规范性示例用于展示SCRAM挑战*

    * 客户端到服务器：CONNECT身份验证方法=“SCRAM-SHA-1”身份验证数据=客户端首次发送数据
    * 服务器到客户端：AUTH rc = 0x18身份验证方法=“SCRAM-SHA-1”身份验证数据=服务器首次发送数据
    * 客户端到服务器AUTH rc = 0x18身份验证方法=“SCRAM-SHA-1”身份验证数据=客户端最终数据
    * 服务器到客户端CONNACK rc = 0认证方法=“SCRAM-SHA-1”认证数据=服务器最终数据

*非规范评注用于展示Kerberos挑战*

    * 客户端到服务器CONNECT身份验证方法=“GS2-KRB5”
    * 服务器到客户端AUTH rc = 0x18身份验证方法=“GS2-KRB5”
    * 客户端到服务器AUTH rc = 0x18身份验证方法=“GS2-KRB5”身份验证数据=初始上下文令牌
    * 服务器到客户端AUTH rc = 0x18身份验证方法=“GS2-KRB5”身份验证数据=响应上下文令牌
    * 客户端到服务器AUTH rc = 0x18身份验证方法=“GS2-KRB5”
    * 服务器到客户端CONNACK rc = 0认证方法=“GS2-KRB5”认证数据=认证结果

## 重新认证（Re-authentication）

如果客户端在`CONNECT`报文中提供了身份验证方法，则可以在收到`CONNACK`后随时启动重新身份验证。它通过发送原因代码为`0x19`（重新认证）的`AUTH`报文来完成此操作。客户端务必将身份验证方法设置为与最初用于验证网络连接的身份验证方法相同的值。如果验证方法需要客户端首先发送数据，则此`AUTH`报文包含第一个验证数据作为验证数据。

服务器通过向客户端发送Reason Code为`0x00`（成功）的`AUTH`报文来响应此重新认证请求，以指示重新认证已完成，或发送Reason Code为`0x18`（继续认证）的`AUTH`报文以指示需要更多身份验证数据。客户端可以通过发送原因代码为`0x18`（继续验证）的`AUTH`报文来响应其他验证数据。此流程与原始身份验证一样继续，直到重新身份验证完成或重新身份验证失败。

如果重新认证失败，客户端或服务器应该发送带有适当Reason Code的`DISCONNECT`，并且必须关闭网络连接。

在此重新验证序列期间，客户端和服务器之间的其他报文流可以继续使用先前的身份验证。

*非规范评注*

    服务器可能会通过拒绝重新身份验证来限制客户端可以在重新身份验证中尝试变更的范围。例如，如果服务器不允许更改用户名，则可能会失败任何更改用户名的重新身份验证尝试。
