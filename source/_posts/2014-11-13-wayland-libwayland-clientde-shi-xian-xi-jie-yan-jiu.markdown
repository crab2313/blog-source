---
layout: post
title: "Wayland - libwayland-client的实现细节研究"
---

# 前言
  本文是本人在研究`wayland`的C语言实现时所做的读书笔记。由于时间经历所限，仅仅完成的client部分的阅读。

# libwayland-client的工作原理
  `wayland`是一套用于替代`X`的图形显示系统。其中最重要的是wayland的协议，它用于服务端与客户端的交互。于是wayland的C语言实现其实就是wayland协议的实现。客户端的开发者可以通过libwayland-client这个库来实现与服务端通过wayland协议进行交互。
  libwayland-client的主要任务就是实现wire format。wire format是wayland协议的底层通信时的表示形态。
  wayland协议是一套基于对象的协议，即服务端给客户端提供了一些全局对象，客户端通过调用这些对象上的方法，收取这些对象发来的事件来实现客户端所期望的功能。通过一个unix domain socket，服务端和客户端可以直接发送一些基本的数据。值得注意的是wayland协议支持直接在客户端和服务端之间发送文件描述符（通过unix domain socket的ancillary data中的SCM_RIGHTS）。


# Wire Format
  服务端与客户端通过一个基于流的unix domain socket进行通信。通信所用的格式就称作wire format。这样的通信是基于消息的，客户端给服务端发送的消息叫做请求，服务端给客户端发送的消息称作事件。每个消息都由32位长的字组成，字节序使用当前主机的字节序。

## 消息头
  每个消息头有两个字长。第一个32位字是消息发送者的对象id。第二个32位字分成两个长度为16的部分。高16位是消息的长度，单位是字节。低16位是这个消息所代表的请求（事件）的操作码（opcode）。

## 参数类型
  每个请求的参数可以从wayland.xml中推出。


# 代码组织
  wayland的主要代码包含在src目录下。其中wayland-client.c和wayland-server.c分别包含客户端与服务端的代码，wayland-util.c包含了一些基本的数据结构实现和常用的工具函数。connection.c包含了wire format的底层实现，而event-loop.c实现了服务端需要用的事件循环。
  protocol/wayland.xml是wayland协议，写在一个xml文件中。向wayland中添加新协议是一件很简单的事，我们只需要写一个xml就行，剩下的工具代劳。简单来说，wayland-client.c仅仅提供了一些简单的建筑模块，剩下的代码可以通过工具来生成。
  wayland-client.c实现了一个代理，wl_proxy。每一wayland对象在客户端中都用一个wl_proxy来表示。实现了这样一个wl_proxy之后，只需要照着wayland.xml中的声明来生成对应的带就行了。

# 基本数据结构
  libwayland-client和libwayland-server共用了一些通用的数据结构。具体的定义和实现可以从wayland-util.c和wayland-util.h中找到。

## wl_list
```
struct wl_list {
	struct wl_list *prev;
	struct wl_list *next;
};
```
  wl_list是一个双向链表。值得注意的是，它是一个嵌入式的链表，这一点跟linux内核中的链表是一致的。即把链表嵌入到某一结构体中，从而把这个结构体变成一个链表。试想，我们将多个结构体中的链表串在一起，就跟普通的链表一样。但是，这个串起来的链表的每个节点都是另一个结构体中的成员。如果我们知道这个结构体的类型，那么就可以得出这个被嵌进去的链表节点相对于这个节点的偏移量，再加上知道这个节点的地址，就可以得到这个结构体的指针。因此，如果一个结构体中出现了`struct wl_list`，那么就分为两种情况。第一种是作为链表的表头，一般会取一个有意义的变量名，例如event_list。第二种就是这个节点中嵌入的链表节点，告诉我们这个结构体是在一个链表里面，这时变量名一般取link。

## wl_message
```
struct wl_message {
	const char *name;
	const char *signature;
	const struct wl_interface **types;
};
```
  wl_message表示一个消息，即请求或者事件。signature是这个消息的签名（原型），跟函数原型差不多，它使用单个字母表示参数的类型。比较需要注意的是字母o，它代表一个wayland对象。事实上，wayland对象需要使用接口进行区分，否则根本不知道这个对象的类型。故signature中每出现一个o，types中对应的wl_interface就是这个对象的接口。

## wl_interface
```
struct wl_interface {
	const char *name;
	int version;
	int method_count;
	const struct wl_message *methods;
	int event_count;
	const struct wl_message *events;
};
```
  wl_interface表示一个wayland对象的接口及实现。每个wayland对象都实现了某个接口，而这个接口就是这个wl_interface。其中包括了这个接口的方法与事件。

## wl_object
```
struct wl_object {
	const struct wl_interface *interface;
	const void *implementation;
	uint32_t id;
};
```
  wl_object是wayland对象的内部表示，很多其他类型的数据都是它的子类。事实上，每个wl_object都包含一个wl_interface，我们根据这个wl_interface来判断这个对象的类型。而implementation则存放某些私有数据。例如，wl_proxy就将listener放入implementation中。

# API
  所有操作都基于wl_proxy，也就是说，在客户端眼睛里，每个wayland对象都是一个wl_proxy。wayland-client.c只实现了wl_proxy和一个特殊的wl_proxy，wl_display，剩下的代码由wayland-scanner通过wayland.xml生成。对每个wayland对象，wayland-scanner都会声明一个结构体。例如对wl_registry这个wayland对象，scanner会声明一个struct wl_registry。但是，仅仅只是声明，其实并没有struct wl_registry的定义。与此同时，scanner根据wayland.xml中的定义来生成关于一个wayland对象的method和event。生成的代码在内部会将struct wl_registry类型转换成wl_proxy，通过wl_proxy将method发给服务端。对于event，scanner会生成一个类似于wl_registry_add_listener的函数，并生成一个类似于struct wl_registry_listener的架构体，其内部是wl_registry的事件处理函数。
