##  86 Partition List

> https://leetcode.com/problems/partition-list/
>
> 题目描述：给出一个链表以及一个数x，将量表重新整理，使得小于x的元素在前；大于等于x的元素在后；
>
> 如：1->4->3->2->5->2    x = 3
>
> 返回：1->2->2->4->3->5



做题情况：没做出来！



> 遇到的一个坑

```
比如链表：
dummyhead -> 1 -> 2 -> 3 -> 666 -> 7 -> 8 -> 9 -> 10 -> null
		a	 b		  pre	cur  	

如果要将 666插入到 dummyhead -> 1之间的节点；
首先需要先将节点 666给截断，才可以再进行插入操作，否则会报OOM异常。

//截断节点
pre.next = cur.next;
//然后插入节点
cur.next = dummyhead.next;//这段的dummyhead代表的是b
dummyhead.next = cur;     //这段的dummyhead则代表的是dummyhead.next 将要指向的节点



```

方法一：

```
public ListNode partition(ListNode head, int x) {
        if(head==null)
            return null;
        ListNode dummy1=new ListNode(0);
        ListNode dummy2=new ListNode(0);
        ListNode curr1=dummy1;
        ListNode curr2=dummy2;
        while(head!=null){
            if(head.val<x){
                curr1.next=head;
                curr1=curr1.next;
            }else{
                curr2.next=head;
                curr2=curr2.next;
            }
            head=head.next;
        }
        curr2.next=null;//这句很重要！链表最后一个元素如果小于x的话，那么curr2.next
不为null
        curr1.next=dummy2.next;
        return dummy1.next;
    }
```



方法二：

分别创建两个子链表，prehead1表示存放小于x的节点，而prehead2表示存放大于x的节点，并且存放节点的相对顺序不变。另外，用cur1表示prehead1子链表的指针，进行节点添加；用cur2表示prehead2子链表的指针；最后，通过cur1.next = prehead2.next将两个子链表连接起来，最后需要将cur2.next = null，截断该末尾节点。

```
public ListNode partition(ListNode head, int x) {
        ListNode preHead1 = new ListNode(0), preHead2 = new ListNode(0);
        ListNode cur1 = preHead1, cur2 = preHead2;
        while (head != null) {
            if (head.val < x) {
                cur1.next = head;
                cur1 = cur1.next;
            } else {
                cur2.next = head;
                cur2 = cur2.next;
            }
            head = head.next;
        }
        cur1.next = preHead2.next;
        cur2.next = null;
        return preHead1.next;
    }
```





方法三：

