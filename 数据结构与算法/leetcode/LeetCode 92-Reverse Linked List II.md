> # LeetCode 92. Reverse Linked List II



Reverse a linked list from position *m* to *n*. Do it in one-pass.

**Note: **1 ≤ *m* ≤ *n* ≤ length of list.

**Example:**

```
Input: 1->2->3->4->5->NULL, m = 2, n = 4
Output: 1->4->3->2->5->NULL
```

方法一：

思路：

将[m,n]的子链表进行反转，然后连接回源链表。

dummyhead -> 1----------NULL <- 2 <- 3 <- 4------ 5 -> NULL
----------------lhead--------pre----cur-next	
...
dummyhead -> 1----------NULL <- 2 <- 3 <- 4-------5 -> NULL
----------------lhead---------------------------pre-----cur---next
then：
dummyhead -> 1----------NULL <- 2 <- 3 <- 4-------5 -> NULL
----------------lhead----------------pre-------ppp-----cur----next

dummyhead -> 1----------NULL <- 5 <- 2 <- 3 <- 4
dummyhead -> 1 -> 4 -> 3 -> 2 -> 5 -> NULL

pre.next = cur;
lhead.next = ppp;

```
		ListNode dummyhead = new ListNode(0);
        dummyhead.next = head;
        ListNode cur = dummyhead;
        ListNode lhead = null;
        for(int i = 0; i < m; i++) {
            lhead = cur;
            cur = cur.next;
        }
        ListNode pre = null;
        ListNode next = null;
        for(int j = m; j <= n; j++) {
            next = cur.next;
            cur.next = pre;
            pre = cur;
            cur = next;
        }
        ListNode ppp = pre;
        while(pre.next != null) {
            pre = pre.next;
        }
        if(pre.next == null) {
            pre.next = cur;
        }
        lhead.next = ppp;
        return dummyhead.next;
```





方法二：

使用5个指针，来围绕[m,n]之间的节点交换。

```
dummyhead -> 1 ->[ 2 -> 3 -> 4 ->] 5 -> NULL
            pre   cur  next last        
            	 phead                      
dummyhead -> 1 ->[ 3 -> 2 -> 4 ->] 5 -> NULL
            pre        cur  next  last     
                 phead    
dummyhead -> 1 ->[ 4 -> 3 -> 2 ->] 5 -> NULL
```



```
if(head == null) {
    return head;
}
ListNode dummyhead = new ListNode(0);
dummyhead.next = head;
ListNode pre = dummyhead;
for(int i = 1; i < m; i++) {
    pre = pre.next;
}
ListNode cur = pre.next;
ListNode next = cur.next;
ListNode phead = pre.next;
for(int j = 0; j < n - m; j++) {
    ListNode last = next.next;
    phead = pre.next;

    cur.next = last;
    next.next = phead;
    pre.next = next;
    next = last;
}
return dummyhead.next;
```