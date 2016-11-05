title: "Linux内核-链表"
date: 2015-03-17 19:36:08
category: Linux
tags: [Linux,内核]
---

Linux内核中链表采用了和传统链表不一样的实现方式，通常我们在C编程中总是将数据结构嵌入到链表中，而内核中则是将链表嵌入到数据结构中，这种奇妙的实现方式体现了内核设计的精妙之处。
<!--more-->
传统教科书式的链表实现方式：
```c
struct node
{
     int a;
     int b;
     struct node * prev;
     struct node * next;
};
```
这种方式将链表指针直接嵌入到结构体中。相比之下，Linux内核中的链表实现方式独树一帜，下面是一个例子
```c
struct list_head
{
     struct list_head * prev;
     struct list_head * next;
};

struct node
{
     int a;
     int b;
     struct list_head list;
};
```
在这个例子中，我们可以看到，node结构体中嵌入了一个list_head指针结构体，正是这个list将一个个的node链接起来，形成了一种独特的链表结构，这种实现方式和传统的链表有很大不同。接下来的分析可以看到，其实采用这种方式的链表处理起来更为便捷，可以实现高效访问、插入、删除、合并等操作。

# 结构体成员的访问
看到上面的node结构体，有人会问，我只知道list指针，而它仅仅只是node结构体的成员之一，如何能访问node的其他成员变量？内核中关于这个问题，通过一组非常巧妙的宏方式实现：
```c
#define list_entry(ptr, type, member) \
        container_of(ptr, type, member)

#define container_of(ptr, type, member) ({                      \
         const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
         (type *)( (char *)__mptr - offsetof(type,member) );})
```
其中list_entry定义在"linux/list.h"中,container定义在"linux/kernel.h"，下面看看如何通过list指针访问node结构体成员变量。看代码：
```c
list_head* p;
struct node* node1;
node1=list_entry(p, struct node, list);
```
这里list_entry进行宏展开就是：
```c
const typeof(((struct node*)0)->list)* __mptr = p;
```
进一步就是:
```c
const list_head* __mptr=(p);
struct node* ((char*)__mptr-offsetof(struct node, list));
```
(char\*)\__mptr将\__mptr从list_head\*强制转换成char类型指针，这里转换成char类型指针的原因在于如\__mptr为int类型指针，那么表达式中减去offset就会减去4个offset字节数，这个问题可和指针自加减联系起来，如int\* ptr是指向一个int数组的指针，那么ptr++，即ptr=ptr+1会向下一个元素移动sizeof(int)个字节，这里减去offset也和\__mptr的类型相关，由于char在所有系统中都占一个字节，因此转换为char类型指针，可以保证只减去1个offset偏移量，这都是很重要的细节。

offsetof宏获取list成员在node结构体中的偏移地址，用list的实际地址，减去list在node中的偏移地址，得到node的真实起始地址。最后计算(exp1,exp2)表达式值exp2，struct node*类型。由此就知node1=list_entry(p, struct node, list);其实就是根据p指针，获取p指针所在结构体的起始地址，也就是结构体的指针。然后就可以访问结构体成员变量了，比如node1->a, node1->b等。