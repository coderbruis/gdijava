## 328. Odd Even Linked List

> Given a singly linked list, group all odd nodes together followed by the even nodes. Please note here we are talking about the node number and not the value in the nodes.
>
> You should try to do it in place. The program should run in O(1) space complexity and O(nodes) time complexity.
>
> **Example 1:**
>
> ```
> Input: 1->2->3->4->5->NULL
> Output: 1->3->5->2->4->NULL
> ```
>
> **Example 2:**
>
> ```
> Input: 2->1->3->5->6->4->7->NULL
> Output: 2->3->6->7->1->5->4->NULL
> ```
>
> **Note:**
>
> - The relative order inside both the even and odd groups should remain as it was in the input.
> - The first node is considered odd, the second node even and so on ...



解题情况：独立完成，一次AC :），第一次把解题思路给屡清楚，然后一次敲完正确的代码，加油！

方法一：

> 思路

```
1.比较的是索引：所有偶数索引都在所有奇数索引的后面，并且要保持相对顺序不变。
2.odd 奇数，even 偶数

 1  ->  2  ->  3  ->  4  ->  5  ->  NULL
(1)     2     (3)     4     (5) 
 
 o      
        l      
				p		 ok				
        h
                
                
 1  ->  2  ->  3 ->  4  ->  5  ->  NULL
(1)    (3)     2     4     (5) 
 
        o
               h
                     ok     p
                     l              
                             
 1  ->  2  ->  3  ->  4 ->  5  ->  NULL
(1)    (3)    (5)     2     4     
               o      
					  h     
  									l
  									p 
```

每轮每个指针的变化情况：

```
1)

o: 1->2->3->4->5->null
l: 2->3->4->5->null
h: 2->3->4->5->null
p: 3->4->5->null

2)

o: 3->4->5->null
l: 4->5->null
h: 2->4->5->null
p: 5->null

3)
这里，隐含着head:1->3->5->null，而o指针为该子链表的尾结点。
o: 5->null
l: null
h: 2->4->null
p: null
```



代码：

```
public ListNode oddEvenList(ListNode head) {
        if(head == null || head.next == null || head.next == null) {
            return head;
        }
        ListNode o = head;
        ListNode h = o.next;
        ListNode l = o.next;
        ListNode p = h.next;
       //结束条件是根据p来判断的，如果p.next.next 为空，则说明已经处理好了
        while(p != null) {
            //将奇数几点连接起来
            o.next = p;
            //o指针向前挪一位
            o = p;

            ListNode ok = p.next;
            if(p.next != null) {
                p = p.next.next;
            } else {//p.next == null
                p = null;
            }

            l.next = ok;
            o.next = h;

            l = ok;
        }
        return head;
}
```

方法二：

> 分析

```
先定义另外的三个指针，然后让head指针代表odd奇数指针的头指针，让evenhead代表even偶数的头指针，然后让odd和even分别代表对应“奇数链表”和“偶数链表”的末尾。   
   
   1  ->  2  ->  3  ->  4  ->  5  ->  NULL
  [1]    [2]    [3]    [4]    [5]
    
  head
  odd
         even
        evenhead
 
1)、初始状态下，四个指针的详细：  
 
	head：1->2->3->4->5->NULL
 	 odd：1->2->3->4->5->NULL
	even：2->3->4->5->NULL
evenhead：2->3->4->5->NULL

分析好初始情况，然后就可以打断点，查看每个指针的变化情况。

2)、判断even当前节点，以及下一个节点不为null，判断条件为while(even != null && even.next != null)，则将odd.next.next的奇数节点进行“前移”，将2先给“移除”出去，odd.next = odd.next.next;然后让odd前移一位，odd = odd.next;
此时，各个节点的情况：

	head：1->3->4->5->NULL
 	 odd：3->4->5->NULL
	even：2->3->4->5->NULL
evenhead：2->3->4->5->NULL

那么，odd指针移动好之后，就该even了。同样的，将even.next.next的偶数节点进行“前移”，也就将3给“移除”出去，even.next = even.next.next;然后，同样让even前移一位，even = even.next;
此时，各个节点的情况：
	head：1->3->4->5->NULL
 	 odd：3->4->5->NULL
	even：4->5->NULL
evenhead：2->4->5->NULL

有没有发现，head和evenhead链表已经快达到了将奇数和偶数“聚集”的目的了？

3）、同样的，经过步骤（2），然后再分析一下各个指针对于链表的情况。
	head：1->3->5->NULL
 	 odd：5->NULL
	even：NULL
evenhead：2->4->NULL

经过步骤（3）之后，even为空了，while循环也就结束了，此时，head和evenhead已经排好了结果了，最后将head和evenhead连接起来就可以了。注意，这里odd作为head代表的“奇数链表”的末尾，用来连接两个链表最合适不过了。
odd.next = evenhead;
这样，通过了在一个链表中“原地”拆分成两个子链表的思想，就完美的解决了这个leetcode的问题了。
```

代码：

```
public ListNode oddEvenList(ListNode head) {
        if (head != null) {
            ListNode odd = head, even = head.next, evenHead = even;
            while (even != null && even.next != null) {
                odd.next = odd.next.next;
                even.next = even.next.next;
                odd = odd.next;
                even = even.next;
            }
            odd.next = evenHead;
        }
        return head;
}
```



