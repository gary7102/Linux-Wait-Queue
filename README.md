# Linux-Wait-Queue

# <font color = "orange">Intro</font>
<font size = 4>**2024 Fall NCU Linux OS Project 2**</font>

# <font color = "orange">Wait Queue</font>


**List head** Macro
```c
/*
 * Circular doubly linked list implementation.
 *
 * Some of the internal functions ("__xxx") are useful when
 * manipulating whole lists rather than single entries, as
 * sometimes we already know the next/prev entries and we can
 * generate better code by using them directly rather than
 * using the generic single-entry routines.
 */
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```
可以看到這是一個 Circular doubly linked list , 也就是我們要用來儲存wait queue的資料結構



**list_add_tail**
```c
/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
       // add     (new, tail-node , head)
}
```
`head` 是circular linked list的 head節點，通過 `head->prev` 獲取尾節點，並將新節點插入到尾節點和頭節點之間，也就是加入linked list最後面。


`__list_add`:  

```c
/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	if (!__list_add_valid(new, prev, next))
		return;

	next->prev = new;
	new->next = next;
	new->prev = prev;
	WRITE_ONCE(prev->next, new);
}
```
將 `new` 插入到 `prev` (tail)和 `next` (head)之間，分別更新 `prev->next` 和 `next->prev`

圖例:
```
# circularly linked list
before: head <-> A <-> B <-> tail

# 加入new node到最後面:
after : head <-> A <-> B <-> old tail <-> new tail
```

```

```

# additional Thoughts

想法:
每個加入wait queue 的flag 設為exclusive, 
這樣的話每次wake up 只會喚醒一個process? 

問題: 
這樣在user space printf 出來的還是會亂掉
