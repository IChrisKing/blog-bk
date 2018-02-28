---
title: 各种查找算法
category:
  - 算法
  - Java
tags:
  - 查找算法
  - 算法
  - Java
description: "顺序查找"
date: 2016-11-21 20:42:33
---
## 源码地址
[SortAndFind](https://github.com/IChrisKing/SortAndFind)
`git@github.com:IChrisKing/SortAndFind.git`

## 顺序查找
* 需求

在array中查找是否存在value，如果存在多个，返回value的所有位置。

```
public class SequelSearch {
	static int[] array;
	static int value;

	public static HashSet<Integer> searchInArray(int[] arr, int val){
		array = arr;
		value = val;
		HashSet<Integer> result = sequelSearch();
		return result;
	}

	private static HashSet<Integer> sequelSearch() {
		// TODO Auto-generated method stub
		HashSet<Integer> res = new HashSet<Integer>();
		for(int i=0;i<array.length;i++){
			if(array[i] == value){
				res.add(i);
			}
		}
		return res;
	}

}

```

## 二分查找
* 需求

在array中查找是否存在value，如果存在，返回位置，如果不存在，返回-1。

使用递归和递推两种方式实现。

```
public class BinarySearch {

	static int[] array;
	static int value;
//	int mid;
	
	public static int searchInArray(int[] arr, int val){
		array = arr;
		value = val;
		int result = binarySearch(0,array.length-1);
		return result;
	}

	//递归方式
	private static int binarySearch(int low, int high) {
		// TODO Auto-generated method stub
		int mid = (low + high)/2;
		if(low<=high){
			if(array[mid] == value){
				return mid;
			}else if(array[mid] < value){
				binarySearch(mid+1, high);
			}else{
				binarySearch(low, mid-1);
			}
		}
		
		return -1;
	}
	
	//递推方式
	private static int binarySearch(int low, int high) {
		// TODO Auto-generated method stub
		int mid = (low + high)/2;
		while(low<high){
			if(array[mid] == value){
				return mid;
			}else if(array[mid] < value){
				low = mid+1;
			}else{
				high = mid-1;
			}
		}
		
		return -1;
	}
	
}
```


