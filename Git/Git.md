

参考：

https://juejin.cn/post/6895246702614806542#heading-24

[Git命令大全](https://www.jianshu.com/p/93318220cdce)

# Git

## SSH配置

```c
cd ~/.ssh //检查是否存在SSH Key
ls
    
ssh-keygen -t rsa -C "git全局注册的邮箱" //没有key则生成
    
cat id_rsa.pub    //查看公钥，复制到github
    
ssh -T git@github.com //测试是否添加成功
```

------

## 重要概念

![img](pic\git.png)

- Remote：远程仓库
- Repository：本地仓库
- Index：暂存区
- WorkSpace：工作区（正看到的代码）

### 节点

每次`commit`提交会生成新的节点。每个节点是一次增量更新，git会自动对比文件的改变。

<img src="pic\node.png" alt="image-20210423085020265" style="zoom:80%;" />

### HEAD

HEAD是一个指针，工作区当前所看到的全部代码就是HEAD所指向节点以往的历史提交集合。HEAD可以指向节点也可以指向分支，当HEAD指向分支时才具有分支相关的操作

### 分支

分支和`HEAD`一样也是一个指针，创建一个分支的本质就是创建一个指针指向某节点。

------

## 命令

### 提交

```bash
//添加到暂存区
git add	文件名					//往暂存区添加某工作区文件
git add .		  			  //往暂存区添加所有工作区文件

//撤销
git reset HEAD 文件名 		     //撤销暂存区文件
git checkout --文件名			//撤销工作区某文件的改动，还原到之前的修改		

//提交本地仓库
git commit -m "该节点的描述信息"  //提交暂存区到本地仓库
git commit --amend              //修改上次提交的描述信息
```

### 分支相关

- 创建分支

```bash
git branch 分支名		//在HEAD指向的节点创建一个分支
```

- 切换分支

```bash
git checkout 分支名    	//HEAD立即指向该分支指向的节点
git checkout -b 分支名  	//创建分支后并且切换到该分支
```

- 删除分支

```bash
git branch -d 分支名		//当某分支完成了它的功能开发后删除该分支，注意只是删除指针而已
```

### 合并

#### Merge

```bash
git merge 分支名/节点哈希值  //创建一个新的节点，当前分支指向该节点，该节点包含两条分支代码的并集。
```

当两个分支不存在`分叉`时，合并不会有冲突。

<img src="pic\merge_single.png" alt="image-20210423094452528" style="zoom:80%;" />

反之则会出现下图情况：

此时会生成一个新节点`C4`来同时包含`C3、C2`的内容。若两个节点同时修改同一文件的同一部分，则还需要先手动消除冲突再合并。

**合并的常规操作：**

1. 通过`git merge bugFix` 在main分支下创建新的节点，该节点会合并`main`和`bugFix`的历史提交。

<img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\merge_mutiple.png" alt="image-20210423121248513" style="zoom: 67%;" />

2. 再通过`git checkout bugFix；git merge main`实现`main`合并到`bugFix`（HEAD指向bugFix）

   <img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\merge1.png" alt="image-20210423123024234" style="zoom:67%;" />

#### Rebase

```bash
#将HEAD指向的分支历史提交记录全部copy到指定分支的后面。
git rebase 分支名/hash    
```

Rebase 实际上就是取出一条分支上的所有提交记录，“复制”它们，然后在另外一条分支后边逐个的放下去。不同于merge会产生分支间的交叉，而是只在一条分支上合并追加。

**在bugFix上 rebase main:**

<img src="pic\rebase.png" alt="image-20210424142914171" style="zoom:67%;" />

注意：提交记录 C3 依然存在（树上那个半透明的节点），而 C3' 是我们 Rebase 到 main 分支上的 C3 的副本。

------

### 版本移动

#### 分离HEAD

将HEAD指向其他分支或者历史节点，当HEAD未指向任何节点时处于分离状态。

```bash
git checkout 节点哈希值			//将HEAD指向hash对应的节点
git checkout --detach        	//也可以指向当前分支

//另外还提供HEAD基于某一分支向前向后相对移动的操作
git checkout  分支名/EHAD^ 	  //HEAD分离并指向指定分支的前一个节点
git checkout 分支名～N			//HEAD分离并指向指定分支的前n个节点
```

#### 强制修改分支位置

将某分支移动到某节点

```bash
git branch -f 分支名 HEAD~3/hash
```

<img src="pic\reset.png" alt="image-20210423150710308" style="zoom: 80%;" />

<img src="pic\reset1.png" alt="image-20210423150811286" style="zoom:80%;" />

#### 版本回退

版本回退有本地版本回退和远程版本回退

**本地分支**

直接清除最新节点，将修改退回到暂存区

```bash
git reset HEAD~N/hash   //回退到指定节点
```

<img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\reset2.png" alt="image-20210423153120512" style="zoom:80%;" />

**远程分支**

revert并不会消除节点，而是追加历史节点的副本

```bash
git revert HEAD/hash  //远程分支回退
```

<img src="pic\revert.png" alt="image-20210423152926897" style="zoom:80%;" />

### 远程仓库

> 如果你看到一个名为 `origin/main` 的分支，那么这个分支就叫 `main`，远程仓库的名称就是 `origin`。	

#### 远程跟踪分支

本地分支要想`fetch，pull、push就必须要和远程分支进行关联（`remote tracking`）。因为不进行关联，本地分支的代码该提交到远程仓库的哪个分支上？

有以下几种方法建立跟踪关系

```bash
# 1. 通过远程分支检出一个新的分支，它跟踪远程分支 o/main
git checkout -b main o/mian
# 2. 本地已经存在的分支跟踪远程分支
git branch -u o/main main   
//如果当前就在 foo 分支上, 还可以省略 foo
git branch -u o/main
```

#### Fetch

- 从远程仓库下载本地仓库中缺失的提交记录
- 更新本地远程分支指针(如 `origin/main`)指向最新记录。

##### Fetch不带参数的用法

```bash
# 将远程仓库所有分支的最新版本全部追加到本地对应的分支上
git fetch 
# 指定哪一个远程仓库
git fetch <remote>
```

- `git fetch`

远程foo、main的历史提交会下载追加到o/foo、o/main后

<img src="pic\fetch7.png" alt="image-20210424222309708" style="zoom:67%;" /><img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\fetch8.png" alt="image-20210424222341438" style="zoom:67%;" />

上面情况是本地和远程未发生冲突。然而很多时候本地分支已经更新了，远程分支被其他小伙伴也更新了，此时就会产生冲突。

<img src="pic\fetch1.png" alt="image-20210424165047529" style="zoom:67%;" /><img src="pic\fetch2.png" alt="image-20210424165145300" style="zoom:67%;" />

##### Fetch带参数的用法

```bash
# 默认先创建本地origin/place（存在则跳过），将远程仓库指定分支place的最新版本追加到origin/place上。
git fetch <remote> <place>

# 默认先创建本地destination分支（存在则跳过），将远程仓库指定source节点（或分支）的最新版本追加到destination上(注意source是远程)。
git fetch <remote> <source> : <destination>
```

- `git fetch <remote> <place>`：

例：`git fetch origin foo`

<img src="pic\fetch3.png" alt="image-20210424214943058" style="zoom:67%;" /><img src="pic\fetch4.png" alt="image-20210424215021109" style="zoom:67%;" />

- `git fetch <remote> <source> : <destination>`

例：`git fetch origin foo~1 : bar`

<img src="pic\fetch5.png" alt="image-20210424220431432" style="zoom:67%;" /><img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\fetch6.png" alt="image-20210424220506670" style="zoom:67%;" />

#### Push

##### Push不带参数的用法

```bash
# 把当前HEAD指向的分支推送到远程关联的分支上，注意不会提交其他分支。
git push
```

##### Push带参数的用法

```bash
#默认先创建远程仓库place分支和本地origin/place分支（存在则跳过），将本地仓库place分支的历史版本提交到远程仓库place上，之后origin/place指向本地place
git push <remote> <place>

#默认先创建远程仓库destination分支和本地origin/destination（存在则跳过），将本地仓库source节点（或分支）的历史版本提交到远程仓库对应的分支destination上(注意source是本地)，之后origin/destination指向本地source
git push <remote> <source>:<destination>

#删除——远程仓库destination的历史提交（慎用）
git push <remote> : <destination>
git push <remote> --delete <destination>
```

- `git push <remote> <place>`

例：先分离HEAD，再使用`git push origin main`明确告诉`git`将本地`main`推送到远程`main`

```bash
git checkout c0
git push origin main
```

<img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\push.png" alt="image-20210424204905057" style="zoom: 67%;" /><img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\push1.png" alt="image-20210424210046441" style="zoom:67%;" />
而若不追加参数直接使用`git push`则不会成功。因为HEAD当前没有追踪任何分支

- `git push origin <source> : <destination>`

例如：`git push o foo^ : main`将本地foo的上一个节点的历史记录推送到远程main分支的后面，若远程main分支不存在，则会创建远程分支main。

<img src="pic\push2.png" alt="image-20210424213018039" style="zoom:67%;" /><img src="C:\Users\lenovo\Desktop\MyStudySummary\MyStudySummary\Git\pic\push3.png" alt="image-20210424213134963" style="zoom:67%;" />

#### Pull

> pull实际上就是fetch+合并的集合指令，其中合并默认用的是merge，当然可以指定rebase。如指令`git pull o foo`等价于`git fetch o foo ; git merge o/foo`(需要格外注意merge的时候HEAD指向的是哪一个节点！)

```bash
#fetch远程仓库所有分支，本地o/source指向最新分支，当前HEAD指向的节点与对应的分支合并
git pull  

#pull默认是fetch+merge，也可以改为fetch+rebase模式
git pull --rebase   

git pull <remote> <place>
```

Fetch完远程最新代码后经常需要与本地`main`合并（o/main的最新节点和本地main已经不在一条分支上了）。Pull相当于fetch+merge的指令集合。

未pull之前：

<img src="pic\pull.png" alt="image-20210423162654731" style="zoom:67%;" />

pull之后：

<img src="pic\pull1.png" alt="image-20210423163412910" style="zoom:67%;" />

****

### 初始化

```bash
git config --global user.name "RhythmCoderZZF"
git config --global user.email "958665147@qq.com"
```

创建仓库并上传码云

```bash
git init
git add README.md
git commit -m "first commit"
git remote add origin git@gitee.com:RhythmCoderZZF/web-summary.git
git push -u origin master 	//-u 表示本地master分支与远程master关联，若远程没有master就新创建
```

> git push -u 的含义：https://www.zhihu.com/question/20019419

### git添加.gitignore文件无效

在添加.gitignore文件后，Android Studio 如果没有忽略我们想要忽略的文件，解决方法就是清除一下缓存。
原因gitignore对已经追踪的文件无效，清除缓存后就可以了。还不行，就从git上重新拉取代码。

```bash
git rm -r --cached .
git add .
git commit -m "clear cached"
```

