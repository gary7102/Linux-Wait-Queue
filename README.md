# <font color = "orange">Intro</font>
<font size = 4>**2024 Fall NCU Linux OS Project 2**</font>
* Implement a custom wait queue-like functionality in kernel space. Allowing user applications to operate through the system call.
* Add a thread to the wait queue to wait. Remove threads from the wait queue, allowing them exit in **FIFO** order

**Link :**  [Project_2](https://github.com/gary7102/Linux-Wait-Queue/blob/main/Project_2.pdf) , [github](https://github.com/gary7102/Linux-Wait-Queue/tree/main)

<font size = 4>**Environment**</font>
```
OS: Ubuntu 22.04 on VMware workstation 17
ARCH: X86_64
Kernel Version: 5.15.137
```


# <font color = "orange">Wait Queue Implemention</font>

<font size = 4>**wait queue node**</font>
```c
typedef struct wait_queue_node {
    struct list_head list;
    struct task_struct *task;
} wait_queue_node_t;
```
* 每個node存表示 `my_wait_queue` 中的一個節點，每個節點對應一個process，裡面存了:
    * `list_head list`：使用 `list_head`(Circular doubly linked list）將所有等待的process接在一起。
    * `task_struct *task`: 每個process 指向自己的 process descriptor

<font size = 4>**global var.**</font>
```c
static LIST_HEAD(my_wait_queue); // self-define FIFO Queue
static spinlock_t queue_lock;    // protect list_add, list_del
```
* `my_wait_queue`： 使用 `LIST_HEAD` 定義了一個空的circularly doubly linked list，作為head node of wait queue。
* `queue_lock`：使用 `spinlock_t` struct 定義了一個spin lock，用於multi-threads對 `my_wait_queue` 的操作，防止race condition。

<font size = 4>**enter_wait_queue**</font>
```c
static int enter_wait_queue(void)
{
    wait_queue_node_t *node;

    // 分配新節點
    node = kmalloc(sizeof(wait_queue_node_t), GFP_KERNEL);
    if (!node)
        return 0;

    node->task = current;
    
    spin_lock(&queue_lock);
    ////////// Critical Region ///////////////////
    list_add_tail(&node->list, &my_wait_queue); // 加入 FIFO Queue
    printk(KERN_INFO "Thread ID %d added to wait queue\n", current->pid);
    ////////// Critical Region End ///////////////
    spin_unlock(&queue_lock);

    set_current_state(TASK_INTERRUPTIBLE); // 設定為可中斷的睡眠狀態
    schedule(); // 進入睡眠

    return 1; // 成功加入
}
```
* 使用 `spin_lock` 保護`list_add_tail`，同時間只會有一個node 加入 `my_wait_queue` ，確保不會出現race condition。
* 使用 `list_add_tail` 將節點加入queue尾部。
* 設置process state為 `TASK_INTERRUPTIBLE`，並使用 `schedule()` 將當前process 設為睡眠，直到呼叫 `wake_up_process` 才可以離開wait queue。
:::warning
記得 `schedule();` 要加在spinlock之外，  
否則在 critical region 中使用 `schedule();` 會使 process 占用 lock 不放，導致deadlock 發生
:::


<font size = 4>**clean_wait_queue**</font>
```c
static int clean_wait_queue(void)
{
    wait_queue_node_t *node, *temp;

    spin_lock(&queue_lock);
    ////////// Critical Region ///////////////////
    list_for_each_entry_safe(node, temp, &my_wait_queue, list) {
        list_del(&node->list); // 從 Queue 中移除

        printk(KERN_INFO "Thread ID %d removed from wait queue\n", node->task->pid);

        wake_up_process(node->task); // 喚醒進程
        kfree(node); // 釋放記憶體

        msleep(100); // sleep 100 ms
    }
    ////////// Critical Region End ///////////////
    spin_unlock(&queue_lock);

    return 1; // 成功清理
}
```
* 使用 `list_for_each_entry_safe` 遍歷wait queue
    * 使用 `list_del` 從queue中移除節點。
    * 使用 `wake_up_process` 喚醒process。
    * 釋放節點的記憶體。

:::success
這邊加入 `msleep(100)`   
因為在user space 中印出的順序受到multi-thread的影響可能會亂掉  
因此即使kernel space 的wait queue已經確保是FIFO的順序移除，但是在User space I/O 卻無法保證照順序印出，  
所以在`wake_up_process`之後加上睡眠100ms，讓user space  有時間可以印出資訊而不被多執行緒打亂 
:::


# <font color = "orange">Function explanation</font>

<font size = 4>**list_head**</font>
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
可以看到這是一個 Circular doubly linked list , 也就是我們要用來儲存wait queue的資料結構，  
使用 `LIST_HEAD` 就定義了一個空的雙向鏈表，作為等待Queue的head節點。


    
<font size = 4>**list_add_tail**</font>
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

// ...

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
       // = add   (new, tail-node , head)
}
```
* `head` 是circular linked list的 head節點
* 透過 `head->prev` 獲取尾節點，並將新節點插入到尾節點和頭節點之間，也就是加入linked list最後面。
* `__list_add` 中，其實就是將`new` 插入到`prev`和`next`當中，透過4個操作:
    * 更新 `next->prev = new` 和 `new->next = next`
    * 更新 `new->prev = prev` 和 `prev->next = new`

**圖例:**
```markdown
# circularly linked list
before add: next <-> A <-> B <-> C <->prev

# 加入new node到最後面:
after add : next <-> A <-> B <-> C <-> prev <-> new
```

<font size = 4>**list_del**</font>

```c
/*
 * Delete a list entry by making the prev/next entries point to each other.
 * This is only for internal list manipulation where we know
   the prev/next entries already!
 */
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	WRITE_ONCE(prev->next, next);
}


/*
 * check the validity of deletion
 * */
static inline void __list_del_entry(struct list_head *entry)
{
	if (!__list_del_entry_valid(entry))
		return;

	__list_del(entry->prev, entry->next);
}


/**
 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty() on entry does not return true after this, the entry is
 * in an undefined state.
 */
static inline void list_del(struct list_head *entry)
{
	__list_del_entry(entry);
	entry->next = LIST_POISON1;  // (void *) 0x00100100, invalid addr.
	entry->prev = LIST_POISON2;  // (void *) 0x00200200, invalid addr.
}
```
* 從雙向鏈表（`list_head`）中刪除指定的節點。
    * 透過 `next->prev = prev` 及 `prev->next = next` 達到delete node
* 使用 `LIST_POISON1` 及 `LIST_POISON2` 明確標記已刪除的節點，便於檢測誤用


**圖例:**
```markdown
# circularly linked list
before delete: head <-> A <-> prev <-> dnode <-> next

# Delete dnode:
after delete : head <-> A <-> prev <-> next
```


<font size = 4>**list_for_each_entry_safe**</font>
```c
/**
 * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_head within the struct.
 */
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_first_entry(head, typeof(*pos), member),	\
		n = list_next_entry(pos, member);			\
	     !list_entry_is_head(pos, head, member); 			\
	     pos = n, n = list_next_entry(n, member))
```
* `pos` 保存當前節點，`n` 保存下一個節點。
* `n` 確保當前節點(`pos`)刪除後，仍然能獲取下一個節點的地址。
* 因為下一個節點已經保存在 `n` 中，刪除 `pos` 不會影響 `n` 的正確性。



# Additional Thoughts

想法:
每個加入wait queue 的flag 設為exclusive, 
這樣的話每次wake up 只會喚醒一個process? 

問題: 
這樣在user space printf 出來的還是會亂掉

# Reference

* [Waitqueue in Linux](https://embetronicx.com/tutorials/linux/device-drivers/waitqueue-in-linux-device-driver-tutorial/)
*
* 
