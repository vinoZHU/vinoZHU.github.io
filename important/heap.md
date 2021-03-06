---
title: 堆
date: 2016-11-07 15:42:09
tags: 
- 堆
- 数据结构
- 排序
- 算法
categories: 算法与数据结构
---

#### 堆介绍
堆（也叫优先队列），是一棵完全二叉树，它的特点是父节点的值大于（小于）两个子节点的值（分别称为大顶堆和小顶堆）。堆可以用数组来存储，但为了和完全二叉树的性质对应，数组的0号元素不存储节点，0号元素可以用来记录堆的节点数量。由于是完全二叉树，假定`i`是其中某个节点，可定义如下操作：

- PARENT(i) = i/2
- LEFT(i) = 2*i
- RIGHT(i) = 2*i+1

大顶堆以及数组存储：
![](/images/data-structure/heap-0.png)

#### 维护堆的性质
对于大顶堆，如果父节点的值小于某个子节点，那么这个堆的性质就需要恢复，过程如下：
![](/images/data-structure/heap-1.png)
这里的前提是i的左右孩子都是堆（在建堆的时候是自下而上的），然后在A[i]、A[LEFT(i)]、A[RIGHT(i)]中选择一个最大的，如果A[i]最大，则堆的性质没有被破坏，否则A[i]和某个孩子交换位置，然后使交换位置后的子节点重复上述过程：

```c
void maxHeapify(int A[], int i, int n){//n为堆的容量大小 
  int left = LEFT(i);
  int right = RIGHT(i);
  int max = i;//max记录最大节点的下标
  if(left <= n){
  		max = A[left] > A[max] ? left : max;
  }      
  if(right <= n) {
  		max = A[right] > A[max] ? right : max;
  }    
  if(max != i){
    swap(A, max, i);
    maxHeapify(A, max, n);
  }
 
}
```

#### 建堆
给定一个数组A，n=A.length,其子数组A[(n/2)+1..n]中的元素都是堆这颗完全二叉树的叶子节点，可以利用这个性质,先找到堆的最后一个非叶子节点（即为第n/2个节点），然后从该节点开始，从后往前逐个调整每个子树，使之称为堆，最终整个数组便是一个堆。来将这个数组转换成一个大顶堆，以下是一个示意图：
![](/images/data-structure/heap-2.png)

```c
void buildMaxHeap(int A[], int n){
  int i;
  for(i = n/2; i>0; i--){
    maxHeapify(A, i, n);//依次向上维护堆的性质
  }
 
}
```

#### 堆排序
给定一个数组A,使用堆排序的步骤如下：

1. 建堆
2. n:=heap-size,swap(A[1],A[n])
3. heap-size--
4. heapify(A,1,heap-size)//维护堆
5. if(heap-size>0) goto (2)

示意图：
![](/images/data-structure/heap-3.png)
![](/images/data-structure/heap-4.png)

```c
void heapSort(int A[], int n){
	buildMaxHeap(A,n);
	int i;
	while(n > 0){
		swap(A[1],A[i]);
		n--;
		maxHeapify(A,1,n);
	}

}
```

#### 优先队列
堆当作优先队列使用时，每个元素都有一个key，常见操作如下：

- insert(A,x):将元素x插入到队列中
- maximum（A):返回最大key所在的元素
- extract-max(A):返回并删除最大key所在的元素
- increase-key(A,x,k):将元素x的key增加为k，假设k大于x

```c
int heapMaximun(int A[], int n){
	if(n < 1) return -1;//error
	return A[1];
}
```

```c
int heapExtractMax(int A[], int n){
	if(n < 1) return -1;//error
	int max = A[1];
	A[1] = A[n];
	n--;
	maxHeapify(A,1,n);
	return max;
}
```

```c
void heapIncreaseKey(int A[], int i, int key){
	if(key < A[i]) error;
	A[i] = key;
	while(i > 1 && A[PARENT(i)] < A[i]){
		swap(A[i],A[PARENT(i)]);
		i = PARENT(i);
	}
}
```


```c
void maxHeapInsert(int A[],int x,int n){
	A[++n] = x;
	while(n > 1 && A[n] > A[PARENT(n)]){
		swap(A[n],A[PARENT(n)]);
		n = PARENT(n);
	}
}
```

#### 堆的应用
- 堆的最常见应用是堆排序，时间复杂度为O(N lg N)。如果是从小到大排序，用大顶堆；从大到小排序，用小顶堆。

- 在O(n lg k)时间内，将k个排序表合并成一个排序表，n为所有有序表中元素个数。

- 一个文件中包含了1亿个随机整数，如何快速的找到最大(小)的100万个数字?（时间复杂度：O（n lg k））

	【解析】取前100 万个整数，构造成了一棵数组方式存储的具有小顶堆，然后接着依次取下一个整数，如果它大于最小元素亦即堆顶元素，则将其赋予堆顶元素，然后用Heapify调整整个堆，如此下去，则最后留在堆中的100万个整数即为所求 100万个数字。该方法可大大节约内存。

参考：

- 《算法导论》

><font color= Darkorange>因本人水平有限，若文章内容存在问题，恳请指出。允许转载，转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 



