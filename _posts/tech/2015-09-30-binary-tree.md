---
layout: post
title: 二叉树面试题集锦
category: 技术
tags: 面试题
keywords: 二叉树, 面试题
description: 
---  

### 二叉树面试题

#### 1. 遍历系列
##### 1.1 前序遍历

###### a.递归实现

	    public static <T> void formPreoderTraversal(Node<T> root) {
        	if (null != root) {
            	System.out.println(root.value);
            	formPreoderTraversal(root.left);
            	formPreoderTraversal(root.right);
        	}
    	}
---
###### b.非递归实现  

思路:  
  
1. currentNode 指向 root 节点  
2. 如果 currentNode 不为 null，输出 currentNode，并入栈存入 stack。currentNode = currentNode.left, 重复步骤 2)  
3. 如果 currentNode 为 null， 则开始出栈，出栈的节点为 node，输出 node。如果 node 有右孩子，则将 currentNode 指向这个右孩子， 然后重复步骤 2)。如果出栈节点无右孩子， 则继续重复出栈的过程， 直到出栈的节点有右孩子，或者栈为空为止。    


终止条件为 currentNode 为 null， 且 stack 为空为止。代码如下：  
	   
	   public static <T> void formPreoderTraversal(Node<T> root) {
            if (null == root) {
                return;
            }
    
            Stack<Node<T>> stack = new Stack<Node<T>>();
            Node<T> currNode = root;
    
            while(currNode != null || !stack.isEmpty()) {
                if (null != currNode) {
                    System.out.println(currNode.value);
                    stack.push(currNode);
                    currNode = currNode.left;
                } else {
                    while(!stack.isEmpty()) {
                        Node<T> node = stack.pop();
                        if (node.right != null) {
                            currNode = node.right;
                            break;
                        }
                    }
                }
            }
        }
        
---        
##### 1.2 中序遍历
###### a.递归实现

	    public static <T> void preoderTraversal(Node<T> root) {
        	if (null != root) {
            	formPreoderTraversal(root.left);
            	System.out.println(root.value);
            	formPreoderTraversal(root.right);
        	}
    	}
---
###### b.非递归实现  

思路：  
  
1. currentNode 指向 root 节点
2. 如果 currentNode 不为 null，并入栈存入 stack。currentNode = currentNode.left, 重复步骤 2）
3. 如果 currentNode 为 null， 则开始出栈，出栈的节点为 node，输出 node。如果 node 有右孩子， 则将 currentNode 指向这个右孩子， 然后重复步骤 2)。如果出栈节点无右孩子， 则继续重复出栈的过程， 直到出栈的节点有右孩子，或者栈为空为止。

终止条件为 currentNode 为 null， 且 stack 为空为止。可以看出前序遍历的非递归算法和中序遍历的过程基本是一样的， 唯一的差别就是输出节点的时间不一样， 前序遍历是在入栈的时候输出节点， 中序遍历是在出栈的时候输出节点。这也很好理解， 因为从上述过程可以看出，入栈的过程左孩子在父亲节点的前面入栈， 所以前部遍历的时候在入栈之前输出节节点就是父亲节点在左孩子之前输出， 反之在出栈的时候输出节点就是父亲节点在做孩子之后输出。实现代码如下：
	   
	   public static <T> void formPreoderTraversal(Node<T> root) {
            if (null == root) {
                return;
            }
    
            Stack<Node<T>> stack = new Stack<Node<T>>();
            Node<T> currNode = root;
    
            while(currNode != null || !stack.isEmpty()) {
                if (null != currNode) {
                    System.out.println(currNode.value);
                    stack.push(currNode);
                    currNode = currNode.left;
                } else {
                    while(!stack.isEmpty()) {
                        Node<T> node = stack.pop();
                        if (node.right != null) {
                            currNode = node.right;
                            break;
                        }
                    }
                }
            }
        }

---
##### 1.3 后序遍历
###### a.递归实现

	    public static <T> void afterPreoderTraversal(Node<T> root) {
        	if (null != root) {
            	formPreoderTraversal(root.left);
            	formPreoderTraversal(root.right);
            	System.out.println(root.value);
        	}
    	}
---
###### b.非递归实现  

思路：  

1. currentNode 指向 root 节点.
2. 如果 currentNode 不为 null，并入栈存入 stack。currentNode = currentNode.left, 重复步骤 2）。
3. 如果 currentNode 为 null， 则检查栈顶元素的右孩子。 如果右孩子为空或者上一个出栈的节点即为其右孩子， 则出栈， 输出该出栈节点。否则将 currentNode 指向该右孩子，然后重复步骤 2)。

终止条件为 currentNode 为 null， 且 stack 为空为止。代码如下：  
	   
        public static <T> void afterPreoderTraversal(Node<T> root) {
            if (null == root) {
                return;
            }
    
            Stack<Node<T>> stack = new Stack<Node<T>>();
            Node<T> currNode = root;
            Node<T> lastPopNode = null;
    
            while(currNode != null || !stack.isEmpty()) {
                if (null != currNode) {
                    stack.push(currNode);
                    currNode = currNode.left;
                } else {
                    if (stack.peek().right == null || lastPopNode == stack.peek().right) {
                        lastPopNode = stack.pop();
                        System.out.println(lastPopNode.value);
                    } else {
                        currNode = stack.peek().right;
                    }
                }
            }
        }
---
##### 1.4 层序遍历/宽度优先遍历

思路：  

1. root 节点并入队列
2. 如果队列不为空，则出队， 出队节点为 node， 将 node 的左右孩子（如果存在的话）入队。然后重复 2)。

终止条件为队列为空。实现代码如下：  
	   
        public static <T> void levelOrderTraversal(Node<T> root) {
            if (null == root) {
                return;
            }
            Queue<Node<T>> queue = new SimpleQueue<Node<T>>();
            queue.add(root);

            while(!queue.isEmpty()) {
                Node<T> node = queue.poll();
                System.out.println(node.value);
                if (null != node.left) {
                    queue.offer(node.left);
                }
                if (null != node.right) {
                    queue.offer(node.right);
                }
            }
        }
        
---
#### 2. 简单递推求值系列

##### 2.1 求第 K 层的节点的个数

递推公式： ``` f(node, K) = f(node.left, K - 1) + f(node.right, K - 1) ```，非递归可以采用层序遍历的方式统计第 K 层的个数。递归实现代码如下：
	
	     public static <T> int nodeCountOfKLevel(Node<T> root, int k) {
            if (null == root) {
                return 0;
            }
    
            if (0 == k) {
                return 0;
            }
    
            if (1 == k) {
                return 1;
            }
    
            return nodeCountOfKLevel(root.left, k - 1) + nodeCountOfKLevel(root.right, k - 1);
        }
---
##### 2.2 求二叉树中叶子节点的个数 
递推公式： ``` f(node) = (isLeaf(node) ? 1 : (f(node.left) + f(node.right)) ```，或者采用任意一种遍历算法，发现叶子节点统计加 1 即可。递归版本的实现如下：

	
        public static <T> int countOfLeaf(Node<T> root) {
            if (null == root) {
                return 0;
            }
    
            // leaf node
            if (null == root.left && null == root.right) {
                return 1;
            }
    
            return countOfLeaf(root.left) + countOfLeaf(root.right);
        }
---
##### 2.3 判断二叉树是否是平衡二叉树
递推公式: ``` f(node) = f(node.left).isAVL && f(node.right).isAVL && abs(f(node.left).height - f(node.right).height) < 1  ``` , 其中 f(node) 有两个返回值: 1. 是否是平衡二叉树 isAVL; 2.二叉树的高度 height。实现代码如下：  

        public static <T> Pair<Boolean, Integer> isAVLAndHeight(Node<T> root) {
            if (null == root) {
                return new Pair<Boolean, Integer>(true, 0);
            }
            Pair<Boolean, Integer> isLeftAVLAndHeight = isAVLAndHeight(root.left);
            Pair<Boolean, Integer> isRightAVLAndHeight = isAVLAndHeight(root.right);
    
            return new Pair<Boolean, Integer>(isLeftAVLAndHeight.fst && isRightAVLAndHeight.fst
                    && (Math.abs(isLeftAVLAndHeight.snd - isRightAVLAndHeight.snd) < 1),
                    Math.max(isLeftAVLAndHeight.snd, isRightAVLAndHeight.snd) + 1);
        }
	
---

#### 3. 路径系列
##### 3.1 最长的路径问题
利用求解二叉树高度的程序， 首先求解通过某个指定节点 node 的最长路径， 即为： ``` maxLength = depth(node.left) + depth(node.right) + 1```。 整个二叉树的最长路径， 即为所有节点中通过某个节点的最长路径。实现代码如下：  

	    public static Integer maxPathLength = 0;
        public static <T> int depth(Node<T> root) {
            if (null == root) {
                return 0;
            }
            int heightOfLeft = depth(root.left);
            int heightOfRight = depth(root.right);
            maxPathLength = Math.max(heightOfLeft + heightOfRight + 1, maxPathLength);
            return Math.max(heightOfLeft, heightOfRight) + 1;
        }
---
##### 3.2 求节点数值和为给定值的所有路径

#### 4. 完全二叉树系列
##### 4.1 判断二叉树是否是完全二叉树
##### 4.2 完全二叉树的插入问题
#### 5. 其他
##### 5.1 二叉树转双向链表
##### 5.2 根据前序遍历和中序遍历重建二叉树
##### 
