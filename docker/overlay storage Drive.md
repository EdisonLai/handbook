参考链接：https://blog.csdn.net/luckyapple1028/article/details/78075358  

以下是linux命令行建立一个overlay storage的实现方式。  
在需要联合的文件夹以外，需要额外两个文件夹，在本处例子中为worker与merge  

worker文件夹为overlay需要的额外文件夹。  
merge为最终展示的文件夹

lower按照由高到低的分层顺序组织。lower为只读的文件。  
upper为对该overlay存储做操作时，变更的存储位置。当upper未指定时，能创建只读的overlay存储。  


```text
687  2021-08-02 19:37:38 mkdir -p lower1 lower2 upper worker merge
688  2021-08-02 19:37:55 mkdir -p lower1/dir lower2/dir upper/dir
689  2021-08-02 19:38:19 touch lower1/foo1 lower2/foo2 upper/foo3
690  2021-08-02 19:39:00 touch lower1/dir/aa lower2/dir/aa lower1/dir/bb upper/dir/bb
691  2021-08-02 19:39:17 echo "from lower1" > lower1/dir/aa
692  2021-08-02 19:39:28 echo "from lower2" > lower2/dir/aa
693  2021-08-02 19:39:35 echo "from lower1" > lower1/dir/bb
694  2021-08-02 19:39:53 echo "from upper" > upper/dir/bb
695  2021-08-02 19:40:29 mount -t overlay overlay -o lowerdir=lower1:lower2,upperdir=upper,workdir=worker merge
```

+ 删除  
  > 当删除lower层的文件时，会在upper层中创建被删除文件的whiteout文件。让系统不显示该文件。  
  > 新建存在whiteout文件的同名文件时，会删除whiteout文件，并在upper层中新建该文件，以覆盖lower中的文件