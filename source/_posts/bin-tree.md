---
title: 二叉搜索树
category:
  - 算法
  - Java
tags:
  - 数据结构
  - 算法
  - 排序算法
  - 查找算法
  - Java
description: "总结了一下二叉搜索树的相关内容。包括二叉搜索树的建立，二叉搜索树的查找，二叉搜索树的删除。由于二叉搜索树的结构，其中序遍历就是排序操作。所以，二叉搜索树是一种可以同时运用到排序和查找中的结构体，并且在排序和查找中都有优势。"
date: 2016-12-12 17:44:29
---
## 二叉搜索树
二叉搜索树是满足以下条件的二叉树：1.左子树上的所有节点值均小于根节点值，2右子树上的所有节点值均不小于根节点值，3，左右子树也满足上述两个条件。

## 二叉搜索树的优点
二叉搜索树又叫二叉排序树。结合了两种数据结构的有点：一种是有序数组，二叉搜索树在查找数据项的速度和在有序数组中查找一样快；另一种是链表，二叉搜索树在插入数据和删除数据项的速度和链表一样。

## 基本数据结构
### 节点
```
public class BinTreeNode {
	public int data;
	public BinTreeNode left;
	public BinTreeNode right;

	public BinTreeNode(int data){
		this.data = data;
	}
	
	public void displayNode(){
		Log.d("BinTree", data+" ");
	}
}
```

### 树
```
public class BinTree {
	public BinTreeNode root;
	
	//以下变量用来在find中保存状态，这样，delete就可以直接调用find了进行待删除节点的查找工作。
	public BinTreeNode current;
	public BinTreeNode parent;
	boolean isLeftChild;
	
	//各种方法
}
```

## 插入方法
```
	public void insert(int data){
		BinTreeNode node = new BinTreeNode(data);
		if(root == null){
			root = node;
		}else{
			BinTreeNode current = root;
			BinTreeNode parent;
			while(true){
				parent = current;
				if(data < current.data){
					current = current.left;
					if(current == null){
						parent.left = node;
						return;
					}
				}else{
					current = current.right;
					if(current == null){
						parent.right = node;
						return;
					}
				}
			}
		}
	}
```

## 中序遍历
```
	public void displayTree(BinTreeNode root){
		inOrder(root);//中序遍历
	}
	
	public void inOrder(BinTreeNode root){
		if(root != null){
			inOrder(root.left);
			root.displayNode();
			inOrder(root.right);
		}
	}
```

## 查找方法
```
	public BinTreeNode find(int data){
		current = root;
		parent = root;
		isLeftChild = false;
		
		while(current.data != data){
			parent = current;
			if(current.data < data){
				current = current.right;
				isLeftChild = false;
			}else{
				current = current.left;
				isLeftChild = true;
			}
			
			if(current == null)
				return null;
		}
		return current;
	}
```

## 删除方法
### 如果删除节点是一个叶子节点
这种情况最简单，直接删除这个节点就行
![image](/assets/img/bin_tree/del_leaf.jpg)

### 如果删除节点只有一个子节点
这种情况，只需要把删除节点的子节点，顶替删除节点的位置
![image](/assets/img/bin_tree/del_one_child.jpg)

### 如果删除节点有两个子节点
这种情况需要以下步骤
1. 找到删除节点的后续节点，也就是排序中的下一个节点，也就是所有比删除节点大的节点中，最小的那一个。这个节点，必然时删除节点的右子树中最小的那个点，它不会有左子树。
2. 调整后继节点及子树的结构，让其成为一个只有右子树，且结构正确的树
3. 把删除节点的左子树变成后继节点的左子树
4. 让后继节点取代删除节点的位置
![image](/assets/img/bin_tree/del_two_child.jpg)

### 代码实现

```
public void delete(int data){
		//首先找到这个删除结点，在这个过程中，会更新current，parent，isLeftChild三个全局变量
		find(data);
		
		//找到删除结点后，要分三种情况：
		if(current != null){
			//1.删除节点是叶子节点，直接删掉
			if(current.left == null && current.right == null){
				if(current == root){
					root = null;
				}else{
					if(isLeftChild){
						parent.left = null;
					}else{
						parent.right = null;
					}
				}
			}
			
			//2.删除节点只有一个子节点（只有左子节点，或者只有右子节点）
			else if(current.right == null){
				//只有左子节点
				if(current == root){
					root = current.left;
				}else if(isLeftChild){
					parent.left = current.left;
				}else{
					parent.right = current.left;
				}
			}
			else if(current.left == null){
				//只有右子节点
				if(current == root){
					root = current.right;
				}else if(isLeftChild){
					parent.left = current.right;
				}else{
					parent.right = current.right;
				}
			}
			
			//3.左右子节点都有
			//首先要找到它的后继节点，也就是中序的下一节点
			else{
				BinTreeNode successor = getSuccessor(current);
				successor.left = current.left;
				if(current == root){
					root = successor;
				}
				//把删除节点的左子树变成后继节点的左子树
				//并且让后继节点取代删除节点的位置
				else if(isLeftChild){
					parent.left = successor;
				}else{
					parent.right = successor;
				}
			}
		}
		
	}

	public BinTreeNode getSuccessor(BinTreeNode delNode) {
		// 这个函数用来给有右子树的节点寻找后继节点。
		// 可以想象，这个后继节点必然是删除节点的右子树中，最左最下的那个孩子
		// 也就是说，这个后继节点必然没有左子节点
		// 此外还负责调整树的结构，使后继节点成为一个只有右子树的，排序结构正确的树
		BinTreeNode successorParent = delNode;
		BinTreeNode successor = delNode;
		BinTreeNode current = delNode.right;
		
		while(current != null){
			successorParent = successor;
			successor = current;
			current = current.left;
		}
		
		if(successor != delNode.right){
			successorParent.left = successor.right;
			successor.right = delNode.right;
		}
		
		return successor;
	}
```

## 项目代码
该项目作为SearchAndSort的一个组成部分。
[SortAndFind](https://github.com/IChrisKing/SortAndFind)
`git@github.com:IChrisKing/SortAndFind.git`