##  19. Remove Nth Node From End of List

自己做的结果：超时，使用的暴力循环。

> Given a linked list, remove the *n*-th node from the end of list and return its head.
>
> **Example:**
>
> Given linked list: 1->2->3->4->5, and n = 2.
>
> After removing the second node from the end, the linked list becomes 1->2->3->5.
>
> **Note:**
>
> Given *n* will always be valid.



使用两个指针分别为p指针和q指针，p指针指向要删除节点的前一个节点，用于进行删除操作。而q指针用于进行链表遍历与否的条件，关系由n确定，即q在p的n各个节点前面。

用图表示：

```
d -> 1 -> 2 -> 3 -> 4 -> 5 -> NULL       n = 2
p
 <-    n = 2 ->q              
```



总结：这种算法思想，就相当于圈出一个固定框，固定框的大小由n决定，然后框的移动终点就由框的末尾的判断方式决定，例如，定义变量p和q分别作为固定框的前和后，当q = null时，则p和q之间有两个节点的距离；如果是q.next = null，则p和q之间就有一个节点的距离。根据这同一思想的两个细节处理，有以下两个解决方法。

方法一：

```
public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dummyhead = new ListNode(0);
        dummyhead.next = head;
        ListNode p = dummyhead;
        p.next = head;
        ListNode q = p;
        for(int i = 0; i < n + 1; i++) {
            q = q.next;
        }
        while(q != null) {
            p = p.next;
            q = q.next;
        }
        p.next = p.next.next;
        return dummyhead.next;
 }
```



方法二：

```
public ListNode removeNthFromEnd2(ListNode head, int n) {
        if(head.next == null && n == 1){
            head = null;
            return head;
        }
        ListNode temp1  = head;
        ListNode temp2 = head;
        ListNode prev = head;
        for(int i=0;i<n;i++) {
            temp2 = temp2.next;
        }
        while(temp2 != null){
            prev = temp1;
            temp1 = temp1.next;
            temp2 = temp2.next;
        }
        if(temp1 == head) {
            head = head.next;
        } else {
            prev.next = temp1.next;
        }
        return head;
}
```

