---
date: 2020-10-16
title: "工作中常用Linux命令整理"
tags: ["Linux", "常用命令"]
---

### 系统参数相关

Linux系统中使用ps命令时支持3种不同类型的命令行参数：

- Unix风格，前面加`-`

- BSD风格，前面不加`-`

- GNU风格，前面加`--`

#### 查看各个进程内存使用情况，并取前面10行

```shell
ps aux --sort -rss | head -10
```

参数描述：

> a：显示跟任意终端关联的所有进程
>
> u：采用基于用户的格式显式
>
> x：显示所有进程，包括未分配任何终端的进程
>
> --sort -rss：按照rss排序显示内容
>
> rss：RSS is Resident Set Size, the non-swapped physical memory used by process.

参考:

[Linux三种风格（Unix、BSD、GNU）下的ps的参数说明](https://blog.csdn.net/ruibin_cao/article/details/84660224)

[Linux查看内存使用情况方法](https://www.jianshu.com/p/e9e0ce23a152)

#### 查看磁盘使用情况

`df`（disk free缩写）

```bash
df -h
```

du (disk usage缩写)

```bash
# 查看当前目录磁盘占用情况
df -d 1 -h
# 查看指定目录磁盘占用总和
df -s /home -h
```

####   查看内存使用情况

首选：

```shell
free -h
```

free 其它参数：

>-m：以MB为单位显示
>
>-g：以GB为单位显示

备用：

```shell
cat /proc/meminfo 
```

参考：

[查看内存](https://www.jianshu.com/p/a9f01a5c3158)

#### 查看版本信息

查看内核版本

首选：

```bash
uname -r
```

其它：

```bash
cat /proc/version
```

```bash
uname -a
```

查看Linux版本

首选：

```bash
lsb_release -a
```

可选：

```bash
cat /etc/issue
```

#### 查看虚拟内存统计

命令介绍：

```bash
用法：
 vmstat [options] [delay [count]]

选项：
 -a, --active           active/inactive memory
 -f, --forks            number of forks since boot
 -m, --slabs            slabinfo
 -n, --one-header       do not redisplay header
 -s, --stats            event counter statistics
 -d, --disk             disk statistics
 -D, --disk-sum         summarize disk statistics
 -p, --partition <dev>  partition specific statistics
 -S, --unit <char>      define display unit（使用指定单位显示，常用有：K,M）
 -w, --wide             wide output
 -t, --timestamp        show timestamp

 -h, --help     显示此帮助然后离开
 -V, --version  显示程序版本然后离开
```

##### 查看虚拟内存使用情况

```bash
# 每2秒刷新一次,刷新3次,显示单位为:M
vmstat 2 3 -S M
#输出结果:
r  b 交换 空闲 缓冲 缓存   si   so    bi    bo   in   cs us sy id wa st
 0  0    879    118    205    655    0    0     3     5    1   27  0  0 99  0  0
 0  0    879    118    205    655    0    0     0     0  485 1284  0  1 99  0  0
 0  0    879    118    205    655    0    0     0     0  497 1299  0  0 99  0  0
```

字段说明可参考[博客](https://www.cnblogs.com/peida/archive/2012/12/25/2833108.html)。

##### 查看内存使用的详细信息

```bash
vmstat -s -S M
# 输出结果:
         3908 M total memory
         2927 M used memory
         1995 M active memory
         1179 M inactive memory
          119 M free memory
          205 M buffer memory
          655 M swap cache
          947 M total swap
          879 M used swap
           67 M free swap
       273022 non-nice user cpu ticks
         7456 nice user cpu ticks
       441127 system cpu ticks
    134508428 idle cpu ticks
        30791 IO-wait cpu ticks
            0 IRQ cpu ticks
        97308 softirq cpu ticks
            0 stolen cpu ticks
      4352762 pages paged in
      7436572 pages paged out
        58397 pages swapped in
       275492 pages swapped out
    345239276 interrupts
    896027400 CPU context switches
   1602136351 boot time
        48266 forks
```



### 权限相关

#### 修改`root`用户密码

在`root`用户身份下，直接输入`passwd`，输入两遍新密码确认即可。

### 用户操作相关

#### 创建一个普通用户

```bash
adduser demo
```

#### 给指定用户赋予`root`权限

- 修改`/etc/sudoers`文件，默认没有写权限，先给该文件添加写权限。

```bash
chmod u+x /etc/sudoers
```

- 修改`/etc/sudoers`文件，找到`root    ALL=(ALL:ALL) ALL`这一行，在下面添加`xxx  ALL=(ALL:ALL) ALL`，这里`xxx`是添加的用户名。

- 撤销文件的写权限

  ```bash
  chmod u-x /etc/sudoers
  ```

  

  ​	

  

  