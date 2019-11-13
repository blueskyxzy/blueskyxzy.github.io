---
layout: post
title: 常规算法总结
date: 2019-11-13
tags: 总结    
---

### 算法归纳分类
排序，查找，图和字符串
#### 排序算法
    插入排序：直接插入排序insertsort，希尔排序(又称缩小增量排序)shellsort，二分插入排序（又称折半插入排序），链表插入排序
    选择排序：直接选择排序selectsort，堆排序heapsort
    交换排序: 冒泡排序popsort，快速排序quicksort
    归并排序mergesort
    基数排序radixsort
    计数排序countsort
    桶排序bucketsort
    
    稳定性算法:   基数排序 , 直接插入排序 , 冒泡排序, 归并排序
    不稳定性算法:  桶排序,    二分插入排序,希尔排序, 快速排序,   简单选择排序,堆排序

#### 十大算法：
    1：归并排序，快速排序和堆排序
    2：傅立叶变换与快速傅立叶变换
    3：Dijkstra （迪杰斯特拉）算法（求最短路径）
    4：RSA算法（质数加密）
    5：安全哈希算法
    6：整数因式分解许多（加密协议（如RSA算法）都基于这样一个原理：对大的合数作因式分解是非常困难的。如果一个算法能够快速地对任意整数进行因式分解，RSA的公钥加密体系就会失去其安全性。）
    7：链接分析
    8：比例积分微分算法
    9：数据压缩算法
    10：随机数生成
 
#### 其他
    贪心算法（又称贪婪算法）是指，在对问题求解时，总是做出在当前看来是最好的选择。也就是说，不从整体最优上加以考虑，他所做出的是在某种意义上的局部最优解。    
   
### 常规算法
#### 1.冒泡排序 popsort
    双循环冒泡  双循环 比较+交换
        for{ for{ if{ 交换} } }
    单循环冒泡  单循环 比较+交换+判断索引结束并重置索引和长度
        for{ if{交换} if{重置} }

#### 2.快速排序 quicksort
    设基准，两个索引，一个从左向右，一个从右向左，交换，左边的都小于基准，右边的都大于基准，然后基准左右边数组再排一遍（递归）
    while{ while{} if{交换} while{} if{交换} } if{递归} if{递归}

#### 3.直接插入排序 insertsort
    一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。
    for{ while{}}

#### 4.希尔排序 shellsort
    按增量分组，每组内进行直接插入排序，减半增量，直至0.
    最坏为O(n^2)，最好为O(n)，平均为O(n^1.3)
    d=len/2 while(d>0){for{ if{ while{}}  d/=2}

#### 5.直接选择排序 selectsort
    2遍历+交换(或者新数组赋值)
    for{for{if{交换}}}

#### 6.堆排序 heapsort
    构建大顶堆(左右树都不大于于对应根节点的完全二叉树) + 递归 + 交换 + 遍历
    1：构建大顶堆(左右树都不大于于对应根节点的完全二叉树)+递归。大顶堆就是最大的数。 mid = (len - 1)/2。递归写某个节点构造大顶堆，初始化数据的时候将mid到0构造成大顶堆
    2：交换+遍历。i从大到小len-1到0,大顶堆数0与最后一个i交换位置。每次交换后再构建大顶堆。

#### 7.归并排序 mergesort
    归并是对多个有序序列排序成一个有序的算法。
    对于一个无序序列，分成多个长度为1的有序序列，递归，即可完成无序序列的归并排序
    二分+递归+归并
    归并：while (i < n && j < m){if{}else{}} , while (i < n){} ,while (j < m){}

#### 8.计数排序 countsort

#### 9.基数排序 radixsort

#### 10.桶排序 bucketsort

#### 11.图的广度优先遍历和深度优先遍历

#### 12.最短路径的dijkstra算法    dijkstra

#### 13.单源最短路径BellmanFord算法  BellmanFord

#### 14.拓扑排序

#### 15.RSA

#### 16.动态规划

### Examples
#### 插入排序：向已经有序的序列中依次插入值到适当位置，使其仍然有序
##### 直接插入排序：
     1：1轮遍历。1，2 ... n依次插入。    for i in range(1, len(arr))
     2：直接插入。从i与i-1比较，将小的交换到前面    while arr[i] < arr[i-1]       交换 i--
##### 二分插入排序：
     1：1轮遍历。
     2：折半插入。
##### 希尔排序：
     1: 按增量分组。增量(波长)d=len(arr)/2，d /=2，d>=1.   for i in range(d, len(x))
     2：直接插入。   while x[i-d] > k and i >= d   交换 i-=d

#### 选择排序：每次从待排序的数据元素中选出最小（或最大）的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完
##### 直接选择排序：
     1：2轮遍历。i从0到len-1,j从i到len-1，比较记录记录最小数的索引
     2：交换。将最小的与i交换位置
##### 堆排序：
     1：构建大顶堆(左右树都不大于于对应根节点的完全二叉树)+递归。大顶堆就是最大的数。 mid = (len - 1)/2。递归写某个节点构造大顶堆，初始化数据的时候将mid到0构造成大顶堆
     2：交换+遍历。i从大到小len-1到0,大顶堆数0与最后一个i交换位置。每次交换后再构建大顶堆。

#### 交换排序：通过比较将键值较大的记录向序列的尾部移动，键值较小的记录向序列的前部移动。
##### 冒泡排序：
     1：2层循环。一般用2次for循环 0到len-1,len -1到i
     2：比较+交换。arr[j-1] > arr[j]，比较将最小的移到i
##### 快速排序：
     1：基准+2个索引。比基准小的排左边，小的排右边，二种方式：通过改变2个从两边到中间的左右索引和交换。通过2个从左到右的索引和交换的方式：(比low小1的索引)
     2：二分+递归。
##### 归并排序：2个序列合并成一个有序+递归
     1：递归。mid = (low + high)/2，从中间将序列分成多个2个序列
     2：2序列合并。将2个序列合并成一个有序序列。比较2序列最小的放到第三个序列(利用哨兵或者while),最初是长度是0或者1，都是有序的。

#### 最短路径dijkstra算法：以起始点为中心向外层层扩展，直到扩展到终点为止。（每次找到离源点最近的一个顶点，然后以该顶点为重心进行扩展）
     1:初始化。dis,T。dis记录顶点到各节点的当前的最短路径，T代表当前已经决定最短路径的顶点
     2:遍历。每一轮路径更新最短路径，直到更新全部路径
     3:确定一个顶点，返回2更新这个顶点的路径。确认顶点是dis中除了T中最小值那个顶点。


### 网上总结
#### 1.冒泡排序（Bubble Sort）
　　冒泡排序就是把小的元素往前调或者把大的元素往后调。比较是相邻的两个元素比较，交换也发生在这两个元素之间。所以，如果两个元素相等，我想你是不会再无聊地把他们俩交换一下的；如果两个相等的元素没有相邻，那么即使通过前面的两两交换把两个相邻起来，这时候也不会交换，所以相同元素的前后顺序并没有改变，所以冒泡排序是一种稳定的排序算法。

#### 2.选择排序（Selection sort）
　　选择排序是给每个位置选择当前元素最小的，比如给第一个位置选择最小的，在剩余元素里面给第二个元素选择第二小的，依次类推，直到第n - 1个元素，第n个元素不用选择了，因为只剩下它一个最大的元素了。那么，在一趟选择，如果当前元素比一个元素小，而该小的元素又出现在一个和当前元素相等的元素后面，那么交换后稳定性就被破坏了。比较拗口，举个例子，序列5 8 5 2 9，我们知道第一遍选择第1个元素5会和2交换，那么原序列中2个5的相对前后顺序就被破坏了，所以选择排序是不稳定的排序算法。

#### 3.插入排序（Insertion sort ）
　　插入排序是在一个已经有序的小序列的基础上，一次插入一个元素。当然，刚开始这个有序的小序列只有1个元素，就是第一个元素。比较是从有序序列的末尾开始，也就是想要插入的元素和已经有序的最大者开始比起，如果比它大则直接插入在其后面，否则一直往前找直到找到它该插入的位置。如果碰见一个和插入元素相等的，那么插入元素把想插入的元素放在相等元素的后面。所以，相等元素的前后顺序没有改变，从原无序序列出去的顺序就是排好序后的顺序，所以插入排序是稳定的。

#### 4.快速排序（Quick sort）
　　快速排序（Quick sort）是对冒泡排序的一种改进，采用的是分治的思想。快速排序简单的说就是选择一个基准（一般选第一个数），将比基准大的数放在一边，小的数放到另一边。对这个数的两边再递归上述方法。  
　　一趟快速排序的算法是：  
1）设置两个变量i、j，排序开始的时候：i=0，j=N-1；  
2）以第一个数组元素作为关键数据，赋值给key，即key=A[0]；  
3）从j开始向前搜索，即由后开始向前搜索(j–)，找到第一个小于key的值A[j]，将A[j]和A[i]互换；  
4）从i开始向后搜索，即由前开始向后搜索(i++)，找到第一个大于key的A[i]，将A[i]和A[j]互换；  
5）重复第3、4步，直到i=j； (3,4步中，没找到符合条件的值，即3中A[j]不小于key,4中A[i]不大于key的时候改变j、i的值，使得j=j-1，i=i+1，直至找到为止。找到符合条件的值，进行交换的时候i， j指针位置不变。另外，i==j这一过程一定正好是i+或j-完成的时候，此时令循环结束）。  
　　如：排列 66 13 51 76 81 26 57 69 23  
　　以66为基准，升序排序的话，比66小的放左边，比66大的放右边， 类似这种情况 13 。。。 66。。。69  

　　具体快速排序的规则一般如下：  
　　从右边开始查找比66小的数，找到的时候先等一下，再从左边开始找比66大的数，将这两个数借助66互换一下位置，继续这个过程直到两次查找过程碰头。  
　　例子中：  
        66  13  51  76  81  26  57  69  23  
从右边找到23比66小，互换  
        23  13  51  76  81  26  57  69  66  
从左边找到76比66大，互换  
        23  13  51  66  81  26  57  69  76  
继续从右边找到57比66小，互换  
        23  13  51  57  81  26  66  69  76  
从左边查找，81比66大，互换  
        23  13  51  57  66  26  81  69  76  
从右边开始查找26比66小，互换  
        23  13  51  57  26  66  81  69  76  
此时66左边的数都比66小，66右边的数都比66大，完成一轮排序，对这个数的两边再递归上述方法。  

　　在中枢元素和a[j]交换的时候，很有可能把前面的元素的稳定性打乱，比如序列为5 3 3 4 3 8 9 10 11，现在中枢元素5和3（第5个元素，下标从1开始计）交换就会把元素3的稳定性打乱，所以快速排序是一个不稳定的排序算法，不稳定发生在中枢元素和a[j] 交换的时刻。  

#### 5.归并排序（MERGE-SORT）
　　归并排序（MERGE-SORT）是建立在归并操作上的一种有效的排序算法,该算法是采用分治法（Divide and Conquer）的一个非常典型的应用。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为二路归并。  
　　如　设有数列{6，202，100，301，38，8，1}  

初始状态：6,202,100,301,38,8，1  
第一次归并后：{6,202},{100,301},{8,38},{1}，比较次数：3；  
第二次归并后：{6,100,202,301}，{1,8,38}，比较次数：4；  
第三次归并后：{1,6,8,38,100,202,301},比较次数：4；  
总的比较次数为：3+4+4=11； 

　　归并排序是把序列递归地分成短序列，递归出口是短序列只有1个元素（认为直接有序）或者2个序列（1次比较和交换），然后把各个有序的段序列合并成一个有序的长序列，不断合并直到原序列全部排好序。可以发现，在1个或2个元素时，1个元素不会交换，2个元素如果大小相等也没有人故意交换，这不会破坏稳定性。那么，在短的有序序列合并的过程中，稳定是是否受到破坏？没有，合并过程中我们可以保证如果两个当前元素相等时，我们把处在前面的序列的元素保存在结果序列的前面，这样就保证了稳定性。所以，归并排序也是稳定的排序算法。

#### 6.基数排序（radix sort）
　　基数排序（radix sort）属于“分配式排序”（distribution sort），又称“桶子法”（bucket sort）或bin sort，基数排序是按照低位先排序，然后收集；再按照高位排序，然后再收集；依次类推，直到最高位。有时候有些属性是有优先级顺序的，先按低优先级排序，再按高优先级排序，最后的次序就是高优先级高的在前，高优先级相同的低优先级高的在前。基数排序基于分别排序，分别收集，所以其是稳定的排序算法。  
　　如有一串数值如下所示：  
　　73, 22, 93, 43, 55, 14, 28, 65, 39, 81  
　　首先根据个位数的数值，在遍历数据时将它们各自分配到编号0至9的桶（个位数值与桶号一一对应）中。  

　　分配结果（逻辑想象）如下图所示：  
这里写图片描述  
　　分配结束后。接下来将所有桶中所盛数据按照桶号由小到大（桶中由顶至底）依次重新收集串起来，得到如下仍然无序的数据序列：  

81  22  73  93  43  14  55  65  28  39//个位是自然排序的  
　　接着，再进行一次分配，这次根据十位数值来分配（原理同上），分配结果（逻辑想象）如下图所示：  
这里写图片描述  
　　分配结束后。接下来再将所有桶中所盛的数据（原理同上）依次重新收集串接起来，得到如下的数据序列：  

14  22  28  39  43  55  65  73  81  93  
　　观察可以看到，此时原无序数据序列已经排序完毕。如果排序的数据序列有三位数以上的数据，则重复进行以上的动作直至最高位数为止。  

#### 7.希尔排序（shell sort）
　　希尔排序(Shell Sort)是插入排序的一种。也称缩小增量排序，是直接插入排序算法的一种更高效的改进版本。希尔排序是不稳定排序算法。  
　　希尔排序的大体思路是：将一个未排序的序列分成多个组，然后在组内使用插入排序对组内序列排序。我们的分组方式是将每隔固定位置（增量）的元素分成一组。之后去调整这个间隔大小，重新分组，组内重新排序。直到分组的间隔为1，也就是所有的元素分成一组，再进行一次插入排序，这样就可以完成整个序列的排序过程。  
希尔排序是按照不同步长对元素进行插入排序，当刚开始元素很无序的时候，步长最大，所以插入排序的元素个数很少，速度很快；当元素基本有序了，步长很小， 插入排序对于有序的序列效率很高。所以，希尔排序的时间复杂度会比O(n^2)好一些。由于多次插入排序，我们知道一次插入排序是稳定的，不会改变相同元素的相对顺序，但在不同的插入排序过程中，相同的元素可能在各自的插入排序中移动，最后其稳定性就会被打乱，所以shell排序是不稳定的。  
　　例如，假设有这样一组数[ 13 14 94 33 82 25 59 94 65 23 45 27 73 25 39 10 ]，如果我们以步长为5开始进行排序，我们可以通过将这列表放在有5列的表中来更好地描述算法，这样他们就应该看起来是这样：  

   13 14 94 33 82  
   25 59 94 65 23  
   45 27 73 25 39  
   10  
然后我们对每列进行排序：  
   10 14 73 25 23  
   13 27 94 33 39  
   25 59 94 65 82  
   45  

将上述四行数字，依序接在一起时我们得到：[ 10 14 73 25 23 13 27 94 33 39 25 59 94 65 82 45 ].这时10已经移至正确位置了，然后再以3为步长进行排序：  
   10 14 73  
   25 23 13  
   27 94 33  
   39 25 59  
   94 65 82  
   45  

排序之后变为：  
   10 14 13  
   25 23 33  
   27 25 59  
   39 65 73  
   45 94 82  
   94  

　　依序连接在一起得到：[10 14 13 25 23 33 27 25 59 39 65 73 45 94 82 94]  
最后以1步长进行排序（此时就是简单的插入排序了）。  

#### 8.堆排序（Heapsort sort）
　　堆分为大根堆和小根堆，是完全二叉树。大根堆的要求是每个节点的值都不大于其父节点的值，即A[PARENT[i]] >= A[i]。在数组的非降序排序中，需要使用的就是大根堆，因为根据大根堆的要求可知，最大的值一定在堆顶。  
　　既然是堆排序，自然需要先建立一个堆，而建堆的核心内容是调整堆，使二叉树满足堆的定义（每个节点的值都不大于其父节点的值）。调堆的过程应该从最后一个非叶子节点开始，假设有数组A = {1, 3, 4, 5, 7, 2, 6, 8, 0}。  
那么调堆的过程如下图，数组下标从0开始，A[3] = 5开始。分别与左孩子和右孩子比较大小，如果A[3]最大，则不用调整，否则和孩子中的值最大的一个交换位置，在图1中是A[7] > A[3] > A[8]，所以A[3]与A[7]对换，从图1.1转到图1.2。  
　　我们知道堆的结构是节点i的孩子为2 * i和2 * i + 1节点，大顶堆要求父节点大于等于其2个子节点，小顶堆要求父节点小于等于其2个子节点。在一个长为n 的序列，堆排序的过程是从第n / 2开始和其子节点共3个值选择最大（大顶堆）或者最小（小顶堆），这3个元素之间的选择当然不会破坏稳定性。但当为n / 2 - 1， n / 2 - 2， … 1这些个父节点选择元素时，就会破坏稳定性。有可能第n / 2个父节点交换把后面一个元素交换过去了，而第n / 2 - 1个父节点把后面一个相同的元素没 有交换，那么这2个相同的元素之间的稳定性就被破坏了。所以，堆排序不是稳定的排序算法。

　　综上，得出结论: 选择排序、快速排序、希尔排序、堆排序不是稳定的排序算法，而冒泡排序、插入排序、归并排序和基数排序是稳定的排序算法