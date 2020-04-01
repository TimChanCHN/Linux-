<!--
 * @filename: filename
 * @Description: 
 * @Author: your name
 * @version: V00.00.00
 * @Date: 2020-03-31 10:15:51
 * @History: 修改历史记录列表，每条修改记录应包括修改日期、修改者、修改内容简述
 * @Copyright: 2020, BZL Robot
 -->
# 常见宏定义说明
```c
/* 1. 初始化链表头 */
void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}

/* 2. DMA_BIT_MASK ： 对底n个bit置1 */
#define DMA_BIT_MASK(n)	(((n) == 64) ? ~0ULL : ((1ULL<<(n))-1))

/* 3. container_of  通过member的地址计算得到对象type的指针 */
#define container_of(ptr, type, member) ({\
const typeof(((type *)0)->member) * __mptr = (ptr);	\			// typeof是获取member的类型
(type *)((char *)__mptr - offsetof(type, member)); })			// offsetof(type,member)--是获取member到type首地址的偏移量；ptr是一个指针（地址），通过减去一个偏移值，便可以获取type的真实地址


/* 4. list_entry ：container_of的再次封装，获取对象type的指针 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

/* 5. list_for_each_entry ： 遍历链表,调用该宏的话，可以利用pos指针对链表元素进行操作  */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_entry((head)->next, typeof(*pos), member);	\
	     &pos->member != (head); 	\											// 因为链表头为空，遍历结束的标志就是member为head
	     pos = list_entry(pos->member.next, typeof(*pos), member))


/* 6. klist_add_tail(a, b)	把结构体a放到链表b的后面 */
klist_add_tail




/* ERROR宏的意思 */
ENOMEM;                                         //内存异常，alloc失败后可以报该故障
EPROBE_DEFER                                    //porbe异常

```