---
title: 执行rm -rf后的一种恢复方案
date: 2018-08-02 19:04:58
tags: Others
---

如果使用`rm -rf`指令删除了某个问价夹，比如：

```bash
rm -rf importantFile
```

删完之后发现这个文件夹里面的内容都是有用的，这时候先不要慌张，可以使用`lsof | grep  FileName`指令查看是否有进程在使用这个文件，如果可以找到，就可以使用如下的解决方案，如果没有找到，那么就需要借助于其它的工具解决了。

比如我的操作：

```bash
[work(mike)@tjtxvm-224-15 web]$ lsof | grep importantFile
java      16941      work  cwd       DIR              252,2        0   4718641 /opt/web/importantFile (deleted)
java      16941      work  DEL       REG              252,2            4718775 /opt/web/importantFile/temp/snappy-unknown-02c7c6de-6fa4-4f3b-b699-ac56a9d6c78a-libsnappyjava.so(deleted)
java      16941      work  DEL       REG              252,2            5112728 /opt/web/importantFile/webapps/WEB-INF/lib/xercesImpl-2.6.2.jar(deleted)
java      16941      work  DEL       REG              252,2            5112601 /opt/web/importantFile/webapps/WEB-INF/lib/xalan-2.6.0.jar(deleted)
java      16941      work  DEL       REG              252,2            5112699 /opt/web/importantFile/webapps/WEB-INF/lib/wredis-udp-1.0.2.jar(deleted)
java      16941      work  DEL       REG              252,2            5112594 /opt/web/importantFile/webapps/WEB-INF/lib/wredis-client-redis-2.0.4.jar(deleted)
java      16941      work  DEL       REG              252,2            5112564 /opt/web/importantFile/webapps/WEB-INF/lib/wredis-client-basic-2.0.4.jar(deleted)
// ....
```

从中我们可以看出java进程在使用这些文件,并且这个进程的**进程号是：16941**。我们可以进入该进程的文件描述符目录中查看它都使用了哪些文件描述符(fd是file descriptor的缩写)：

```bash
[work(mike)@tjtxvm-224-15 web]$ cd /proc/16941/fd
[work(mike)@tjtxvm-224-15 fd]$ ls
0    102  107  111  142  157  183  20   222  238  247  258  262  267  271  276  280  285  29   294  299  302  307  311  316  34  40  45  5   54  59  63  68  72  77  81  86  90  95
1    103  108  112  143  16   184  21   225  24   248  259  263  268  272  277  281  286  290  295  3    303  308  312  32   36  41  46  50  55  6   64  69  73  78  82  87  91  96
10   104  109  12   15   167  19   219  226  241  25   26   264  269  273  278  282  287  291  296  30   304  309  313  324  38  42  47  51  56  60  65  7   74  79  83  88  92  97
100  105  11   13   153  17   196  22   23   242  256  260  265  27   274  279  283  288  292  297  300  305  31   314  325  39  43  48  52  57  61  66  70  75  8   84  89  93  98
101  106  110  14   156  18   2    220  232  244  257  261  266  270  275  28   284  289  293  298  301  306  310  315  33   4   44  49  53  58  62  67  71  76  80  85  9   94  99
```

这些就是文件描述符，但是从中我们看不到其所指的具体文件，可以使用ll指令查看描述符的详细信息，同时，如果想要过滤出和某个目录相关的信息可以结合**grep**指令。

查询所有被删除的文件：

```bash
[work(mike)@tjtxvm-224-15 fd]$ ll | grep importantFile
l-wx------ 1 work work 64 8月   2 10:41 1 -> /opt/web/importantFile/logs/catalina.out (deleted)
lr-x------ 1 work work 64 8月   2 10:41 100 -> /opt/web/importantFile/webapps/WEB-INF/lib/com.me.vip.scf_service.utils-2.2.0-SNAPSHOT.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 101 -> /opt/web/importantFile/webapps/WEB-INF/lib/com.me.vip.sns.contract-1.3.8.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 102 -> /opt/web/importantFile/webapps/WEB-INF/lib/com.me.vip.wlt.contract-4.5.7.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 103 -> /opt/web/importantFile/webapps/WEB-INF/lib/com.me.wf.core-1.2.7.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 104 -> /opt/web/importantFile/webapps/WEB-INF/lib/com.me.wf.mvc-1.2.17.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 143 -> /opt/web/importantFile/webapps/WEB-INF/lib/dionaea-common-0.0.1-SNAPSHOT.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 222 -> /opt/web/importantFile/webapps/WEB-INF/lib/snappy-java-1.1.1.6.jar (deleted)
lr-x------ 1 work work 64 8月   2 10:41 225 -> /opt/web/importantFile/webapps/WEB-INF/lib/spring-beans-3.0.5.RELEASE.jar (deleted)
// .......

```

从中可以看到文件最后的括号中有一个deleted的标示。其实我们只要使用cp指令把相应的描述符拷贝到相应的文件即可：

```bash
cp FdNumber DestinationFile
```

但因为这时`DestinationFile`这个文件的所有目录都被我们删除了，所以不能用直接用cp指令来做这件事，需要使用如下的脚本:

```bash
test -d "$des" || mkdir -p "$des" && cp $src $des
```

这里第一个参数是原文件，第二个参数是目标文件以及其路径。这样我们就可以通过循环来获取每个文件的描述符和其目标地址来进行恢复，所有的代码如下([脚本的地址](http://7xsbfz.com1.z0.glb.clouddn.com/rmrf_recover.sh))：

```bash
#
# 使用方法：./rmrf_recover.sh pid deletedFillName
#

#
# 判断传入的参数是否正确
#

if [ "$#" -ne 2 ]
then
    echo "Incorrect number of argumments."
    echo "Usage:./rmrf_recover.sh  pid  DeletedFillName"
    exit 1
fi

#
# 进入到需要应用删除文件进程的文件描述符目录
#

fdDir="/proc/""$1""/fd/"
cd $fdDir

temp_deleted_listPath=/tmp/temp_deleted_list

#
# 获取被删除文件的详细信息
#

ls -l | grep "$2" > $temp_deleted_listPath


echo "Process Id: ""$1"
echo "Deleted Filename: ""$2"
echo "Deleted List:"
## 输出删除的列表
cat $temp_deleted_listPath

#
# 获取所有的文件描述符
# 这里的11和13是以空格为分割符的，需要根据自己的情况做调整
#

fileDeses=$(cut -d' ' -f11 $temp_deleted_listPath)
fileDires=$(cut -d' ' -f13 $temp_deleted_listPath)

## 输出最终的值

echo $fileDes
echo $fileDires

#
# 将被删除文件的全目录存储到fileArray中
#

fileNum=0
for fileDir in $fileDires
do
    fileArray[$fileNum]=$fileDir
    fileNum=$((fileNum + 1))
done

#
# 遍历所有的文件描述符并且恢复
#

num=0
for fileDes in $fileDeses
do
    echo " DeletedFile:---------"$fileDes" " 
    echo " DestinationFile:-----""${fileArray[num]}"
    test -d ${fileArray[num]} || mkdir -p ${fileArray[num]} && cp $fdDir$fileDes ${fileArray[num]}
    num=$((num + 1))
done

rm -rf $temp_deleted_listPath

```
