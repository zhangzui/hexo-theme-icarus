#linux常用命令介绍
linux常用命令介绍
<!--more-->
##ssh登录
ssh wanglu9@accessgw-wfj.cbpmgt.com -p 22
2.目前磁盘剩余的空间：df -h
3.查看当前的目录所占的磁盘空间：du -h
    du/df/fdisk
    du +文件名 显示目录文件大小 （默认kb?)
    du -h 大小按照合理单位输出,如G、kb
    df 硬盘被占用了多少空间，目前还剩下多少空间等信息
    df -hl 查看磁盘剩余空间
    df -h 查看每个根路径的分区大小
    du -sh [目录名] 返回该目录的大小
    du -sm [文件夹] 返回该文件夹总M数

    $ du /home 得到目录下磁盘使用情况，
    $ du -h /home:读取的大小是KB 或者 MB
    $ du -s /home：总磁盘大小
    df 命令：显示设备名称、总块数、总磁盘空间、已用磁盘空间、可用磁盘空间和文件系统上的挂载点。
    $ df -h：人类可读格式的信息
    $ df -hT /etc： 显示特定分区的信息





4.查找文件
    a.find -name "*xxx*"
    b.find /home/ -size 500M
        find 硬搜索文件名，可模糊查询
       which 通过path查找可执行文件
       whereis 数据库查，只搜索二进制、手册、源代码文件
       locate 同样数据库查询，应该是文件类型不限
       数据库查询缺点：更新不及时，搜索到已被删除的文件
       grep 在输出流中找到内容
5.查看目录下的所有文件
    ls -a
    ls -l 查看文件详细
    ls -F 查看表示文件类型的符号（×代表可执行文件，/ 表示目录，@表示连结文件）（ls 其实已经内建了-F 参数）
6.合并文件：
    cat file1 file2 > file3
    cat file 一次显示文件内容
    cat > file 创建新文件，不能编辑
7.less 从下往上显示文件;
    ? 向上搜索文件
    / 向下搜索文件
    n 重复前一个搜索
    N 反向重复前一个搜索
    -N 显示行号
8.more 从上往下显示文件
9.tail 跟踪显示日志
    tail:显示文件结尾 -f 当文件增长时,输出后续添加的数据,
   可以用作最新日志输出
   -n 输出最后N行
   -s 与-f合用,表示在每次反复的间隔休眠S秒]
10.clear 清屏
11.查找文件中符合字符串的哪行：grep
   grep -n st text.c
   -n 显示行号
12.chmod:4可读，2写，1可执行d开头代表目录
   u文件所属人g代表群组o代表其他人,a代表所有
   chmod 777 文件名chmod a+rwx 文件名
   所有人拥有读写操作权限
13.fdisk：诸多功能，可以用来划分分区
14.创建文件命令
   touch 文件名
   vi 文件名（文件名不存在的话）
   mkdir 文件夹名
   mv 移动文件，也算创建？
15.远程访问
   scp 文件移动到远程服务器
   wget 下载服务器文件到本地
   ssh 远程登录
16.压缩和解压
   tar：打包
   tar -xvf file.tar
   //解压
   tar包tar -xzvf file.tar.gz
   //解压tar.gz
   -x 解压-z gizp文件
   -v 过程显示-f 后面跟解压文件名
   gzip:.gz打包
   gunzip：.gz解压
   zip:.zip压缩
   unzip:解压zip
17.服务器状况
   kill 杀死进程
   kill -9 强制杀死
   free：查看内存使用情况
   ps:瞬间进程动态
    ps -aux
    显示其他用户启动的进程（a）
   查看系统中属于自己的进程（x）
   启动这个进程的用户和它启动的时间（u）
   netstat：显示网络连接.路由表和网络接口信息
18.ifconfig:查看ip网关等信息
19.netstat 显示网络连接.路由表和网络接口信息
    netstat -aux
    查看端口：netstat -tunlp
    top:查看系统的CPU、内存、运行时间、
   交换分区、执行的线程等信息
   ps:瞬时查看进程
   ps -a 显示所有进程 -u 显示用户相关信息
    -x 显示所有进程端，不以终端进行区分
20.date -s "2016-3-1 08:28:00"

解决方案：这是win的编码引起的，可通过如下解决。
1.查看该文件：vim  start.sh
2.查看该错误文件的格式（一般报错的文件格式是DOS）：
  :set ff
3.修改该文件格式为UNIX：
  :set ff=unix
4.再保存。
 :wq!





