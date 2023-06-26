

## Linux三剑客

- grep
- sed
- awk

### grep

#### 使用场景

> 很多时候，我们并不需要列出文件的全部内容，而是从文件中找到包含指定信息的那些行

#### 介绍

> grep命令能够在一个或多个文件中，搜索指定的字符(可使用正则)，当然也可以是单一的字符、字符串、单词、句子等。
>
> **正则表达式**：是表述一组字符串的一个模式，正则表达式的构成模仿了数学表达式，通过使用操作符将较小的表达式组合成一个新的表达式，正则表达式可以是一些纯文本文字，也可以是用来产生模式的一些特殊字符

#### grep语法

```yaml
语法：
grep	[options]	[pattern]	file
命令		参数		匹配模式	文件名
					-i 忽略大小写
					-c 统计出现的字数
					-n 输出行号
					-v 反向匹配
					-E 开启扩展正则
```

#### 基本正则表达式

- 字符匹配

| 通配符      | 功能                                                         |
| ----------- | ------------------------------------------------------------ |
| [abc]       | 匹配方括号中的任意一个字符   **示例1**                       |
| [^xyz]      | 匹配除方括号中字符外的所有字符 **示例2**                     |
| ^           | 锁定行的开头  **示例3**                                      |
| $           | 锁定行的结尾  **示例4**                                      |
| ^$          | 空行                                                         |
| [[:alnum:]] | 字母和数字                                                   |
| [[:alpha:]] | 代表任何英文大小写字符，亦即A-Z, a-z                         |
| [[:lower:]] | 小写字母[:upper:] 大写字母                                   |
| [[:blank:]] | 空白字符（空格和制表符）                                     |
| [[:space:]] | 水平和垂直的空白字符（比[:blank:]包含的范围广）              |
| *           | 表示前面的字符匹配任意次，可以0次，可以无限，贪婪模式  **示例5** |
| .           | 将匹配任何一个字符，且只能是一个字符  **示例6**              |
| .*          | .* 表示任意内容任意长度  **示例7**                           |

> 补充：
>
> \b边界匹配  \broot\b 单个单词root匹配
> \B非边界匹配  xxx\B
> \w字母数字下划线
> \W非数字字母下划线

##### 示例

- 示例1

![image-20230227155926145](https://gitee.com/root_007/md_file_image/raw/master/202302271559217.png)

- 示例2

  ![](https://gitee.com/root_007/md_file_image/raw/master/202302271600501.png)

- 示例3

  ![image-20230227160155373](https://gitee.com/root_007/md_file_image/raw/master/202302271601407.png)

- 示例4

  ![image-20230227160221918](https://gitee.com/root_007/md_file_image/raw/master/202302271602960.png)

- 示例5

  ![image-20230227160304635](https://gitee.com/root_007/md_file_image/raw/master/202302271603674.png)

- 示例6

  ![image-20230227160325118](https://gitee.com/root_007/md_file_image/raw/master/202302271603153.png)

- 示例7

  ![image-20230227160343805](https://gitee.com/root_007/md_file_image/raw/master/202302271603841.png)

#### 扩展正则表达式

| 通配符        | 功能                                      |
| ------------- | ----------------------------------------- |
| **\?**        | **表示前面的内容匹配0次或1次 **  示例1    |
| **\\+**       | **表示前的的内容匹配1次以上**  示2        |
| **\\{n\\}**   | **匹配前面的字符n次**                     |
| **\\{m,n\\}** | **匹配前面的字符至少m次，至多n次 ** 示例3 |
| **\\{,n\\}**  | **匹配前面的字符至多n次**                 |
| **\\{n,\\}**  | **匹配前面的字符至少n次**                 |

> -E 等于egrep 使用扩展正则表达式

##### 	示例

- 示例1

  ![image-20230227160648457](https://gitee.com/root_007/md_file_image/raw/master/202302271606500.png)

- 示例2

  ![image-20230227160702367](https://gitee.com/root_007/md_file_image/raw/master/202302271607402.png)

- 示例3

  ![image-20230227160729593](https://gitee.com/root_007/md_file_image/raw/master/202302271607625.png)

#### grep -e 使用

> -e 表示条件选择 一般使用格式为 -e 条件 -e 条件

示例：

```shell
[root@localhost ~]# grep -e "shutdown"  -e "nologin" /etc/passwd 
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
tcpdump:x:72:72::/:/sbin/nologin
ntp:x:38:38::/etc/ntp:/sbin/nologin
nginx:x:997:993:Nginx web server:/var/lib/nginx:/sbin/nologin

```

### sed

#### 使用场景

> sed是一种流编辑器，它是文本处理工具，支持正则表达式，通过一行一行遍历，执行相应命令，来处理、编辑文本文件。主要用来对文件行进行操作

#### 常用选项

> -n 经过sed处理的行进行输出
> -e 支持多个sed语句 与grep类似
> -r 支持扩展正则表达式
> -i 使sed语句修改源文件

##### 示例

​	-n使用

```shell
[root@localhost ~]# seq 1 10 |sed   '2p'
1
2
2
3
4
5
6
7
8
9
10
[root@localhost ~]# seq 1 10 |sed  -n  '2p'
2

```

-e使用

```
[root@localhost ~]# seq 1 10 |sed  -n  -e     '2p'  -e  '3p'
2
3

```

-r使用

```shell
[root@localhost ~]# cat sed_test 
10
11
12
13
14
15
16
17
18
19
20
a
b
c
d
aaa
sadfasf
bad

[root@localhost ~]# cat sed_test | sed -n  -r  '/^a.*/p'
a
aaa

```

-i使用，一般与**常用命令**中的s一起使用

```
[root@localhost ~]# sed -i 's/1/2/g' sed_test  
[root@localhost ~]# cat sed_test  
20
22
22
23
24
25
26
27
28
29
20
a
b
c
d
aaa
sadfasf
bad


```



#### 常用命令

> a 新增 a的后面可以接字符串，而这些字符串会在新的一行出现(下一行)
> c 取代 c的后买你可以接字符串，
> d 删除 删除指定的行
> i 插入 i的后面可以接字符串，这些字符串会在新的一行出现(上一行)
> p 行打印
> y 将字符串换成另外一个字符,只能替换字符不能替换字符串，且不支持正则表达式
> n 打印奇数行(sed -n 'p;n')、偶数行(sed -n 'n;p')
> s 用一个字符串替换另外一个字符串  seq 1 10 |sed   's/1/2/g' g为全部匹配 i为忽略大小写  gi一般使用  //里可以使用正则
> /xx/ 包含xx字符串
> n,m 打印n到m行  一般与-n结合使用  如果n>m 只会打印第n行

a的使用

```
[root@localhost ~]# seq 1 10 | sed 'a xx'
1
xx
2
xx
3
xx
4
xx
5
xx
6
xx
7
xx
8
xx
9
xx
10
xx

```

> 没有指定行，每行都下边(下一行都添加)

```
[root@localhost ~]# seq 1 10 | sed '2a xx'
1
2
xx
3
4
5
6
7
8
9
10

```

> 在指定行下一行添加

c的使用

```
[root@localhost ~]# seq 1 10 | sed '2c xx'
1
xx
3
4
5
6
7
8
9
10

```

p的使用

```
[root@localhost ~]# seq 1 10 | sed '2p'
1
2
2
3
4
5
6
7
8
9
10
[root@localhost ~]# seq 1 10 | sed -n '2p'
2

```

y的使用

```
[root@localhost ~]# seq 1 10 | sed 'y/1/2/'
2
2
3
4
5
6
7
8
9
20

```

/xx/的使用

```
[root@localhost ~]# seq 1 10 | sed -n    '/1/p'
1
10

```

n,m的使用

```
[root@localhost ~]# seq 1 10 | sed -n   '2,3p'
2
3
[root@localhost ~]# seq 1 10 | sed -n   '2,1p'
2

```

扩展其他用法

```
seq 1 10 |sed -n '1~2p'
seq 1 10 |sed -n '1~2!p'  或 seq 1 10 |sed -n '2~2p'
seq 1 10 |sed -n '$p'
seq 1 10 |sed -n '3,5p'
seq 1 10 |sed -n '2,+3p'
```



#### 高级用法(了解即可)

> seq 1 8 |sed -n 'n;p'  输出偶数行
> seq 1 10 |sed '1!G;h;$!d' 或 seq 1 10 |sed -n '1!G;h;$p' 倒序输出
> seq 1 10 |sed 'N;D' 或 seq 1 10 |sed '$!d' 只输出最后一行
> seq 1 10 |sed '$!N;$!D' 输出倒数后两行
> seq 1 10 |sed 'G' 每行后加一个空行
> seq 1 10 |sed 'g'  所有行变为空行
> cat seq.txt |sed '/^$/d;G' 空行删除,每行后加一个空行,即保证每行后只有一个空行
> seq 1 10 |sed 'n;d' 只显示奇数行

### awk

#### 使用场景

> awk作为三剑客之一，其更是一种编程语言，用在linux/unix下对文本和数据进行处理，数据可以来自标准输入、一个文件或其他命令的输出

#### 语法结构

awk 分隔符 'BEGIN{ 动作;… } pattern{ 动作;… } END{ 动作;… }' 文件

> BEGIN动作是读文件前执行的，一般用于头文件的生成，pattern部分是读文件的部分(pattern标识符省略)，基于文件的行来处理，END是文件读之后的动作，一般用于总结等

示例说明

```
[root@localhost ~]# cat /etc/passwd | awk 'BEGIN{print "hello world"} {print $3} END{print "end"}' 
hello world













Management:/:/sbin/nologin
bus:/:/sbin/nologin
polkitd:/:/sbin/nologin





server:/var/lib/nginx:/sbin/nologin
end

```

> 默认分隔符为**空格**

#### 常用内置变量

- FS 输入字段分隔符，类似cut -d 

  ```
  [root@localhost ~]# awk -v FS=":" '{print $1}' /etc/passwd
  root
  bin
  daemon
  adm
  lp
  sync
  shutdown
  halt
  mail
  operator
  games
  ftp
  nobody
  systemd-network
  dbus
  polkitd
  sshd
  postfix
  chrony
  tcpdump
  ntp
  nginx
  
  ```

  > 说明：以“：”为分隔符，打印/etc/passw的第一个字段

- OFS 输出字段分隔符

  ```
  [root@localhost ~]# awk -v FS=':' -v OFS='%' '{print $1,$3}' /etc/passwd
  root%0
  bin%1
  daemon%2
  adm%3
  lp%4
  sync%5
  shutdown%6
  halt%7
  mail%8
  operator%11
  games%12
  ftp%14
  nobody%99
  systemd-network%192
  dbus%81
  polkitd%999
  sshd%74
  postfix%89
  chrony%998
  tcpdump%72
  ntp%38
  nginx%997
  [root@localhost ~]# awk -v FS=':'  '{print $1,$3}' /etc/passwd
  root 0
  bin 1
  daemon 2
  adm 3
  lp 4
  sync 5
  shutdown 6
  halt 7
  mail 8
  operator 11
  games 12
  ftp 14
  nobody 99
  systemd-network 192
  dbus 81
  polkitd 999
  sshd 74
  postfix 89
  chrony 998
  tcpdump 72
  ntp 38
  nginx 997
  
  ```

  

- NF 字段的数量（其是一个int类型的数字），要分清其与$NF

  ```
  [root@localhost ~]# awk -F: '{print NF}' /etc/passwd
  7
  7
  7
  7
  .
  .
  .
  
  
  [root@localhost ~]# awk -F: '{print $NF}' /etc/passwd
  /bin/bash
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /bin/sync
  /sbin/shutdown
  /sbin/halt
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  /sbin/nologin
  
  ```

  > NF表示的是字段的数量（被分隔符分割了几段），而$NF也可视其为最后一个字段

- NR 记录号，一般都为行号（不改变输入记录分割符的情况下）

  ```
  [root@localhost ~]# awk -F: '{print NR}' /etc/passwd
  1
  2
  3
  4
  5
  6
  .
  .
  .
  21
  22
  
  ```

- 自定义变量

  ```
  
  [root@localhost ~]# awk -v tt="hello" '{print tt}' /etc/passwd
  hello
  hello
  hello
  .
  .
  .
  [root@localhost ~]# awk  '{tt="hello";print tt}' /etc/passwd
  hello
  hello
  hello
  .
  .
  .
  ```

  

#### 格式化输出

> %c: 显示字符的ASCII码
>
> %d, %i: 显示十进制整数
>
> %e, %E:显示科学计数法数值
>
> %f：显示为浮点数
>
> %g, %G：以科学计数法或浮点形式显示数值
>
> %s：显示字符串
>
> %u：无符号整数
>
> %%: 显示%自身

左对齐

```
[root@localhost ~]# awk -F: '{printf "%-8s%-8s\n%-8s%-8s\n","Name","UID",$1,$3}' /etc/passwd
Name    UID     
root    0       
Name    UID     
bin     1       
......
```

> 这里打印要用**printf**其中的%-8是左对齐方式，其数字可根据需求调整，当打印字符串时用双引号引起来
>
> 例如： awk -F: '{printf "Username:%-15s UID:%s\n",$1,$3}' /etc/passwd

输出十进制整数

```
[root@localhost ~]# awk -F: '{printf "UID:%d\n",$3}' /etc/passwd
UID:0
UID:1
UID:2
UID:3
UID:4
UID:5
UID:6
......
```

保留到小数位

```
[root@localhost ~]# awk -F: 'BEGIN{printf "%.d %.2f\n",1.73333,3.1111}'
1 3.11

```

> %.f为float类型 其中 %.2f为保留2位小数 ，%.3f为保留3位小数，依次类推

#### 数字运算

```
awk 'BEGIN{print 2^5}'
awk 'BEGIN{print 5%3}'
awk 'BEGIN{n=2;print n+=2}'
```

> 结果为：
>
> 32
>
> 2
>
> 4

#### 模式匹配

```
[root@localhost ~]# awk '$0 ~ "^root" {print $0}' /etc/passwd
root:x:0:0:root:/root:/bin/bash

[root@localhost ~]# awk -v FS=: '$3 == 0 {print $0}' /etc/passwd
root:x:0:0:root:/root:/bin/bash

[root@localhost ~]# awk '$0~/root/' /etc/passwd
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin


```

> ~：左边是否和右边匹配包含
>
> !~：是否不匹配
>
> $0表示打印匹配到的所有，~/../表示正则部分
>
> 注意最后一行命令和倒数第二行的命令，条件后的打印动作可省略，条件成立则打印匹配到的，相反不打印

#### 和、或、非

```
[root@localhost ~]# awk -F: '$3>500 && $3<1000{print $3}' /etc/passwd
999
998
997
```

> 和，两个条件同时满足

```
[root@localhost ~]# awk -F: '$3>500 || $3<1000{print $3}' /etc/passwd
0
1
2
3
4
5
6
7
8
11
12
14
99
192
81
999
74
89
998
72
38
997

```

> 或、两个条件满足一个即可

```
[root@localhost ~]# awk -F: '$3 !=999 {print $3}' /etc/passwd
0
1
2
3
4
5
6
7
8
11
12
14
99
192
81
74
89
998
72
38
997

```

> 非，取反的意思

#### 判断和循环

```
[root@localhost ~]#  awk -F: '{if($3>500){text="Good"}else{text="just so"}{print $3,text}}' /etc/passwd
0 just so
1 just so
2 just so
3 just so
4 just so
5 just so
6 just so
7 just so
8 just so
11 just so
12 just so
14 just so
99 just so
192 just so
81 just so
999 Good
74 just so
89 just so
998 Good
72 just so
38 just so
997 Good

```

> if($3>500){text="Good"} 解读：if $3的内容**大于500**，改if条件成立，则text=Good
>
> else{text="just so"}  if $3的内容**不大于500** 则text=just so

```
[root@localhost ~]# awk -F: '{n=1;while(n<NF){print $3,n++}}' /etc/passwd
0 1
0 2
0 3
0 4
0 5
0 6
......
997 1
997 2
997 3
997 4
997 5
997 6

```

> 解读：
>
> NF为7，意思就是以**:**为分隔符，将一行内容分割为几份，这里为7份
>
> n=1;while(n<NF) 可以理解为 n=1;while(n<7) 那么n的取值范围为1、2、3、4、5、6
>
> print $3,n++}}  $3 这个就是分割后的第三份内容  n++表示n递增 也可以是n+=1

```
[root@localhost ~]# awk -F: '{for(i=1;i<NF;i++)print $3,i}' /etc/passwd
0 1
0 2
0 3
0 4
0 5
0 6
......
```



#### 数组

#### 函数

```shell
[root@localhost ~]# awk 'BEGIN{print length("hi baby")}'  #打印字符的长度
7
[root@localhost ~]# echo "hi baby"|awk 'gsub(/ /,"-",$0)'  #替换
hi-baby
[root@localhost ~]# awk 'BEGIN{system("echo hi baby")}'  #awk中使用shell命令
hi baby
[root@localhost ~]# awk 'BEGIN{me="marry";system("echo hi baby my name is " me)}'
hi baby my name is marry
```

## 磁盘管理

- df
- du
- fdisk

### df

> 检查文件系统的磁盘空间占用情况，可以利用该命令获取磁盘被占用了多少空间，目前还剩余多少空间信息等

#### 语法

```
df [-ahikHTm] [目录或文件名]
```

#### 选项参数

> - -a：列出所有文件系统，包括系统特有的/proc等文件系统
> - -k：以KBytes的容量像是各文件系统
> - -m：以MBytes的容量显示各文件系统
> - -h：以人较容易阅读的GBytes，MBytes，KBytes等格式自行显示
> - -H：以M=1000k取代M=1024K的进位方式
> - -T：显示文件系统类型
> - -i：不用硬盘容量，而以inode的数量来显示

#### 示例

```shell
[root@localhost ~]# df
文件系统                   1K-块     已用     可用 已用% 挂载点
devtmpfs                 1928592        0  1928592    0% /dev
tmpfs                    1940364        0  1940364    0% /dev/shm
tmpfs                    1940364   189544  1750820   10% /run
tmpfs                    1940364        0  1940364    0% /sys/fs/cgroup
/dev/mapper/centos-root 36805060 21078988 15726072   58% /
/dev/sda1                1038336   152852   885484   15% /boot
overlay                 36805060 21078988 15726072   58% /var/lib/docker/overlay2/2aa2499260ea8b5f32cc4b529b9e0c3ed5a604c9fd0045933d7ffef464248a89/merged
overlay                 36805060 21078988 15726072   58% /var/lib/docker/overlay2/6c435a81c23a3a5200bffbb0bc8092bfe7375f28f7927b0fc470b61bfbf94659/merged
shm                        65536        0    65536    0% /var/lib/docker/containers/dd906dafe21762af70aed7a0fb0c8db352249afb8b3b57d6d010bd922b0448ec/mounts/shm
tmpfs                     388076        0   388076    0% /run/user/0
### 在 Linux 底下如果 df 没有加任何选项，那么默认会将系统内所有的 (不含特殊内存内的文件系统与 swap) 都以 1 Kbytes 的容量来列出来！

[root@localhost ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 1.9G     0  1.9G    0% /dev
tmpfs                    1.9G     0  1.9G    0% /dev/shm
tmpfs                    1.9G  186M  1.7G   10% /run
tmpfs                    1.9G     0  1.9G    0% /sys/fs/cgroup
/dev/mapper/centos-root   36G   21G   15G   58% /
/dev/sda1               1014M  150M  865M   15% /boot
overlay                   36G   21G   15G   58% /var/lib/docker/overlay2/2aa2499260ea8b5f32cc4b529b9e0c3ed5a604c9fd0045933d7ffef464248a89/merged
overlay                   36G   21G   15G   58% /var/lib/docker/overlay2/6c435a81c23a3a5200bffbb0bc8092bfe7375f28f7927b0fc470b61bfbf94659/merged
shm                       64M     0   64M    0% /var/lib/docker/containers/dd906dafe21762af70aed7a0fb0c8db352249afb8b3b57d6d010bd922b0448ec/mounts/shm
tmpfs                    379M     0  379M    0% /run/user/0
### 将容量结果以易读的容量格式显示出来

[root@localhost ~]# df -aT
文件系统                类型           1K-块     已用     可用 已用% 挂载点
sysfs                   sysfs              0        0        0     - /sys
proc                    proc               0        0        0     - /proc
devtmpfs                devtmpfs     1928592        0  1928592    0% /dev
securityfs              securityfs         0        0        0     - /sys/kernel/security
tmpfs                   tmpfs        1940364        0  1940364    0% /dev/shm
devpts                  devpts             0        0        0     - /dev/pts
tmpfs                   tmpfs        1940364   189544  1750820   10% /run
tmpfs                   tmpfs        1940364        0  1940364    0% /sys/fs/cgroup
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/systemd
pstore                  pstore             0        0        0     - /sys/fs/pstore
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/cpu,cpuacct
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/devices
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/blkio
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/perf_event
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/freezer
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/hugetlb
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/net_cls,net_prio
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/pids
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/memory
cgroup                  cgroup             0        0        0     - /sys/fs/cgroup/cpuset
configfs                configfs           0        0        0     - /sys/kernel/config
/dev/mapper/centos-root xfs         36805060 21078988 15726072   58% /
systemd-1               -                  -        -        -     - /proc/sys/fs/binfmt_misc
hugetlbfs               hugetlbfs          0        0        0     - /dev/hugepages
debugfs                 debugfs            0        0        0     - /sys/kernel/debug
mqueue                  mqueue             0        0        0     - /dev/mqueue
/dev/sda1               xfs          1038336   152852   885484   15% /boot
overlay                 overlay     36805060 21078988 15726072   58% /var/lib/docker/overlay2/2aa2499260ea8b5f32cc4b529b9e0c3ed5a604c9fd0045933d7ffef464248a89/merged
overlay                 overlay     36805060 21078988 15726072   58% /var/lib/docker/overlay2/6c435a81c23a3a5200bffbb0bc8092bfe7375f28f7927b0fc470b61bfbf94659/merged
shm                     tmpfs          65536        0    65536    0% /var/lib/docker/containers/dd906dafe21762af70aed7a0fb0c8db352249afb8b3b57d6d010bd922b0448ec/mounts/shm
proc                    proc               0        0        0     - /run/docker/netns/e57c3da83d5a
proc                    proc               0        0        0     - /run/docker/netns/b77c91d38026
tmpfs                   tmpfs         388076        0   388076    0% /run/user/0
binfmt_misc             binfmt_misc        0        0        0     - /proc/sys/fs/binfmt_misc
### 将系统内的所有特殊文件格式及名称都列出来

[root@localhost ~]# df -h /etc
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   36G   21G   15G   58% /
### 将 /etc 底下的可用的磁盘容量以易读的容量格式显示

```

### du

> du命令也是查看使用空间的，但是与df命令不通的是du命令是对文件和目录磁盘使用的空间查看

#### 语法

```
du [-ahskm] 文件或目录名称
```

#### 选项参数

> - -a：列出所有的文件与目录容量，因为默认仅统计目录底下的文件量而已
> - **-h：以人们较容易的容量(G/M)显示**
> - **-s：列出总量而已，而不列出每个个别目录占用的使用容量**
> - -S：不包括子目录下的总计，与-s有点差别
> - -k：以KBytes列出容量显示
> - -m：以MBytes列出容量显示
>
> 常用选项组合： du -sh ./*

#### 示例

```shell
[root@localhost data]# du 
136	./openldap/config/cn=config/cn=schema
8	./openldap/config/cn=config/olcDatabase={1}mdb
168	./openldap/config/cn=config
172	./openldap/config
140	./openldap/database
###  当前目录下的目录中每个目录容量
[root@localhost data]# du -a 
4	./openldap/config/cn=config.ldif
4	./openldap/config/cn=config/olcDatabase={-1}frontend.ldif
4	./openldap/config/cn=config/olcDatabase={0}config.ldif
4	./openldap/config/cn=config/cn=schema.ldif
16	./openldap/config/cn=config/cn=schema/cn={0}core.ldif
### 将当前目录下的每个目录中的每个文件的容量列出来
[root@localhost data]# du -sh ./*
312K	./openldap
2.7M	./rootfs
### 列出当前目录下的每个目录的容量   可以使用通配符

```

### fdisk

#### 当前虚拟机信息

![image-20230301111100927](https://gitee.com/root_007/md_file_image/raw/master/202303011111020.png)

![image-20230301133209513](https://gitee.com/root_007/md_file_image/raw/master/202303011332576.png)

#### 添加新磁盘

需要关机添加

![image-20230301133441231](https://gitee.com/root_007/md_file_image/raw/master/202303011334317.png)

![image-20230301133508882](https://gitee.com/root_007/md_file_image/raw/master/202303011335934.png)



![image-20230301133955206](https://gitee.com/root_007/md_file_image/raw/master/202303011339264.png)

#### 对新添加的硬盘进行分区

```shell
[root@localhost ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xde2b0141.

Command (m for help): 
```

> Command (m for help): m 该指令是查看帮助信息
>
>    a   toggle a bootable flag
>    b   edit bsd disklabel
>    c   toggle the dos compatibility flag
>    d   delete a partition
>    g   create a new empty GPT partition table
>    G   create an IRIX (SGI) partition table
>    l   list known partition types
>    m   print this menu
>    n   add a new partition
>    o   create a new empty DOS partition table
>    p   print the partition table
>    q   quit without saving changes
>    s   create a new empty Sun disklabel
>    t   change a partition's system id
>    u   change display/entry units
>    v   verify the partition table
>    w   write table to disk and exit
>    x   extra functionality (experts only)
>
> 常用指令解释：
>
> ```shell
> m: 帮助
> o: 创建msdos分区label
> n: 创建新的分区
> d: 删除分区
> p: 查看当前分区表
> a: 添加/取消 启动标记
> t: 转换分区类型ID
> l/L: 显示分区类型ID表
> ```

创建一个主分区

```shell
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-41943039, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): +5G
Partition 1 of type Linux and of size 5 GiB is set

```

查看刚分配的主分区

```
Command (m for help): p

Disk /dev/sdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xde2b0141

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    10487807     5242880   83  Linux

```

保存分区并推出

```
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

```

系统中查看分区

```
### 刷新一下，或者重启服务器也可以
[root@localhost ~]# partprobe /dev/sdb
###  查看分区
[root@localhost ~]# ll /dev/sd*
brw-rw----. 1 root disk 8,  0 Mar  1 00:38 /dev/sda
brw-rw----. 1 root disk 8,  1 Mar  1 00:38 /dev/sda1
brw-rw----. 1 root disk 8,  2 Mar  1 00:38 /dev/sda2
brw-rw----. 1 root disk 8, 16 Mar  1 00:51 /dev/sdb
brw-rw----. 1 root disk 8, 17 Mar  1 00:51 /dev/sdb1
#### 或者使用该命令查看
[root@localhost ~]# parted /dev/sdb print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 21.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  5370MB  5369MB  primary
### 或者使用 fdisk 查看
[root@localhost ~]# fdisk  -l 

Disk /dev/sdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xde2b0141

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    10487807     5242880   83  Linux


```

#### 格式化磁盘

```
[root@localhost ~]# mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

```

> 根据自己的需求格式化磁盘格式，centos7建议使用xfs、ext4

#### 创建挂载目录

```
### 查看当前系统是否有所需的挂载目录
[root@localhost /]# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
### 创建新的目录作为挂载点  /app目录
[root@localhost /]# mkdir /app
[root@localhost /]# ls
app  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

```

> 挂载目录可以自定义，演示效果而已，创建了/app目录作为挂载点，根据自己的需求进行挂载即可，**不要挂载到已经使用的目录上建议**

#### 将磁盘挂载到目录

```
### 挂载前查看挂载情况
[root@localhost /]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   40G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   39G  0 part 
  ├─centos-root 253:0    0 35.1G  0 lvm  /
  └─centos-swap 253:1    0  3.9G  0 lvm  [SWAP]
sdb               8:16   0   20G  0 disk 
└─sdb1            8:17   0    5G  0 part 
sr0              11:0    1  4.4G  0 rom 
### 进行挂载
[root@localhost /]# mount /dev/sdb1 /app
### 挂载后查看挂载情况
[root@localhost /]# lsblk  
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   40G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   39G  0 part 
  ├─centos-root 253:0    0 35.1G  0 lvm  /
  └─centos-swap 253:1    0  3.9G  0 lvm  [SWAP]
sdb               8:16   0   20G  0 disk 
└─sdb1            8:17   0    5G  0 part /app
sr0              11:0    1  4.4G  0 rom  


```

#### 挂载后查看磁盘空间

```
[root@localhost /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 1.9G     0  1.9G   0% /dev
tmpfs                    1.9G     0  1.9G   0% /dev/shm
tmpfs                    1.9G   12M  1.9G   1% /run
tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   36G  1.7G   34G   5% /
/dev/sda1               1014M  151M  864M  15% /boot
tmpfs                    378M     0  378M   0% /run/user/0
/dev/sdb1                5.0G   33M  5.0G   1% /app

```

#### 卸载挂载

```
[root@localhost app]# umount  /app 
umount: /app: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))

```

> 直接卸载发现，目标正在使用，卸载失败，该情况是该bash在该挂载目录下(/app)

```
[root@localhost app]# cd 
[root@localhost ~]# umount  /app 
[root@localhost ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 1.9G     0  1.9G   0% /dev
tmpfs                    1.9G     0  1.9G   0% /dev/shm
tmpfs                    1.9G   12M  1.9G   1% /run
tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   36G  1.7G   34G   5% /
/dev/sda1               1014M  151M  864M  15% /boot
tmpfs                    378M     0  378M   0% /run/user/0

```

> 当我们切换一个目录后，直接进行卸载，就成功了

还有一种情况，当我们不清楚那个进程正在使用该磁盘时

- 编写一个死循环脚本

  ```
  [root@localhost app]# cat test.sh  
  #!/bin/bash
  i=1
  while true
  do
    echo $i
    let i++
    sleep 1
  done
  
  ```

- 另开一个终端进行卸载挂载

  ```
  [root@localhost ~]# pwd
  /root
  [root@localhost ~]# umount /app
  umount: /app: target is busy.
          (In some cases useful info about processes that use
           the device is found by lsof(8) or fuser(1))
  
  ```

  > 即便切换到非挂载目录，还是卸载不掉

- 使用lsof查看那个进程占用了该目录

  ```
  [root@localhost ~]# lsof  /app
  COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
  bash    1920 root  cwd    DIR   8,17       21   64 /app
  bash    6890 root  cwd    DIR   8,17       21   64 /app
  bash    6890 root  255r   REG   8,17       65   68 /app/test.sh
  
  ```

  > lsof 挂载点，查看在指定挂载点上运行的程序，显示其进程号
  >
  > 或者也可使用 fuser 挂载点 可以查看并杀死在挂载点上执行的程序
  >
  > -v 详细查看
  > -m 递归，如不加m，只查看挂载点自身，不查看子目录
  > -k 杀死
  >
  > -vmk 常用的选项组合

- kill掉占用进程

  ```
  [root@localhost ~]# kill 1920
  [root@localhost ~]# kill 6890
  
  ```

  > 然后尽可以进行卸载了，发现执行的死循环脚本被kill掉了

## 练习题

> 题目1
>
> 姓名	成绩	性别
> zhangsan	90	nan
> mali	70	nv
> wangwu	50	nan
> zhaoliu	30	nan
> lisa	100	nv
> 要求：分别打印男生成绩、女生成绩(只要成绩一列)
>
> 题目2：
> 1 2 3 4 5 6 7 8 9
> 2 3 4 5 6 7 8 9 1
> 3 4 5 6 7 8 9 1 2
> 4 5 6 7 8 9 1 2 3 
> 5 6 7 8 9 1 2 3 4 
> 6 7 8 9 1 2 3 4 5 
> 7 8 9 1 2 3 4 5 6 
> 8 9 1 2 3 4 5 6 7 
> 9 1 2 3 4 5 6 7 8 
> 要求： 打印出如下
> 6 7 8 9 1
> 7 8 9 1 2
> 8 9 1 2 3
> 9 1 2 3 4
>
> 题目3：
>
> 提取出字符串Yd$C@M05MB%9&Bdh7dq+YVixp3vpw中的所有数字
>
> 题目4：
> 利用sed取出ifconfig命令中本机的IPv4地址
>
> 题目5：
> 删除centos7系统/etc/grub2.cfg文件中所有以空白开头的行行首的空白字符