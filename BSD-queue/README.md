# BSD-queue

本文主要介绍 OpenBSD 系统里自带的 queue 文件，里面实现了常用的单链表，双链表，队列等数据结构，并且只有一个 `queue.h` 文件，代码全部由宏实现。使用较广泛，NimBLE 协议栈里的单链表就是使用的 BSD-queue 。

BSD-queue 支持 5 种数据结构：单链表，双链表，尾队列，基于单链表的尾队列，环形队列。

**单链表**是其中最简单的数据结构，支持以下操作：

1. 在列表头部插入一个新的记录
2. 在任意列表元素后插入一个新的记录
3. 从列表头部移除一个记录
4. 正向遍历整个列表

其他的数据结构都支持上述操作，并增加了新的功能：

- **双链表**：在任意列表元素前插入一个新的记录
- **尾队列**：在列表的尾部插入一个新的记录，反向遍历列表
- **单链表的尾队列**：在列表的尾部插入一个新的记录
- **环形队列**：在列表的尾部插入一个新的记录，反向遍历列表

所有数据结构可用的功能可参考下述表格：

```c
/**
 *                      SLIST   LIST    STAILQ  TAILQ   CIRCLEQ
 * _HEAD                +       +       +       +       +
 * _HEAD_INITIALIZER    +       +       +       +       +
 * _ENTRY               +       +       +       +       +
 * _INIT                +       +       +       +       +
 * _EMPTY               +       +       +       +       +
 * _FIRST               +       +       +       +       +
 * _NEXT                +       +       +       +       +
 * _PREV                -       -       -       +       +
 * _LAST                -       -       +       +       +
 * _FOREACH             +       +       +       +       +
 * _FOREACH_REVERSE     -       -       -       +       +
 * _INSERT_HEAD         +       +       +       +       +
 * _INSERT_BEFORE       -       +       -       +       +
 * _INSERT_AFTER        +       +       +       +       +
 * _INSERT_TAIL         -       -       +       +       +
 * _REMOVE_HEAD         +       -       +       -       -
 * _REMOVE              +       +       +       +       +
*/
```

## 单链表

本小节以单链表为例，介绍 `SLIST_*` 宏该如何使用，学会了单链表，其他的数据结构也可以类似使用。



声明单链表头部：

```c
SLIST_HEAD(headname, type) head;
```

`SLIST_HEAD()` 宏用于申明单链表头节点，其内部包含指向链表里第一个元素的指针。`headname` 是该链表的名称，`type` 是挂载到链表上的结构体类型。声明完链表的头节点名称后，我们就可以定义头节点以及其指针：

```c
struct headname head, *headp;
```

若是不关心头节点名称，声明代码可省略成：

```c
SLIST_HEAD(, type) head, *headp;
```

示例代码如下：

```c
struct peer_chr {
    SLIST_ENTRY(peer_chr) next;
    ......
};
SLIST_HEAD(peer_chr_list, peer_chr);

struct peer_svc {
    struct peer_chr_list chrs;
    ......
};
```

上述代码声明了一个单链表类型，名称为 `peer_chr_list`，链表上挂载的结构体类型是 `peer_chr`，并且在 `peer_svc` 结构体里创建了一个该单链表。

初始化链表头部有两种方式，一种是 `SLIST_INIT(head)` ，另一种是静态初始化 `SLIST_HEAD_INITIALIZER(head)`，**注意在使用单链表时传入的都是头节点指针**。

`SLIST_ENTRY(type)` 宏用于定义链表里指向下一个元素的指针，`type` 为链表上挂载的结构体类型，具体使用可参考上述示例。



接下来就是一系列操作单链表的宏了，包括插入元素，删除元素，遍历单链表。

插入元素使用下述两个宏：

```c
SLIST_INSERT_HEAD(head, elm, field);
SLIST_INSERT_AFTER(slistelm, elm, field);
```

`SLIST_INSERT_HEAD()` 宏用于在链表头部插入新元素，`head` 是链表头节点，`elm` 是待插入元素，`field` 是 `SLIST_ENTRY()` 宏创建的连接对象，例如上述示例的 `next` 。

`SLIST_INSERT_AFTER()` 宏用于在链表中的一个元素后插入新元素，`slistelm` 代表链表中的元素，即将在该元素后插入新元素。其余的宏参数参考上述描述。

删除元素使用下述两个宏：

```c
SLIST_REMOVE_HEAD(head, field);
SLIST_REMOVE(head, elm, type, field);
```

`SLIST_REMOVE_HEAD()` 用于删除链表的第一个元素，`SLIST_REMOVE()` 用于在链表中删除指定元素。

单链表支持下述两种遍历方式：

```c
SLIST_FIRST(head);
SLIST_NEXT(elm, field);

for (np = SLIST_FIRST(&head); np != NULL; np = SLIST_NEXT(np, FIELDNAME))
    
SLIST_FOREACH(np, head, FIELDNAME)
```

还可以用 `SLIST_EMPTY(head)` 判断单链表是否为空。

例程如下：

```c
SLIST_HEAD(listhead, entry) head;
struct entry {
	...
	SLIST_ENTRY(entry) entries;	/* Simple list. */
	...
} *n1, *n2, *np;

SLIST_INIT(&head);			/* Initialize simple list. */

n1 = malloc(sizeof(struct entry));	/* Insert at the head. */
SLIST_INSERT_HEAD(&head, n1, entries);

n2 = malloc(sizeof(struct entry));	/* Insert after. */
SLIST_INSERT_AFTER(n1, n2, entries);

SLIST_FOREACH(np, &head, entries)	/* Forward traversal. */
	np-> ...

while (!SLIST_EMPTY(&head)) {	 	/* Delete. */
	n1 = SLIST_FIRST(&head);
	SLIST_REMOVE_HEAD(&head, entries);
	free(n1);
}
```



参考：

- OpenBSD queue(3) 手册：https://man.openbsd.org/queue.3
