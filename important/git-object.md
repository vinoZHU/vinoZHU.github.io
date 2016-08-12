---
title: 理解git对象
date: 2016-05-16 18:51:22
tags: Git
categories: Git

---
##### 首次提交，提交一个简单的文件 a.txt ，commit 之后的图如下：
 ![](/images/git/git-object-0.png)
 如图所示，生成了 3 个对象，一个 commit 对象，一个 tree 对象，一个 blob 对象。图上蓝底是 commit 对象，灰底的是 tree 对象，白底的是 blob 对象，每个对象节点的标题是对象的 key (SHA 摘要)缩略表示。
 对于 commit 对象，tree 内容表示这个 commit 对应根目录的 tree 对象，parent 表示父 commit 节点，通常commit 只有一个父节点，也可能没有（首次提交时 parent 为空），也可能有多个（合并节点），commit 对象还保存了 commit message 等信息。
 对于 tree 对象，里面的内容包含了文件名，文件对应的 blob 对象的 key，或者是目录名和目录对应 tree 对象的 key。
 对于 blob 对象，表示一个实际文件对象的内容，但不包括文件名，文件名是在 tree 对象里存的。

 这个图怎么得到的呢？主要是两个命令：
 - 通过 git log 命令获取最新 commit 的 key
 - 通过 git cat-file -p <object key> 获取 key 对应 object 的内容，根据 object 里的内容，继续探索，就可以访问到所有关联 object

##### 第 2 次提交，修改了 a.txt 文件：
![](/images/git/git-object-1.png)
因为 a.txt 文件已经修改，生成了一个新的 blob 对象，tree 对象和 commit 对象。如图所示，commit 对象之间是有关联的，新提交的 commit 对象的 parent 是上一次提交的 commit 对象。

##### 第 3 次提交，这次已稍微复杂一点，增加一个新文件 b.txt ，一个新目录 lib ，lib 里增加一个文件 c.txt
![](/images/git/git-object-2.png)
如图所示，目录是有一个 tree 对象表示的，里面的内容指明了目录包含的文件或子目录。

##### 第 4 次提交，这次弄出一个新的分支 test1，并且在新分支中做了一次 commit
![](/images/git/git-object-3.png)
0c5ca 对应的 commit 对象就是生成的分支 test1 中的。分支在 Git 中是一个非常轻量化的操作，建立分支甚至都不增加新的对象。

##### 第 5 次提交，这次涉及到一个合并操作，图已经变得比较复杂了
![](/images/git/git-object-4.png)
def18 就是合并后的 commit 对象。合并生成了一个新的commit ，这个 commit 的 parent 有两个，指向合并的两个原分支对应的 commit 上。

作者:vincent (谢文威)
