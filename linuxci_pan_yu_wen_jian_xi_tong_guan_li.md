# Linux磁盘与文件系统管理

###认识ext2文件系统

索引式文件系统

* superblock：记录此filesystem的整体信息，包括inode/block的总量、使用量、剩余量，以及文件系统的格式与相关信息等
* inode：记录档案的属性，一个档案占用一个inode，同时记录此档案的数据所在的block号码
* block：实际记录档案的内容，若档案太大时，会占用多个block

![](选区_001.png)

###data block（资料区块）

data block是用来放置档案内容数据的地方

* 原则上，block的大小与数量载格式化完就不能再改变
* 每个block内最多只能放置一个档案的数据
* 若档案大于block，则一个档案会占用多个block
* 若档案小于block，则该block的剩余容量就不能再被使用

###inode table（inode表格）

inode的内容至少有

* 该档案的存取模式（read/write/excute）
* 该档案的拥有者与群组（owner、group）
* 该档案的容量
* 该档案建立或状态改变的时间（ctime）
* 最近读取时间（atime）
* 最近修改时间（mtime）
* 定义档案特性的flag，如SetUID
* 该档案真正内容的指向

inode的特点

* 每个inode的大小均固定为128bytes
* 每个档案都仅会占用一个inode
* 文件系统能够建立的档案数量与inode的数量有关
* 系统读取档案时需要先找到inode，并分析inode所记录的权限与用户是否符合，若符合才能开始实际读取block的内容`

![](选区_002.png)

###superblock（超级区块）

* block与inode的总量
* 未使用与已使用的inode/block数量
* block与inode的大小
* filesystem的挂载时间、最近一次写入数据的时间、最近一次检验磁盘的时间等文件系统的相关信息
* 一个valid bit数值，若此文件系统已被挂载，则valid bit为0,若未被挂载，则valid bit为1

###Filesystem Description（文件系统描述说明）

这个区段可以描述每个block group的开始与结束的block号码，以及说明每个区段分别介于哪一个block号码之间

###block bitmap（区块对照表）

记录了所有block的使用信息

###inode bitmap（inode对照表）

记录了所有inode的使用信息

```
dumpe2fs [-bh] 装置文件名

-b：列出保留为坏轨的部分
-h：仅列出superblock的数据，不会列出其他区段内容
```

###日志式文件系统

* 预备：当系统要写入一个档案时，会先在日志记录某个档案准备要写入的信息
* 实际写入：开始写入档案的权限与数据，开始更新metadata的数据
* 结束：完成数据与metadata的更新后，在日志记录区块当中完成该档案的记录


* metadata（中介资料）：super block、block bitmap、inode bitmap等
* 数据存放区域：inode block、data block


###Linux文件系统的运作

* 系统会将常用的档案数据放置到主存储器的缓冲区，以加速文件的读写
* 因此Linux的物理内存最后都会被用光
* 可以手动使用sync来强迫内存中设定为Dirty的档案回写当磁盘中
* 正常关机时，关机指令会主动呼叫sync来将内存的数据回写入磁盘内
* 不正常关机，由于数据尚未回写到磁盘内，因此重新启动后可能会花很多时间在进行磁盘检验，甚至可能导致文件系统的损毁

###Linux VFS（Virtual Filesystem Switch）

![](VFS 文件系统示意图.png)

###文件系统的简单操作

####df：列出文件系统的整体磁盘使用量

```
df [-ahikHTm] [目录或文件名]

-a：列出所有文件系统，包括系统特有的/proc等文件系统
-k：以KBytes的容量显示各文件系统
-m：以MBytes的容量显示各文件系统
-h：以易阅读的GBytes、MBytes、KBytes等格式自行显示
-H：以M=1000K取代M=1024K的进位方式
-T：连同该partition的filesystem名称也列出
-i：不用硬盘容量，而以inode数量来表示

常用：hi
```

####du：评估文件系统的磁盘使用量

```
du [-ahskm] 档案或目录名称

-a：列出所有档案与目录容量，因为默认仅统计目录底下的档案量而已
-h：以易读的方式显示
-s：列出总量，而不列出每个目录占用容量
-S：不包括子目录下的总计
-k：以KBytes列出
-m：以MBytes列出
```

####实体链接与符号链接：ln

* Hard Link：实体链接，直接链接inode，不使用block，不能跨filesystem，不能link目录
* Symbolic Link：符号链接，创建新档案资源读取指向原档，占用inode与block

```
ln [-sf] 来源文件 目录文件

-s：默认hard link，加上后symbolic link
-f：如果目标文件存在时，就主动将目标文件直接移除后再建立
```
###磁盘的分割、格式化、检验与挂载

####磁盘分区：fdisk

```
fdisk [-l] 装置名称

-l：输出后面接的装置所有的partition内容。若仅有fdisk -l时，则系统将会把整个系统内能够搜索到的装置的partition均列出来

无法处理大于2TB的磁盘分区槽
```

####磁盘格式化：mkfs

```
mkfs [-t 文件系统格式] 装置文件名

-t：可以接文件系统格式，例如ext3、ext2、vfat等（系统支持才有效）
```

####磁盘检验：fsck、badblocks

fsck

```
fsck [-t 文件系统] [-ACay] 装置名称

-t：后接文件系统，通常会自动分辨
-A：依据/etc/fstab的内容，将需要的装置扫描一遍
-a：自动修复检查到的有问题的
-y：与-a类似，但是某些filesystem仅支持-y这个参数
-C：可以再检验的过程中，使用一个直方图来显示目前的进度
```

badblocks

```
badblocks -[svw] 装置名称

-s：在屏幕上列出进度
-v：在屏幕上列出进度
-w：使用写入的方式来测试
```

####磁盘挂载与卸除

* 单一文件系统不应该被重复挂载在不同的挂载点中
* 单一目录不应该重复挂载多个文件系统
* 要作为挂载点的目录，理论上应该都是空目录

```
mount -a
mount [-l]
mount [-t 文件系统] [-L label名] [-o 额外选项] [-n] 装置文件名 挂载点

-a：依照配置文件/etc/fstab的数据将所有未挂载的磁盘都挂载上来
-l：单纯的输入mount会显示当前挂载的信息，加上-l可增加Label名称
-t：后接文件系统类型
-n：默认情况下，系统会将实际挂载的情况实时写入/etc/mtab中，加上后不会写入
-L：系统除了利用装置文件名之外，还可以利用文件系统的Label来进行挂载
-o：后面可以接一些挂载时额外加上的参数
```

```
umount [-fn] 装置文件名或挂载点

-f：强制执行
-n：不更新/etc/mtab情况下卸除
```

####磁盘参数修订

mknod

```
mknod 装置文件名 [bcp] [Major] [Minor]

装置种类：
  b：设定装置名称成为一个周边存储设备档案
  c：设定装置名称成为一个周边输入设备档案
  p：设定装置名称成为一个FIFO档案
Major：主要装置代码
Minor：次要装置代码
```

hdparm

```
hdparm [-icdmXTt] 装置名称

-i：将核心侦测到的硬盘参数显示出来
-c：设定32-bit存取模式
-d：设定是否启用dma模式
-m：设定同步读取多个sector模式
-X：设定UtraDMA模式
-T：测试暂存区的存取性能
-t：测试硬盘的实际存取性能
```

###设定开机挂载

####开机挂载/etc/fstab及/etc/mtab

系统挂载的限制

* 根目录/是必须挂载的，而且一定要先于其它mount point被挂载进来
* 其它mount point必须为已建立的目录。可以任意指定，但一定要遵守必须的系统目录架构原则
* 所有mount point再同一时间之内，只能挂载一次
* 所有partition在同一时间之内，只能挂载一次
* 若进行卸除，必须先将工作目录移到mount point及其子目录之外

###文件系统的特殊观察与操作

####boot sector与superblock的关系

* superblock大小为1024bytes
* superblock前面需要保留1024bytes下来，以让开机管理程序可以安装
* block为1024bytes时，boot sector与superblock在不同的block内
* block大于1024bytes时，boot sector与superblock在同一block内

####利用GNU的parted进行分割行为

```
parted [装置] [指令 [参数]]

新增分割：mkpart [分割槽类型] [文件系统] 开始 结束
分割表：print
删除分割：rm [partition]
```
























