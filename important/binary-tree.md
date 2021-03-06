---
title: 二叉树基础以及常用操作
date: 2016-10-27 13:47:14
tags: 
- 数据结构
- 二叉树
- 算法
categories: 算法与数据结构
---

#### 常见基本概念

- 节点的度：某节点的度定义为该节点孩子节点的个数。
- 叶子节点：度为0 的结点称为叶结点，或者称为终端结点。
- 树的度：树中各结点度的最大值。
- 节点的高度：从该节点起到叶子节点的最长简单路径的边数。(简单路径：无重复边的路径)
- 树的高度：根节点的高度。
- 节点的层数：规定树的根结点的层数为1，其余结点的层数等于它的双亲结点的层数加1。
- 节点的深度：即该节点的层数。
- 树的层数：树中所有节点的最大层数。
- 树的深度：树中所有结点的最大层数。
- 外节点：叶子节点。
- 内节点：除叶子节点之外的节点。
- 满二叉树：所有终端都在同一层次，且非终端结点的度数为2。在满二叉树中若其深度为k，则其所包含的结点数必为2^k-1。
- 完全二叉树：除最后一层，每一层的节点数都达到最大。最后一层若是没满，则节点集中在左边，空的只能是右边。对于完全二叉树，设一个结点为i则其父节点为i/2，2i为左子节点，2i+1为右子节点。
- 扩充二叉树：对二叉树中度为1的节点和叶子节点添加空节点，使之成为满二叉树。
二叉树的每个结点最多有两棵子树，左子树和右子树，次序不可以颠倒。


与树不同，树的节点个数至少为1，而二叉树的节点个数可以为0；树中节点的最大度数没有限制，而二叉树节点的最大度数为2；树的节点无左、右之分，而二叉树的节点有左、右之分。

注：关于树深度、层数、高度的定义会有不同的说法：有从0开始计数的，也有从1开始计数的。从哪儿开始计数不是关键，关键在于一致性，量的写法要一致。


#### 存储方式

##### 顺序存储表示
二叉树可以用数组或链接串列来存储，而且如果这是满二叉树，这种方法不会浪费空间。用这种紧凑排列，如果一个节点的索引为i，它的子节点能在索引2i+1和2i+2找到，并且它的父节点（如果有）能在索引floor((i-1)/2)找到（假设根节点的索引为0）。这种方法更有利于紧凑存储和更好的访问的局部性，特别是在[前序遍历]中。然而，它需要连续的存储空间，这样在存储高度为h的n个结点组成的一般普通树时将会浪费很多空间。一种最极坏的情况下如果深度为h的二叉树每个节点只有右孩子需要占用2的h次幂减1，而实际却只有h个结点，空间的浪费太大，这是顺序存储结构的一大缺点。

![](/images/data-structure/binary-tree-0.png)

```c
#define MAX_TREE_SIZE 100 /* 二叉树的最大节点数 */
 typedef TElemType SqBiTree[MAX_TREE_SIZE]; /* 顺序存储的数组 */

 typedef struct
 {
   int level,order; /* 节点所在层以及在该层的序号 */
 }position;
```

##### 二叉链表存储表示
在使用记录或内存地址指针的程序设计语言中，二叉树通常用树结点结构来存储。有时也包含指向唯一的父节点的指针。如果一个结点的子结点个数小于2，一些子结点指针可能为空值，或者为特殊的哨兵结点。 使用链表能避免顺序存储浪费空间的问题，算法和结构相对简单，但使用二叉链表，由于缺乏父链的指引，在找回父节点时需要重新扫描树得知父节点的节点地址。

```c
/* 二叉树的二叉链表存储表示 */
 typedef struct BiTNode
 {
   TElemType data;
   struct BiTNode *lchild,*rchild; /* 左右孩子指針 */
 }BiTNode,*BiTree;
```

##### 三叉链表存储表示

改进于二叉链表，增加父节点的指引，能更好地实现节点间的访问，不过算法相对复杂。 当二叉树用三叉链表表示时，有N个结点，就会有N+2个空指针。

```c
/* 二叉树的三叉链表存储表示 */
 typedef struct BiTPNode
 {
   TElemType data;
   struct BiTPNode *parent,*lchild,*rchild; /* 父、左右孩子指針 */
 }BiTPNode,*BiPTree;
```

#### 创建二叉树
本文使用二叉树的二叉链表存储表示：

```c
#define SIZE 100

typedef char dataType;

struct treeNode{
    dataType val;
    struct treeNode * leftChild;
    struct treeNode * rightChild;
    int visit = 0;//后序遍历时使用
};
typedef struct treeNode * treeNodeP;
```
创建如下这样一颗二叉树：
![](/images/data-structure/binary-tree-1.png)

##### 括号表示法创建
对应的括号表示法为：`A(B(D(H),E(I,J)),C(F,G))`,括号表示法中，对应`()`中的内容为`(`前节点的孩子，`()`中的`,`用于分隔左右孩子。


```c
void create(treeNodeP &root,char* str){
    root = NULL;
    struct treeStack stack;
    (&stack)->top = -1;
    int pos = 0;//字符串遍历时的当前位置
    char ch = str[pos];
    treeNodeP node = NULL;
    int type= 0;
    while(ch != '\0'){
        switch (ch) {
            case '('://遇到'('时，表示下一个节点是上一个节点的左孩子。
                push(&stack, node);//此时node保存着上一个节点，将上一个节点入栈。
                ch = str[++pos];//遍历下一个字符
                type = 1;//表示下一个元素是左孩子
                break;
            case ',':
                type = 2;//表示下一个孩子是右孩子
                ch = str[++pos];//遍历下一个字符
                break;
            case ')'://遇到`)`时，表示栈顶元素的孩子输入完毕了，可以将栈顶节点出栈了。
                pop(&stack);//栈顶节点出栈
                ch = str[++pos];//遍历下一个字符
                break;
            default://表示遇到的是节点的内容值
                node = new treeNode();//新建节点并初始化
                node->val = ch;
                node->leftChild = NULL;
                node->rightChild = NULL;
                if(root == NULL){//如果root节点为NULL,则当前输入的节点为根节点
                    root = node;
                }
                else if((&stack)->top > -1){//栈不为空
                    treeNodeP t = top(&stack);//获取栈顶节点（不弹出）
                    if(type == 1){
                        t->leftChild = node;
                    }
                    else if(type == 2){
                        t->rightChild = node;
                    }
                }
                ch = str[++pos];
                break;
        }
    }
}
```

#####  按前序遍历输入
表示为：`ABDH###EI##J##CF##G##`，此表示法按照`前序遍历`的顺序输入，`#`表示`NULL`。

核心思想：如果字符连续的出现，则下一个字符为上一个字符的左孩子，在`#`出现之前，所有的字符按照顺序都入栈。如果字符在`#`后第一次出现，则必为栈顶节点的右节点。当出现第一个`#`时，说明上一个字符为当前分支的最左节点，上一个字符的左孩子赋值NULL。连续出现第二个`#`时，说第一个`#`前的字符的右孩子也赋值NULL。当一个节点的右孩子赋值完成时，便可以从栈中弹出了。如果连续出现2个以上的`#`,则栈顶的节点的右孩子依次置NULL，并弹出。

```c
void createByPreorder(treeNodeP &root, char* str){
    root = NULL;
    int pos = 0;
    char ch = str[pos];
    treeNodeP node = NULL;
    struct treeStack stack;
    (&stack)->top = -1;
    int count = 0;//记录`#`连续出现的记录
    while(ch != '\0'){
        if(ch != '#'){
            count = 0;//一旦`#`终止连续出现，count赋0
            node = new treeNode();//新建节点并初始化
            node->val = ch;
            node->leftChild = NULL;
            node->rightChild = NULL;
            if(root == NULL){//如果root节点为NULL,则当前输入的节点为根节
                root = node;
            }
            if((&stack)->top > -1){//栈不为空
            //遇见一个字符，既然栈中存在节点，则不为左节点便为右节点
                if(top(&stack)->leftChild){
                    pop(&stack)->rightChild = node;//如果是右节点，则赋值完整后便可以将节点弹出栈
                }
                else{
                    top(&stack)->leftChild = node;
                }   
            }        
            push(&stack, node);//节点入栈
            
        }
        else{
            count++;
            if((&stack)->top > -1){
                if(count == 1){//`#`第一次出现时，必然是栈顶节点的左孩子
                    top(&stack)->leftChild = NULL;
                }
                if(count == 2){//`#`第二次连续出现时，必然是栈顶节点的右孩子
                    pop(&stack)->rightChild = NULL;//此时栈顶节点已完成左右孩子的赋值，可以弹出。
                }
                if(count > 2){//将栈中的元素弹出并将右孩子置NULL
                    pop(&stack)->rightChild = NULL;
                }
                
            }
        }
        ch = str[++pos];//遍历下一个字符
    }
}
```

#### 非递归的前序遍历

```c
void preorder(treeNodeP root){
    if(!root){
        printf("the tree is empty!");
    }
    struct treeStack stack;
    (&stack)->top = -1;
    treeNodeP tmp = root;
    //只有栈空（所有节点的右孩子都已开始访问或已访问完成）或者
    //节点为NULL（栈空时，当访问到最右孩子时tmp为NULL）遍历才算完成
    while (tmp || (&stack)->top > -1) {
        while(tmp){//此循环的目的是遍历到当前分支的最左节点，将遍历的节点依次输出并入栈。
            printf("%c",tmp->val);
            push(&stack, tmp);//栈中的所有节点的右孩子都没有开始访问
            tmp = tmp->leftChild;
        }
        //上面的循环结束表示当前分支的所有左孩子都入栈了，此时弹出最后一个左孩子节点。
        tmp = pop(&stack);
        tmp = tmp->rightChild;//这里考虑的逻辑就是右孩子是否为NULL，
        //若为NULL，则弹出栈顶节点，弹出节点为此节点的父节点，此节点后的内容遍历完成，开始父节点的右孩子的前序遍历。
        //若不为NULL，开始上面的循环，遍历到此节点右孩子分支的最左节点。
    }
    
}

```
输出结果为：`ABDHEIJCFG`。

#### 非递归的中序遍历

```c
void midorder(treeNodeP root){
    if(!root){
        printf("the tree is empty!");
    }
    struct treeStack stack;
    (&stack)->top = -1;
    treeNodeP tmp = root;
    while (tmp || (&stack)->top > -1) {
        while(tmp){
            push(&stack, tmp);
            tmp = tmp->leftChild;
        }
        //来到当前分支的最左，弹出最左节点，输出。
        tmp = pop(&stack);
        printf("%c",tmp->val);
        tmp = tmp->rightChild;//这里考虑的逻辑就是右孩子是否为NULL，
        //若为NULL，则弹出栈顶节点，弹出节点为此节点的父节点，并输出，此节点后的内容遍历完成，开始父节点的右孩子的遍历。
        //若不为NULL，则再次上面的循环，开始当前节点右孩子的遍历。
    }

}
```
与前序遍历唯一的不同就是输出字符的位置变了。前序遍历先是父节点，而中序遍历则是左孩子先。两个输出位置可以细细体味一下。

输出结果为：`HDBIEJAFCG`。

#### 非递归的后序遍历
考虑到后序遍历时父节点需要被回溯2次，所以用一个int型的visit变量加以区分，初始值为0。

```c
void postorder(treeNodeP root){
    if(!root){
        printf("the tree is empty!");
    }
    struct treeStack stack;
    (&stack)->top = -1;
    treeNodeP tmp = root;
    while (tmp || (&stack)->top > -1) {
        while(tmp){
            if(tmp->visit == 1){//入栈次数为1，表示还未访问过右孩子，需要先访问右孩子，此节点入栈，入栈次数+1，同时切换到右孩子分支。
                push(&stack, tmp);
                tmp->visit++;
                tmp = tmp->rightChild;
                goto right;
            }
            else if(tmp->visit == 2){//入栈次数为2，表示右孩子以访问完成，可以输出。输出完毕后从栈中取下一个访问节点，也就是此节点的父节点，开始内循环。
                printf("%c",tmp->val);
                tmp = pop(&stack);
                goto right;
            }
            push(&stack, tmp);
            tmp->visit = 1;//第一次入栈
            tmp = tmp->leftChild;
        }
        tmp = pop(&stack);
        if(tmp->rightChild){//如果最左节点有右孩子，因为后序遍历时右孩子前于父节点，所以右孩子入栈。
            push(&stack, tmp);//此节点需再次入栈
            tmp->visit++;//入栈次数+1
            tmp = tmp->rightChild;//切换到右孩子
            continue;//继续内循环，遍历到右孩子的最左节点
        }
        //节点无右孩子，则可以输出。
        printf("%c",tmp->val);
        tmp = pop(&stack);
        
    right:;
    }

}
```
因为visit一开始都为0，所以内循环中直接遍历到最左节点，并将节点依次入栈。

输出结果为：`HDIJEBFGCA`。

#### 层次遍历
层次遍历的思想即使将当前节点的孩子依次从左到右入队列，此节点访问后便从队列中取下一个节点。重复以上过程。
```c
void levelTraverse(treeNodeP root){
    if(!root){
        printf("the tree is empty!");
    }
    struct treeQueue queue;
    (&queue)->front = -1;
    (&queue)->rear = -1;
    treeNodeP tmp = root;
    while(tmp){
        printf("%c",tmp->val);
        if(tmp->leftChild){
            enterNode(&queue, tmp->leftChild);
        }
        if(tmp->rightChild){
            enterNode(&queue, tmp->rightChild);
        }
        tmp = getNode(&queue);
    }
}
```
输出结果为：`ABCDEFGHIJ`。
#### 查找节点

```c
treeNodeP findNode(treeNodeP root,char ch){
    if(!root){
        return NULL;
    }
    if(root->val == ch){
        return root;
    }
    else{
        treeNodeP node = findNode(root->leftChild, ch);
        if(node){
            return node;
        }
        else{
            node = findNode(root->rightChild, ch);
            if(node){
                return node;
            }
        }
    }
    return NULL;
}
```

#### 求高度

```c
int getHigh(treeNodeP root){
    if(!root){
        return 0;
    }
    int h1 = getHigh(root->leftChild);
    int h2 = getHigh(root->rightChild);
    return h1 > h2 ? h1 + 1 : h2 +1;
}
```
#### 所使用的栈、队列数据结构

```c
struct treeStack{
    treeNodeP node[SIZE];
    int top;
    
};
typedef struct treeStack * treeStackP;

void push(treeStackP stack, treeNodeP node){
    if(stack->top < SIZE - 1){
        stack->node[++stack->top] = node;
    }
    else{
        printf("the stack is full!\n");
    }
    
}

treeNodeP pop(treeStackP stack){
    if(stack->top > -1){
        return stack->node[stack->top--];
    }
    else{
        return NULL;
    }
}

treeNodeP top(treeStackP stack){
    if(stack->top > -1){
        return stack->node[stack->top];
    }
    else{
        return NULL;
    }
}

struct treeQueue {
    treeNodeP node[SIZE];
    int front;
    int rear;
};

typedef struct treeQueue * treeQueueP;

treeNodeP getNode(treeQueueP queue){
    if(queue->front == queue->rear){
        return NULL;
    }
    queue->front = (queue->front + 1) % SIZE;
    return queue->node[queue->front];
}

void enterNode(treeQueueP queue, treeNodeP node){
    
    if((queue->rear + 1) % SIZE == queue->front){
        printf("the queue is full!");
    }
    queue->rear = (queue->rear + 1)% SIZE;
    queue->node[queue->rear] = node;
    
}
```

><font color= Darkorange>因本人水平有限，若文章内容存在问题，恳请指出。允许转载，转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 
