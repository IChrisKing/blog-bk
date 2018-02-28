---
title: 各种排序算法
category:
  - 算法
  - Java
tags:
  - 排序算法
  - 算法
  - Java
description: "直接插入排序，希尔排序，简单选择排序，堆排序，冒泡排序，快速排序，归并排序"
date: 2016-11-02 18:42:33
---
## 源码地址
[SortAndFind](https://github.com/IChrisKing/SortAndFind)
`git@github.com:IChrisKing/SortAndFind.git`

## 总结
所需辅助空间最多：归并排序 

所需辅助空间最少：堆排序 

平均速度最快：快速排序 

不稳定：快速排序，希尔排序，堆排序。 

## 直接插入排序
在要排序的一组数中，假设前面(n-1)[n>=2] 个数已经是排好顺序的，现在要把第n 个数插到前面的有序数中，使得这n个数也是排好顺序的。如此反复循环，直到全部排好顺序。
```
	  public static void insertSort(int[] arr){    
			array = arr;
	      int temp=0;   
	      for(int i=1;i<array.length;i++){   
	         int j=i-1;   
	        temp=array[i];   
	        for(;j>=0&&temp<array[j];j--){   
	            array[j+1]=array[j];  //将大于temp 的值整体后移一个单位   
	        }   
	        array[j+1]=temp;   
	     }   
	 }  

```

## 希尔排序
算法先将要排序的一组数按某个增量 d（n/2,n为要排序数的个数）分成若干组，每组中记录的下标相差 d.对每组中全部元素进行直接插入排序，然后再用一个较小的增量（d/2）对它进行分组，在每组中再进行直接插入排序。当增量减到 1 时，进行直接插入排序后，排序完成。
```
	public static void shellSort(int[] arr){
		array = arr;
		d=arr.length/2;

		while(d>1){
			begin = 0;
			while(begin<d){
				insertSortForShell(begin,d);
				begin++;
			}
			d=d/2;
		}
		
		insertSortForShell(0,1);
	}
	

	  public static void insertSortForShell(int begin, int d){    
	      int temp;   
	      for(int i=begin+d;i<array.length;i+=d){   
	         int j=i-d;   
	        temp=array[i];   
	        for(;j>=0&&temp<array[j];j-=d){   
	            array[j+d]=array[j];  
	        }   
	        array[j+d]=temp;   
	     }   
	 } 
```

## 简单选择排序
在要排序的一组数中，选出最小的一个数与第一个位置的数交换；然后在剩下的数当中再找最小的与第二个位置的数交换，如此循环到倒数第二个数和最后一个数比较为止。
```
public static void selectSort(int[] arr){
		array = arr;
		for(int i=0;i<array.length;i++){
			pos = findMinPos(i);
			int temp = array[i];
			array[i]=array[pos];
			array[pos]=temp;
		}
	}

	private static int findMinPos(int i) {
		int min = array[i];
		int minPos = i;
		for(int j=i+1;j<array.length;j++){
			if(array[j]<min){
				min = array[j];
				minPos = j;
			}
		}
		return minPos;
	}
```

## 堆排序
堆排序是一种树形选择排序，是对直接选择排序的有效改进。

堆的定义如下：具有n个元素的序列（h1,h2,…,hn),当且仅当满足（hi>=h2i,hi>=2i+1）或（hi<=h2i,hi<=2i+1） (i=1,2,…,n/2)时称之为堆。在这里只讨论满足前者条件的堆。由堆的定义可以看出，堆顶元素（即第一个元素）必为最大项（大顶堆）。完全二叉树可以很直观地表示堆的结构。堆顶为根，其它为左子树、右子树。初始时把要排序的数的序列看作是一棵顺序存储的二叉树，调整它们的存储序，使之成为一个堆，这时堆的根节点的数最大。然后将根节点与堆的最后一个节点交换。然后对前面(n-1)个数重新调整使之成为堆。依此类推，直到只有两个节点的堆，并对它们作交换，最后得到有n个节点的有序序列。从算法描述来看，堆排序需要两个过程，一是建立堆，二是堆顶与堆的最后一个元素交换位置。所以堆排序有两个函数组成。一是建堆的渗透函数，二是反复调用渗透函数实现排序的函数。

堆排序是不稳定的排序方法，辅助空间为O(1)， 最坏时间复杂度为O(nlog2n) ，堆排序的堆序的平均性能较接近于最坏性能。 
```
	public static int[] array;
	static int end;
	
	public static void sortArray(int[] arr){
		array = arr;
		end = array.length - 1;
		for(int i=0;i<array.length-1;i++){
			buildMaxHeap(end);
			swap(0,end);
			end--;
		}
	}

	private static void swap(int i, int end) {
		// TODO Auto-generated method stub
		int temp = array[i];
		array[i] = array[end];
		array[end] = temp;
	}

	private static void buildMaxHeap(int end) {
		// TODO Auto-generated method stub
		int biggerIndex = 0;
		//从最后一个节点的父节点开始
		for(int j = (end-1)/2;j>=0;j--){
			//k标记当前比较的节点
			int k = j;
			//当节点的左孩子存在时
			while(k*2+1<=end){
				//现假设左孩子是两个孩子中的较大者
				biggerIndex = k*2+1;
				//查看右孩子是否存在
				if(biggerIndex+1<=end){
					//查看左右孩子哪个更大
					if(array[biggerIndex]< array[biggerIndex+1]){
						//如果右孩子更大，更改biggerIndex，使它指向右孩子
						biggerIndex++;
					}
				}
				
				//查看当前节点和孩子中的较大节点哪个更大
				if(array[k]<array[biggerIndex]){
					//如果孩子中的较大节点更大，交换当前节点和较大孩子节点的值
					swap(k,biggerIndex);
					//把较大孩子节点作为下一次while的当前节点，来考察经过一次交换之后，有没有影响到子树的正确性。
					k=biggerIndex;
				}else{
					break;//没有交换操作，则不需要检查子树的正确性
				}
			}
		}
	}
```

## 冒泡排序
在要排序的一组数中，对当前还未排好序的范围内的全部数，自上而下对相邻的两个数依次进行比较和调整，让较大的数往下沉，较小的往上冒。即：每当两相邻的数比较后发现它们的排序与排序要求相反时，就将它们互换。 
```
	public static int[] array;
	
	public static void sortArray(int[] arr){
		array = arr;
		for(int i=0;i<array.length;i++){
			bubble(i);
		}
	}

	private static void bubble(int i) {
		// TODO Auto-generated method stub
		int tag=array.length-1;
		while(tag>i){
			if(array[tag]<array[tag-1]){
				swap(tag,tag-1);
			}
			tag--;
		}
	}
	
	private static void swap(int i, int end) {
		// TODO Auto-generated method stub
		int temp = array[i];
		array[i] = array[end];
		array[end] = temp;
	}
```

## 快速排序
选择一个基准元素,通常选择第一个元素或者最后一个元素,通过一趟扫描，

将待排序列分成两部分,一部分比基准元素小,一部分大于等于基准元素,此时基准元素在其

排好序后的正确位置,然后再用同样的方法递归地排序划分的两部分。 
```
public static void sortArray(int[] arr){
		array=arr;
		sort(0,array.length-1);
	}

	private static void sort(int low, int high) {
		// TODO Auto-generated method stub
		if(low<high){
			int middle =getMiddle(low, high);  
            sort(low, middle - 1);       //对低字表进行递归排序     
            sort(middle + 1, high);       //对高字表进行递归排
		}
	}
	
	public static int getMiddle(int low, int high) {     
             int tmp =array[low];    //数组的第一个作为中轴     
             while (low < high){     
                 while (low < high&& array[high] >= tmp) {     
                    high--;     
                 }     
    
                 array[low] =array[high];   //比中轴小的记录移到低端     
                 while (low < high&& array[low] <= tmp) {     
                     low++;     
                 }     
    
                 array[high] =array[low];   //比中轴大的记录移到高端     
             }     
            array[low] = tmp;              //中轴记录到尾     
             return low;                   //返回中轴的位置     
 } 
```

## 归并排序
归并排序（Merge）是将两个（或两个以上）有序表合并成一个新的有序表，即把待排序序列分为若干个子序列，每个子序列是有序的。然后再把有序子序列合并为整体有序序列。

归并排序是建立在归并操作上的一种有效的排序算法。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用。 将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为2-路归并。

归并排序算法稳定，数组需要O(n)的额外空间，链表需要O(log(n))的额外空间，时间复杂度为O(nlog(n))，算法不是自适应的，不需要对数据的随机读取。

* 工作原理
1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列

2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置

3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置

4. 重复步骤3直到某一指针达到序列尾

5. 将另一序列剩下的所有元素直接复制到合并序列尾

```
	static int[] array;
//	static int left,right,mid;
	public static void sortArray(int[] arr){
		array = arr;
		sort(0,array.length-1);
	}
	private static void sort(int left, int right) {
		// TODO Auto-generated method stub
		if(left<right){
			int center = (left+right)/2;
			sort(left,center);
			sort(center+1,right);
			merge(left,center,right);
		}
	}
	private static void merge(int left, int center, int right) {
		// TODO Auto-generated method stub
		int[] tmpArr = new int[array.length];
		int tmpTag = left;
		int leftTag = left;
		int rightTag = center+1;
		while(leftTag<=center && rightTag<=right){
			if(array[leftTag]<array[rightTag]){
				tmpArr[tmpTag]=array[leftTag];
				tmpTag++;
				leftTag++;
			}else{
				tmpArr[tmpTag]=array[rightTag];
				tmpTag++;
				rightTag++;
			}
		}
		
		while(leftTag<=center){
			tmpArr[tmpTag]=array[leftTag];
			tmpTag++;
			leftTag++;
		}
		while(rightTag<=right){
			tmpArr[tmpTag]=array[rightTag];
			tmpTag++;
			rightTag++;
		}
		
		tmpTag = left;
		while(tmpTag<=right){
			array[tmpTag]=tmpArr[tmpTag];
			tmpTag++;
		}
	}
```
