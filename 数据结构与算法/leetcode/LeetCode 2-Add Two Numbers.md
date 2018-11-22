## Add Two Numbers

https://leetcode.com/problems/add-two-numbers/

> Description：
>
> You are given two **non-empty** linked lists representing two non-negative integers. The digits are stored in **reverse order** and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.
>
> You may assume the two numbers do not contain any leading zero, except the number 0 itself.
>
> **Example:**
>
> ```
> Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
> Output: 7 -> 0 -> 8
> Explanation: 342 + 465 = 807.
> ```

做题情况：错误？对于错误的case，不知道为啥IDEA运行就没报错。

方法一：（自己的错误方法）

思路：

比较的笨，比较常规的方法，就是先从两列中取出节点值，分别放在i1和i2中，然后在将i1和i2的值进行相加，最后在处理相加的值，完成节点填充。



```
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode h1 = l1;
        ListNode h2 = l2;
        int n1 = 1, n2 = 1;
        int i1 = 0, i2 = 0;
        if(l1.val == 0 && l2.val == 0) {
            return new ListNode(0);
        }
        while(h1 != null) {
            int val = h1.val;
            i1 += val * n1;

            n1 = n1 * 10;
            h1 = h1.next;
        }

        while(h2 != null) {
            int val2 = h2.val;
            i2 += val2 * n2;

            n2 = n2 * 10;
            h2 = h2.next;
        }

        int result = i1 + i2;
        // 要判断整型溢出吗？
        // abc
        ListNode dummy = new ListNode(-1);
        ListNode cur = dummy;
        while(result > 0) {
            int k = result % 10;
            result = result / 10;
            ListNode node = new ListNode(k);
            cur.next = node;
            cur = cur.next;
        }
        return dummy.next;
}
```



方法二：

思路：对于两个链表，进行同时遍历，遍历过程中就对没个节点的值进行相加，然后在填充至返回的节点。





```
public ListNode addTwoNumbers2(ListNode l1, ListNode l2) {
        ListNode res = new ListNode (0);
        ListNode head = res;

        while(l1!=null || l2 != null){

            int value1 = (l1==null ? 0:l1.val);
            int value2 = (l2==null ? 0:l2.val);

            res.val+=value1+value2;

            if(l1!=null) {
                l1 = (l1.next==null? null:l1.next);
            }
            if(l2!=null) {
                l2 = (l2.next==null? null:l2.next);
            }

            boolean incremented = false;

            if(res.val>9 && !incremented){
                res.val %=10;
                res.next = new ListNode(0);
                res.next.val=1;
                res = res.next;
                incremented=true;
            }
            if((l1!=null || l2 !=null)&&(!incremented)) {
                res.next = new ListNode(0);
                res = res.next;
            }
        }
        return head;
}
```

