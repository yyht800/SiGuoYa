# 算法笔记——模板代和记录

## 1. 数组链表

### 1.0 数组

数组（Array）是一种线性表数据结构。它用一组连续的内存空间，来存储一组具有相同类型的数据。

查找的时候可以通下标以及寻址公式算出内存地址。

数组本身也是一个对象，**数组可以持有值类型，而容器则不能**（容器指的是ArrayList这种，他就必须用到包装类），验证数组对象的方式。

[Java中数组的特性](https://blog.csdn.net/zhangjg_blog/article/details/16116613#t1), 所以数组可以作为泛型对象类似下面的使用方式.

```java
PriorityQueue<int[]> priorityQueue = new PriorityQueue<>
```

### 1.1 链表交换两个节点的值

链表可以引入哨兵节点，解决边界问题

```java
public ListNode swapPairs(ListNode head) {
        if(head==null||head.next==null) return head;
        ListNode next = head.next;
        head.next = swapPairs(next.next);
        next.next = head;
        return next;
    }
```

### 1.2 翻转链表

```java
public ListNode reverseList(ListNode head) {
        ListNode curr = head;
        ListNode pre = null;
        while(curr!=null){
            ListNode next = curr.next;
            curr.next = pre;
            pre= curr;
            curr = next;
        }

        return pre;
    }
```

### 1.4 跳表

![](https://static001.geekbang.org/resource/image/49/65/492206afe5e2fef9f683c7cff83afa65.jpg)

跳表会多一个down指针指向下一级结点。多级索引的状态如上图所示

## 1.3 Deqe 的API差异

由于first和last的功能相似，就不展开赘述了。

```java
void addFirst(E e);//将对象e插入到双端队列头部，容间不足时，抛出IllegalStateException异常；

boolean offerFirst(E e);//将对象e插入到双端队列头部

E removeFirst();//获取并移除队列第一个元素，队列为空，抛出NoSuchElementException异常；

E pollFirst();//获取并移除队列第一个元素，队列为空，返回null；

E getFirst();//获取队列第一个元素，但不移除，队列为空，抛出NoSuchElementException异常；

E peekFirst();//获取队列第一个元素，队列为空，返回null；
```

总结，这里的api，采用list相似的名字使用，会有抛出异常的风险，采用Stack风格类似的api不会抛出异常。

## 2. 堆栈

### 2.1 堆

堆是一个特殊的树，具有以下两个特点

* 堆是一个完全二叉树
* 堆中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值

### 2.2 Java 的 Stack 类不建议使用，why？

![image-20210125142609039](https://github.com/yyht800/SiGuoYa/tree/66aab4864445d871d843c967469c9144123175ef/Users/yyht800/Library/Application%20Support/typora-user-images/image-20210125142609039.png)

以上截图来自[https://docs.oracle.com/javase/8/docs/api/java/util/Stack.html](https://docs.oracle.com/javase/8/docs/api/java/util/Stack.html) 文档，因此，如果你想使用栈这种数据结构，Java 官方推荐的写法是这样的（假设容器中的类型是 Integer）：

```java
   Deque<Integer> stack = new ArrayDeque<Integer>();
```

Java 中的 Stack 类，最大的问题是，继承了 Vector 这个类（结构图可以参考上面截图）。先看一下下面的图：

![](https://pic3.zhimg.com/80/v2-76c3c04de2e8609c488fa0081fb99c26_1440w.png)

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable{}

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{}
```

从两个class的定义可以看出ArrayList和vector定义上是一样的，两者的区别是什么？

* Vector是线程安全的，ArrayList不是线程安全的。
* ArrayList在底层数组不够用时在原来的基础上扩展0.5倍，Vector是扩展1倍。

Vector 作为动态数组，是有能力在数组中的任何位置添加或者删除元素的。因此，Stack 继承了 Vector，Stack 也有这样的能力,所以Stack可以调用类似的方法

```java
Stack<Integer> stack = new Stack<>();
//在1，2的中间插入了一个数字12345
stack.add(1,12345)
```

这样的方式，明显破坏了栈设计的初衷，操作不当会带来意想不到的bug。下面可以看下推荐的Deque

### 2.3 Queue以及Deque和 Priority Queue

Queue是队列（接口），Deque继承自Queue也是一个接口，不同的是Deque是双端队列。

当使用队列功能时建议使用LikedList 当使用栈功能时建议使用ArrayDeque

优先级队列PriorityQueue——常常也被用作堆的实现,并且他实现了Queue 的接口，不允许放入null。

* 默认实现是小顶堆
* 初始化的容量是11
* 保证线程安全可以使用PriorityBlockingQueue 类，compare的方法中return的m-n是小顶，反之是大顶堆

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>(new Comparator<Integer>() {
            @Override
            public int compare(Integer m, Integer n) {
                return m-n;
            }
        });
```

## 3. N叉树

### 3.1 递归前中后序遍历

这个太简单不记录，主要是记录前中后表示父节点的位置

### 3.2 中序遍历，迭代 模板

```java
public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> res = new ArrayList<>();
        Deque<TreeNode> stack = new ArrayDeque<>();
        while (!stack.isEmpty() || root != null) {
            if (root != null) {
                stack.push(root);
                root = root.left;
            } else {
                root = stack.pop();
                res.add(root.val);
                root = root.right;
            }
        }
        return res;
    }
```

### 3.3 递归模板

```java
public void recur(int level, int param) { 
        // terminator  
    if (level > MAX_LEVEL) {    
                // process result  递归终止条件   
                return;   
        }  

        // process current logic 处理当前的循环层级的数据逻辑  
            process(level, param);

    // drill down  返回到下一层
    recur( level: level + 1, newParam);   

    // restore current status  回收当前的状态，如果有的话

}
```

### 3.4 回溯算法模板

```java
private static int divide_conquer(Problem problem){
        if (problem == NULL) {    
                int res = process_last_result(); 
        return res;       
    }  

    subProblems = split_problem(problem)    
    res0 = divide_conquer(subProblems[0])  
    res1 = divide_conquer(subProblems[1])    
    result = process_result(res0, res1);    
    return result;

}
```

### 3.5 B+树

B+树的特点：

* 每个节点中子节点的个数不能超过 m，也不能小于 m/2；
* 根节点的子节点个数可以不超过 m/2，这是一个例外；
* m 叉树只存储索引，并不真正存储数据，这个有点儿类似跳表；
* 通过链表将叶子节点串联在一起，这样可以方便按区间查找；一般情况，根节点会被存储在内存中，其他节点存储在磁盘中。

除此之外还有B树，B 树只是一个每个节点的子节点个数不能小于 m/2 的 m 叉树。

### 3.5 AVL和红黑树

现补充一下，二叉搜索树，常见的二叉树，不是平衡二叉树。

AVL树（平衡二叉树）

* 平衡因子

  某个节点左子树的高度减去右子树的高度，值为{-1,0,1};

* 找回平衡（找到最小失衡树）
  * 左旋
  * 右旋
  * 左右旋
  * 右左旋
  * 动画效果展示[https://www.cs.usfca.edu/~galles/visualization/AVLtree.html](https://www.cs.usfca.edu/~galles/visualization/AVLtree.html)
* 不足

  节点需要存储额外的信息，操作比较频繁

红黑树（近似平衡二叉树）

他能确保任意一个节点的左右子树的高度小于两倍，具体特征：

* 每个节点要么是红色，要么是黑色
* 根节点是黑色
* 每个叶节点（nil节点、空节点）是黑色
* 不能有两个相邻的红色节点
* 任意节点到其每个叶子节点的所有路劲都包含相同数量的黑色节点

两个的比较：

1. AVL查询比红黑树更快（因为AVL树是严格的平衡树）
2. 添加和删除红黑树更快
3. AVL需要更多的内存来存储信息（需要存储平衡因子）
4. 查询操作多的情况下适合AVL，如果读写频繁的可以选择红黑树

## 4. 动态规划

### 4.1 股票问题

[股票问题系列通解](https://leetcode-cn.com/circle/article/qiAgHn/)

### 4.2 分治代码模板

```java
            private static int divide_conquer (Problem problem, ){
            if (problem == NULL) {
                int res = process_last_result();
                return res;
            }
            subProblems = split_problem(problem);

            res0 = divide_conquer(subProblems[0]);
            res1 = divide_conquer(subProblems[1]);

            result = process_result(res0, res1);
            return result;
        }
```

### 并查集代码模板

岛屿数量或者染色问题

```java
public class UnionFind {

    private int count = 0; 
    private int[] parent; 

    public UnionFind(int n) { 
        count = n;
        parent = new int[n]; 
        for (int i = 0; i < n; i++) { 
            parent[i] = i;
        }
    } 

    public int find(int p) { 
        while (p != parent[p]) {
             parent[p] = parent[parent[p]]; p = parent[p];
        }
        return p; 
    }

    public void union(int p, int q) {
        int rootP = find(p); 
        int rootQ = find(q); 
        if (rootP == rootQ) return; 
        parent[rootP] = rootQ; 
        count--;
    }

}
```

### 位运算

1. 计算机内部只有高电位和低电位就是010101，所以计算机使用的事二进制。
2. 左移 &lt;&lt;

   0011 ==&gt; 0110 将数字左移，不足位数补0；

3. 右移 &gt;&gt;

   0011 ==&gt; 0110 将数字左移，不足位数补0；

4. \| 或

   0011\|1011 ==&gt;1011 按位进行操作

5. & 与

   0011&1011 ==&gt;0011 按位进行&操作

6. ~ 取反

   ~0011==&gt;1100

7. ^ 异或相同为0，不同为1

   0011^1011==&gt;1000 关于异或常见的一些如下所示：

   x^0 = x;

   x^1s = ~x; // 1s = ~0;

   x^-x = 1s;

   x^x = 0;

   c= a^b ==&gt;a^c = b; b^c = a;//交换两个数

   a^b^c = a^\(b^c\) = \(a^b\)^c;//结合

8. 常见的位运算
   1. 将x的最右边n位清零：x&\(~0&lt;&lt;n\)
   2. 获取x的第n位值（0或者1）：\(x&gt;&gt;n\)&1
   3. 获取x的第n位的幂值：x&\(1&lt;&lt;n\)
   4. 仅将第n位的为1：x\|\(1&lt;&lt;n\)
   5. 仅将第n位为0：x&\(~\(1&lt;&lt;n\)\)
   6. 将x的最高位至第n位清零：x&\(\(1&lt;&lt;n\)-1\)
9. 实战中位运算要点
   1. 判断奇偶性

      x%2==1 -----&gt;\(x&1\)==1;

      x%2==0 ------&gt;\(x&1\)==0;

   2. x&gt;&gt;1 -----&gt;x/2

      即 mid = （left+right）/2 ------&gt; mid = \(left+right\)&gt;&gt;1;

   3. 清零最低位的1

      x= x&\(x-1\);

   4. 得到最低位的1

      x&\(-x\)

   5. x&~x = 0;

### 布隆过滤器

他是一个很长的二进制向量和一系列的随机映射函数。布隆过滤器可以用于检索一个元素是否在集合中，有点事空间效率和查询时间优于一般算法。缺点是有一定的误识别率和删除困难（查询结果为1，可能存在也可能不存在，查询结果为0 ，一定为不存在的。

可以用于缓存的快速查询。

### LRU缓存

### 排序算法

十种常见排序算法可以分为两大类：

* **比较类排序**：通过比较来决定元素间的相对次序，由于其时间复杂度不能突破O\(nlogn\)，因此也称为非线性时间比较类排序。
* **非比较类排序**：不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此也称为线性时间非比较类排序。 

![](https://images2018.cnblogs.com/blog/849589/201804/849589-20180402133438219-1946132192.png)

