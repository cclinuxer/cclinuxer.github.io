在系统中创建一个空文件：

> dd if=/dev/zero of=./ext4_image_test bs=4096  count=40960



将这个空文件格式化为指定的系统：

mkfs.ext4 -b 4096 ./ext4_image_test 



将文件系统mount到刚才的目录:

加上-o loop的意思是将这个文件当成设备挂载上去

sudo mount -t ext4 -o loop ./ext4_image_test  ./ext4_test/



查看文件系统的内容：

dumpe2fs   ./ext4_image_test 



查看文件的inode号：这就会显示出在这个文件系统下该文件的innode号，我们根据

dumpe2fs打印出来的内容就可以进行调试了。

df -i  xxxx



hexdump出来某一段数据，就可以分析二进制文件了



debugfs 可以查看数据：



blesss分析二进制：

https://blog.csdn.net/u010168781/article/details/79649486



ext4: layout

https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Layout



debugfs恢复文件：

https://www.jianshu.com/p/90142b8a5c1b









