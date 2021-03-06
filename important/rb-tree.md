---
title: 红黑树介绍及实现
date: 2016-10-30 12:07:38
tags: 
- 数据结构
- 红黑树
- 二叉树
- 查找
- 算法
categories: 算法与数据结构
---
#### 自平衡二叉查找树（AVL）
了解自平衡二叉查找树前，先讲讲二叉查找树。二叉查找树是一颗二叉树，对于其中的每一个节点，左孩子小于自己，右孩子大于自己。在插入一个节点时，判断插入节点与当前遍历的树节点的大小，若小于当前树节点，则与左孩子比较，反之，则与右孩子比较。
![](/images/data-structure/rb-tree-0.png)
但是，一颗普通的二叉查找树容易退化成链表：
![](/images/data-structure/rb-tree-1.png)
我们希望一颗二叉查找树的时间复杂度为O（logn）,而这种情况下时间复杂度已经变成了O（n）。所以就出现了`自平衡二叉查找树`。

在AVL树中任何节点的两个子树的高度最大差别为一，所以它也被称为高度平衡树。查找、插入和删除在平均和最坏情况下都是O（log n）。增加和删除可能需要通过一次或多次树旋转来重新平衡这个树。

节点的平衡因子是它的左子树的高度减去它的右子树的高度（有时相反）。带有平衡因子1、0或 -1的节点被认为是平衡的。带有平衡因子 -2或2的节点被认为是不平衡的，并需要重新平衡这个树。平衡因子可以直接存储在每个节点中，或从可能存储在节点中的子树高度计算出来。

当失去平衡时，就需要通过所谓的`AVL旋转`来恢复平衡。以下图表以四列表示四种情况，每行表示在该种情况下要进行的操作。在左左和右右的情况下，只需要进行一次旋转操作；在左右和右左的情况下，需要进行两次旋转操作。
![](/images/data-structure/rb-tree-2.png)

#### 红黑树
AVL有几种不同的实现方式，红黑树就是其中一种，其他常见的还有树堆（Treap,树和堆的合并写法）、伸展树（Splay）等。它们之间的具体区别有兴趣的同学可以Google下，`总的来说红黑树是功能、性能、空间开销的折中结果`。以下比较来自知乎：

>AVL更平衡，结构上更加直观，时间效能针对读取而言更高；维护稍慢，空间开销较大。

>红黑树，读取略逊于AVL，维护强于AVL，空间开销与AVL类似，内容极多时略优于AVL，维护优于AVL。

>Splay Tree，读取维护速度不稳定，属于理论上的统计logN时间复杂度，特殊情况下能直接退化为链表，空间最节省，部分功能优于前两者。

>Treap，速度快，功能有缺失。
>
>作者：Coldwings
链接：https://www.zhihu.com/question/20545708/answer/44370878
来源：知乎
著作权归作者所有，转载请联系作者获得授权。

红黑树是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求，每次改动之后都必须保证不破坏这些性质，或者破坏之后恢复：

1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子都是黑色（叶子是NIL节点）。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

下面是一个具体的红黑树的图例：
![](/images/data-structure/rb-tree-3.png)
"nil叶子"或"空（null）叶子"不包含数据而只充当树在此结束的指示。这些节点在绘图中经常被省略，导致了这些树好像同上述原则相矛盾，而实际上不是这样。

因为每一个红黑树也是一个特化的二叉查找树，因此红黑树上的只读操作与普通二叉查找树上的只读操作相同。然而，在红黑树上进行插入操作和删除操作会导致不再匹配红黑树的性质。恢复红黑树的性质需要少量（O(log n)）的颜色变更（实际是非常快速的）和不超过三次树旋转（对于插入操作是两次）。虽然插入和删除很复杂，但操作时间仍可以保持为O(log n)次。

#### 插入
红黑树作为一颗特殊的二叉树，在插入的第一阶段和普通二叉树一样，先不考虑是否破坏红黑树的性质，而是找到一个合适的插入位置，这个位置处于树分支的最底下的某个节点。

注：未出现的数据结构和函数可在附录找到。

```c
//参数z为需插入的节点
void insert(treeNodeP& root,treeNodeP z){
    if(!root){
        root = z;
        root->color = BLACK;
        return;
    }
    treeNodeP y = NIL;
    treeNodeP x = root;
    while(x != NIL){
        y = x;
        if(z->val < x->val){//插入节点的值小于当前比较的树节点
            x = x->leftChild;//切换到左孩子
        }
        else{
            x = x->rightChild;
        }
    }
    z->parent = y;//y就是插入节点的父节点，y是没有左右孩子的
    if(y == NIL){
        root = z;
    }
    else if(z->val < y->val){//判断插入节点应该是左孩子还是右孩子
        y->leftChild = z;
    }
    else{
        y->rightChild = z;
    }
    insertFixup(root,z);//此函数用于在插入新节点后恢复红黑树的性质
}
```

所有的新插入节点都是红色的，这样的话不会破坏性质5。插入节点是红色的，如果父节点是黑色的，红黑树的任何性质都不会被破坏，所以在插入时需要恢复的所有情况下，插入节点的父节点都是红色的。因为红色节点不会连续，所以祖父节点必定是黑色的。**把叔叔节点为红色分为情况一;叔叔节点是黑色，而自己为父节点的右孩子分为情况二;叔叔节点是黑色，自己为父节点的左孩子分为情况三。**以上三种情况针对插入节点属于祖父节点的左分支，右分支的情况类似。`insertFixup `函数会将情况一和情况二转换（构造）为情况三，然后情况三通过一个旋转操作后就可以恢复红黑树的性质。下图中深色的表示黑色，z为新插入节点，y为z的叔叔。
![](/images/data-structure/rb-tree-4.png)
在上图的情况一中，如果2号节点是黑色的，那么只要一遍染色就恢复了红黑树性质，无需转换为情况二、三了。最后的(d)图就是恢复红黑树性质的树。

调用`insertFixup `函数来恢复红黑树的性质，参数`z`为新插入的节点,`z`在插入前`需要已被初始化为红色`(初始化函数查看附录)。

```c
void insertFixup(treeNodeP& root, treeNodeP z){
//如果插入节点z的父节点是黑色的，那么插入的红色节点不破坏任何红黑树的性质
    while (z->parent && z->parent->color == RED) {
    //插入节点z处于祖父的左分支
        if(z->parent == z->parent->parent->leftChild){
	        //y是插入节点z的叔叔
            treeNodeP y = z->parent->parent->rightChild;
			 //情况一
            if(y->color == RED){
                y->color = BLACK;
                z->parent->color = BLACK;
                z->parent->parent->color = RED;
                z = z->parent->parent;
            }//情况二
            else if(z == z->parent->rightChild){
                z = z->parent;
                leftRotate(root, z);
            }//情况三
            else{
                z->parent->color = BLACK;
                z->parent->parent->color = RED;
                rightRoate(root, z->parent->parent);

            }
        }
        else{//插入节点z处于祖父节点的右分支，和上面的代码左右互反即可
            treeNodeP y = z->parent->parent->leftChild;
            if(y->color == RED){
                y->color = BLACK;
                z->parent->color = BLACK;
                z->parent->parent->color = RED;
                z = z->parent->parent;
            }
            else if(z == z->parent->leftChild){
                z = z->parent;
                rightRoate(root, z);
            }
            else{
                z->parent->color = BLACK;
                z->parent->parent->color = RED;
                leftRotate(root, z->parent->parent);
            }
            
        }
        root->color = BLACK;
    }
}
```



#### 删除
删除一个节点同样可能破坏红黑树的性质，然而和插入的时候一样，在删除的第一阶段，同样先不考虑颜色，先按照一颗普通二叉树将节点正常删除，然后再恢复。普通二叉查找树在几种情况下的删除示意图如下（z为需删除的节点）：
![](/images/data-structure/rb-tree-5.png)
上面的图里面没有`z`的两个孩子都是NIL的情况，因为这种情况其实可以合并到第一种情况中，最后的结果是一样的。当`z`的左右孩子都不为NIL时，需要找到z的右孩子分支中的最左（小）节点，用于替换`z`（貌似左孩子的最右（大）节点也是可以的）。图（c）和图(d)中的`y`就是`z`的右孩子的最左（小）节点，`z`被删除后`y`就用于顶替`z`的位置,**同时`y`被染上与`z`同样的颜色（这个染色只有在（c）、(d)的情况下）**。

`del `函数就是按照上面的示意图实现的，代码如下：

```c
void del(treeNodeP& root,treeNodeP z){
    bool originalColor = z->color;//（a）、(b)情况下的删除节点的颜色
    treeNodeP x = NULL;
    if(z->leftChild == NIL){//z的左节点为NIL，用右节点替换z
        x = z->rightChild;
        translate(root, z, x);
    }
    else if(z->rightChild == NIL){//z的右节点为NIL，用左节点替换z
        x = z->leftChild;
        translate(root, z, x);
    }
    else{
        treeNodeP y = treeMiniMum(z->rightChild);//y为z的后继节点（z删除后用y替换）
        originalColor = y->color;//因为y要用来替换z，所以需要记住y的原来颜色
        x = y->rightChild;//x为y移走后替换y的节点
        
        if(y->parent == z){
            x->parent = y;//如果x==NIL时有用
        }
        else{
            translate(root, y, x);//x替换y
            y->rightChild = z->rightChild;//y替换z（和z的右孩子建立联系）
            y->rightChild->parent = y;//y替换z（和z的右孩子建立联系）
        }
        translate(root, z, y);//y替换z（和z的父节点建立联系）
        y->leftChild = z->leftChild;//y替换z（和z的左孩子建立联系）
        y->leftChild->parent = y;//y替换z（和z的左孩子建立联系）
        y->color = z->color;//此时y已成功替换了z,将z的颜色赋予y
    }
    if(originalColor == BLACK){//原因在下文给出
        delFixup(root, x);//删除完成后修复红黑树的性质
    }
}
```

对于（a）、（b）来说，删除节点至少有一个是NIL节点，`如果删除节点是红色的`，则性质5没有被破坏，其他性质也一样，无需恢复。为什么性质5没有被破坏呢？因为对于其中一个NIL节点来说，在删除节点后就没有黑色节点了，所以在删除节点的另一个分支上，也不会出现任何黑色节点，用于替换删除节点的那个节点只可能是红色的。`如果删除节点是黑色的`,那么删除节点的2个分支的性质5都被破坏了，此时需要恢复。

对于（c）、(d)来说，因为删除节点被替换的时候包括颜色，所以删除节点处的性质都能保持，重点就是替换节点了 。`如果替换节点是红色的`，那么替换节点处的性质也能保持。`如果替换节点是黑色的`，那么替换节点的分支处的红黑树性质就会被破坏，最明显的就是性质5，因为少了个黑色节点。

**综上，对于（a)(b),在删除节点是黑色时，性质被破坏。对于(c)(d)，替换节点是黑色时，性质被破坏。**

现在来看下如何恢复。

对于(a)(b)的情况，如果替换的节点是红色的，那么直接将该节点的颜色染成黑色便可以了。如果替换的节点是NIL（这种情况是2个子节点都是NIL），那么将这种情况分到(c)(d)中去。

对于(c)(d)的情况，如果替换y节点的x节点是红色的，同样直接将颜色染成黑色就可以了(注意：x为替换y的那个节点，y替换了删除节点z)。如果x节点是黑色的，可分为如下4种情况，这4种情况针对x为左孩子，右孩子的情况类似(黑色最深)：
![](/images/data-structure/rb-tree-6.png)

- 情况一：x的兄弟节点w是红色的。
- 情况二：x的兄弟节点w是黑色的，w的两个孩子也是黑色的。
- 情况三：x的兄弟节点w是黑色的，w的左孩子是红色的，右孩子是黑色的。
- 情况四：x的兄弟节点w是黑色的，w的右孩子是红色的。

因为少了一个黑色节点，x在某种意义上就是`双重黑色`了，但要想办法把这个黑色抽出来。上面4种情况的最终目标是转换为情况四，在情况四中可以通过旋转和染色，把失去的黑色给重新补回来。所以恢复时，将黑色不断地往上移，过程中不能破坏原有的性质，当黑色移动到了根节点，或者构造出了情况四，就可以恢复红黑树性质了。

`delFixup`函数实现如下(**这个实现是有一点问题的，因为树的每一个NIL节点都是独立的，而我在实现的时候只有一个NIL对象，所有需要用的地方都使用了这一个。当在恢复红黑树性质的过程中涉及到了多个节点的左右孩子（孩子又是NIL节点）时，因为无法区分，所以会出现问题。但是这个NIL节点在上面的插入函数中不影响结果。在这个函数当中的NIL节点你可以把它当做是独立的来看待（虽然不是独立的^-^）。**):

```c
void delFixup(treeNodeP& root,treeNodeP x){
   
    treeNodeP w = NULL;
    while(x != root && x->color == BLACK){
    //x是左孩子
        if(x == x->parent->leftChild){
            w = x->parent->rightChild;//w是x的兄弟节点
            //情况一
            if(w->color == RED){
                w->color = BLACK;
                x->parent->color= RED;//w为x的兄弟，w为RED，则父节点之前不可能为RED
                leftRotate(root, x->parent);
                w = x->parent->rightChild;
                
            }//情况二
            if(w->leftChild->color == BLACK && w->rightChild->color == BLACK){
                w->color = RED;//抽去黑色，因为x有双重黑色，所以x依旧是黑色
                x = x->parent;//在循环最后给新的x节点染上抽出的黑色,使黑色上移
            }//情况三
            else if(w->rightChild->color == BLACK){
                w->leftChild->color = BLACK;
                w->color = RED;
                rightRoate(root, w);
               
            }//情况四
            else{
                w->color = x->parent->color;
                x->parent->color = BLACK;
                w->rightChild->color = BLACK;
                leftRotate(root, x->parent);
                x = root;//到了情况四就可以通过这一步结束循环了
            }
        }//x是右孩子，代码和上面的左右互反即可
        else{
            w = x->parent->leftChild;
            if(w->color == RED){
                w->color = BLACK;
                x->parent->color = RED;
                rightRoate(root, x->parent);
                x = x->parent;
                w = x->parent->leftChild;
            }
            if(w->leftChild->color == BLACK && w->rightChild->color == BLACK){
                w->color = RED;
                x = x->parent;
            }
            else if(w->leftChild->color == BLACK){
                w->rightChild->color = BLACK;
                w->color = RED;
                leftRotate(root, w);
            
            }
            else{
                w->color = x->parent->color;
                x->parent->color = BLACK;
                w->leftChild->color = BLACK;
                rightRoate(root, x->parent);
                x = root;
            }
        }
        x->color = BLACK;
    }
}
```

#### 附录

##### 树节点数据结构

```c
bool BLACK = true;
bool RED = false;
struct treeNode{
    int val;
    bool color;
    struct treeNode * parent;
    struct treeNode * leftChild;
    struct treeNode * rightChild;
};
typedef struct treeNode * treeNodeP;
```
##### NIL节点表示
```c
struct treeNode node = {-1,BLACK,NULL,NULL,NULL};
treeNodeP NIL = &node;
```
##### 初始化节点
```c
void init(treeNodeP node,int val,bool color){
    node->parent = NIL;
    node->leftChild = NIL;
    node->rightChild = NIL;
    node->val = val;
    node->color = color;
}
```
##### 左旋右旋代码
左旋和右旋都是有规律的，可以按照上面的示意图对照理解。

```c
//左旋，x为旋转的顶点
void leftRotate(treeNodeP& root,treeNodeP x){
    treeNodeP y = x->rightChild;
    x->rightChild = y->leftChild;
    if(y->leftChild != NIL){//若不判断，有可能空指针异常
        y->leftChild->parent = x;
    }
    y->parent = x->parent;
    if(x->parent == NIL){//根节点的父节点为NIL
        root = y;
    }
    else if(x == x->parent->leftChild){
        x->parent->leftChild = y;
    }
    else{
        x->parent->rightChild = y;
    }
    y->leftChild = x;
    x->parent = y;
    
}
//右旋，x为旋转的顶点
void rightRoate(treeNodeP& root, treeNodeP x){
    treeNodeP y = x->leftChild;
    x->leftChild = y->rightChild;
    if(y->rightChild != NIL){
        y->rightChild->parent = x;
    }
    y->parent = x->parent;
    if(x->parent == NIL){
        root = y;
    }
    else if(x == x->parent->leftChild){
        x->parent->leftChild = y;
    }
    else{
        x->parent->rightChild = y;
    }
    y->rightChild = x;
    x->parent = y;
}
```

##### translate函数
```c
//v替换u（仅仅是v和u的父节点建立联系）
void translate(treeNodeP& root, treeNodeP u, treeNodeP v){
    if(u->parent == NIL){
        root = v;
    }
    else if(u == u->parent->leftChild){
        u->parent->leftChild = v;
    }
    else{
        u->parent->rightChild = v;
    }
    v->parent = u->parent;
}

```
#####  treeMiniMum函数

```c
//找到节点x的右子树种的最左（小）节点
treeNodeP treeMiniMum(treeNodeP x){
    while (x->leftChild != NIL) {
        x = x->leftChild;
    }
    return x;
}
```

参考：

- 《算法导论》
- 维基百科

><font color= Darkorange>因本人水平有限，若文章内容存在问题，恳请指出。允许转载，转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 
