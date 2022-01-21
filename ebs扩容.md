#### 备份快照 - aws console

#### 调整EBS大小 - aws console

#### 查看当前文件系统所使用的大小和百分比。
```
ubuntu@ip-172-31-32-114:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1      7.7G  7.7G     0 100% /
/dev/xvdf       7.9G  7.1G  370M  96% /home/ubuntu/test
```

#### 使用 lsblk 命令显示 xvdf 卷的大小。 
```
ubuntu@ip-172-31-32-114:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdf    202:80   0   16G  0 disk /home/ubuntu/test
```

#### 使用 resize2fs 命令可自动将 /dev/xvdf/ 文件系统的大小扩展到卷上的完整空间。
```
ubuntu@ip-172-31-32-114:~$ sudo resize2fs /dev/xvdf
```

#### 重新运行 df –h 命令。
```
ubuntu@ip-172-31-32-114:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1      7.7G  7.7G     0 100% /
/dev/xvdf        16G  7.1G  8.0G  48% /home/ubuntu/test
```
