## nginx简单介绍

> Nginx是俄罗斯人lgor Sysoev编写的轻量级web服务器，它不仅是一个高性能的HTTP喝反向代理服务器，同时也是一个IMAP/POP3/SMTP代理服务器
>
> Nginx以事件驱动的方式编写，所以有非常好的性能，同时也是一个非常高效的反向代理、负载平衡服务器。在性能上，Nginx占用很少的系统资源，能支持更多的并发连接，达到更高的访问效率；在功能上，Nginx是优秀的代理服务器和负载均衡服务器；在安装配置上，Nginx安装简单、配置灵活。
>
> Nginx支持热部署，启动速度特别快，还可以在不间断服务的情况下对软件版本或配置进行升级，即使运行数月也无需重新启动。

## 流量请求简单示意图

![image-20230308102752778](https://gitee.com/root_007/md_file_image/raw/master/202303081027855.png)

## 负载均衡介绍

> **负载平衡**（英语：load balancing）是一种[电子计算机](https://zh.wikipedia.org/wiki/电子计算机)技术，用来在多个计算机（[计算机集群](https://zh.wikipedia.org/wiki/计算机集群)）、网络连接、CPU、磁碟驱动器或其他资源中分配负载，以达到最佳化资源使用、最大化吞吐率、最小化响应时间、同时避免过载的目的。 使用带有负载平衡的多个伺服器组件，取代单一的组件，可以通过[冗余](https://zh.wikipedia.org/wiki/冗余)提高可靠性。负载平衡服务通常是由专用软件和硬件来完成。 主要作用是将大量作业合理地分摊到多个操作单元上进行执行，用于解决互联网架构中的[高并发](https://zh.wikipedia.org/wiki/并发性)和[高可用](https://zh.wikipedia.org/wiki/高可用性)的问题。
>
> **常用负载均衡(LB)**：nginx、lvs

## nginx安装

- 服务器环境

  > 系统：CentOS Linux release 7.9.2009 (Core)
  >
  > 内存：4G
  >
  > CPU：4核、

- yum安装nginx

  > 1. 下载nginx的yum资源库 
  >
  >    ```
  >    wget http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
  >    ```
  >
  > 2. 安装该rpm库文件
  >
  >    ```
  >    [root@localhost ~]# rpm -ivh nginx-release-centos-7-0.el7.ngx.noarch.rpm  
  >    warning: nginx-release-centos-7-0.el7.ngx.noarch.rpm: Header V4 RSA/SHA1 Signature, key ID 7bd9bf62: NOKEY
  >    Preparing...                          ################################# [100%]
  >    Updating / installing...
  >       1:nginx-release-centos-7-0.el7.ngx ################################# [100%]
  >    ```
  >
  > 3. 查看该库是否已经安装
  >
  >    ```
  >    [root@localhost ~]# cd /etc/yum.repos.d/
  >    [root@localhost yum.repos.d]# ll
  >    total 44
  >    -rw-r--r--. 1 root root 1664 Oct 23  2020 CentOS-Base.repo
  >    -rw-r--r--. 1 root root 1309 Oct 23  2020 CentOS-CR.repo
  >    -rw-r--r--. 1 root root  649 Oct 23  2020 CentOS-Debuginfo.repo
  >    -rw-r--r--. 1 root root  314 Oct 23  2020 CentOS-fasttrack.repo
  >    -rw-r--r--. 1 root root  630 Oct 23  2020 CentOS-Media.repo
  >    -rw-r--r--. 1 root root 1331 Oct 23  2020 CentOS-Sources.repo
  >    -rw-r--r--. 1 root root 8515 Oct 23  2020 CentOS-Vault.repo
  >    -rw-r--r--. 1 root root  616 Oct 23  2020 CentOS-x86_64-kernel.repo
  >    -rw-r--r--. 1 root root  113 Jul 15  2014 nginx.repo
  >    
  >    ```
  >
  > 4. 安装nginx
  >
  >    ```
  >    [root@localhost yum.repos.d]# yum install -y nginx
  >    Loaded plugins: fastestmirror
  >    Loading mirror speeds from cached hostfile
  >     * base: mirrors.ustc.edu.cn
  >     * extras: mirrors.ustc.edu.cn
  >     * updates: mirrors.huaweicloud.com
  >    base                                                                                                                                                          | 3.6 kB  00:00:00     
  >    extras                                                                                                                                                        | 2.9 kB  00:00:00     
  >    nginx                                                                                                                                                         | 2.9 kB  00:00:00     
  >    updates                                                                                                                                                       | 2.9 kB  00:00:00     
  >    (1/2): nginx/x86_64/primary_db                                                                                                                                |  81 kB  00:00:04     
  >    (2/2): updates/7/x86_64/primary_db                                                                                                                            |  19 MB  00:00:05     
  >    Resolving Dependencies
  >    --> Running transaction check
  >    ---> Package nginx.x86_64 1:1.22.1-1.el7.ngx will be installed
  >    --> Processing Dependency: libpcre2-8.so.0()(64bit) for package: 1:nginx-1.22.1-1.el7.ngx.x86_64
  >    --> Running transaction check
  >    ---> Package pcre2.x86_64 0:10.23-2.el7 will be installed
  >    --> Finished Dependency Resolution
  >    
  >    Dependencies Resolved
  >    
  >    =====================================================================================================================================================================================
  >     Package                                 Arch                                     Version                                              Repository                               Size
  >    =====================================================================================================================================================================================
  >    Installing:
  >     nginx                                   x86_64                                   1:1.22.1-1.el7.ngx                                   nginx                                   797 k
  >    Installing for dependencies:
  >     pcre2                                   x86_64                                   10.23-2.el7                                          base                                    201 k
  >    
  >    Transaction Summary
  >    =====================================================================================================================================================================================
  >    Install  1 Package (+1 Dependent package)
  >    
  >    Total download size: 998 k
  >    Installed size: 3.3 M
  >    Downloading packages:
  >    (1/2): pcre2-10.23-2.el7.x86_64.rpm                                                                                                                           | 201 kB  00:00:02     
  >    (2/2): nginx-1.22.1-1.el7.ngx.x86_64.rpm                                                                                                                      | 797 kB  00:00:04     
  >    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  >    Total                                                                                                                                                229 kB/s | 998 kB  00:00:04     
  >    Running transaction check
  >    Running transaction test
  >    Transaction test succeeded
  >    Running transaction
  >    Warning: RPMDB altered outside of yum.
  >      Installing : pcre2-10.23-2.el7.x86_64                                                                                                                                          1/2 
  >      Installing : 1:nginx-1.22.1-1.el7.ngx.x86_64                                                                                                                                   2/2 
  >    ----------------------------------------------------------------------
  >    
  >    Thanks for using nginx!
  >    
  >    Please find the official documentation for nginx here:
  >    * https://nginx.org/en/docs/
  >    
  >    Please subscribe to nginx-announce mailing list to get
  >    the most important news about nginx:
  >    * https://nginx.org/en/support.html
  >    
  >    Commercial subscriptions for nginx are available on:
  >    * https://nginx.com/products/
  >    
  >    ----------------------------------------------------------------------
  >      Verifying  : pcre2-10.23-2.el7.x86_64                                                                                                                                          1/2 
  >      Verifying  : 1:nginx-1.22.1-1.el7.ngx.x86_64                                                                                                                                   2/2 
  >    
  >    Installed:
  >      nginx.x86_64 1:1.22.1-1.el7.ngx                                                                                                                                                    
  >    
  >    Dependency Installed:
  >      pcre2.x86_64 0:10.23-2.el7                                                                                                                                                         
  >    
  >    Complete!
  >    
  >    ```
  >
  > 5. 检查nginx是否已经安装成功
  >
  >    ```
  >    [root@localhost yum.repos.d]# nginx -v 
  >    nginx version: nginx/1.22.1
  >    
  >    ```

## nginx常用指令

> - **-t**
>
>   不运行，仅仅测试nginx配置文件语法是否正确
>
> - -v
>
>   显示 nginx 的版本。
>
> - -V
>
>   显示 nginx 的版本，编译器版本和配置参数
>
> - -c
>
>   指定nginx配置文件，默认配置文件不用指定
>
> - **-s**
>
>   一般与reload一起使用，表示热更新  **nginx -s reload**

## nginx配置文件概述

- 配置文件位置

  ```shell
  [root@localhost nginx]# pwd
  /etc/nginx
  [root@localhost nginx]# ll
  total 24
  drwxr-xr-x. 2 root root   26 Mar  7 22:01 conf.d
  -rw-r--r--. 1 root root 1007 Oct 19 06:48 fastcgi_params
  -rw-r--r--. 1 root root 5349 Oct 19 06:48 mime.types
  lrwxrwxrwx. 1 root root   29 Mar  7 22:01 modules -> ../../usr/lib64/nginx/modules
  -rw-r--r--. 1 root root  648 Oct 19 06:47 nginx.conf
  -rw-r--r--. 1 root root  636 Oct 19 06:48 scgi_params
  -rw-r--r--. 1 root root  664 Oct 19 06:48 uwsgi_params
  
  ```

  > 其中nginx.conf为该nginx的配置文件

- 配置文件讲解

  ```shell
  [root@localhost nginx]# cat /etc/nginx/nginx.conf 
  
  user  nginx;   ### 表示nginx的启动用户
  worker_processes  auto;    ### 工作进程
  
  error_log  /var/log/nginx/english-error.log notice;  ### 错误日志路径及文件
  pid        /var/run/nginx.pid;   ### pid存放位置
  
  
  events {
      worker_connections  1024;    ### 最大连接数
  }
  
  
  http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;
  
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';    ### 日志格式化
  
      access_log  /var/log/nginx/access.log  main;    ### 访问日志
  
      sendfile        on;
      #tcp_nopush     on;
  
      keepalive_timeout  65;   ### 连接超时时间
  
      #gzip  on;
  
      include /etc/nginx/conf.d/*.conf;   ### 其他配置文件可以存放的位置
  }
  
  ```

- 其他配置文件

  /etc/nginx/conf.d/default.conf

  ```shell
  server {
      listen      80;   ### 监听端口
      server_name  localhost;  ### 绑定的域名、IP
  
      #access_log  /var/log/nginx/host.access.log  main;
  
      location / {    ### 服务
          root   /usr/share/nginx/html;   ### 服务跟目录
          index  index.html index.htm;   ### 加载的文件
      }
  
      #error_page  404              /404.html;
  
      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;   ### 错误码
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  
      # proxy the PHP scripts to Apache listening on 127.0.0.1:80
      #
      #location ~ \.php$ {
      #    proxy_pass   http://127.0.0.1;
      #}
  
      # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
      #
      #location ~ \.php$ {
      #    root           html;
      #    fastcgi_pass   127.0.0.1:9000;
      #    fastcgi_index  index.php;
      #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
      #    include        fastcgi_params;
      #}
  
      # deny access to .htaccess files, if Apache's document root
      # concurs with nginx's one
      #
      #location ~ /\.ht {
      #    deny  all;
      #}
  }
  
  ```

## 启动nginx

- systemctl start nginx

  ```
  [root@localhost ~]# systemctl start nginx 
  [root@localhost ~]# ss -tnl 
  State       Recv-Q Send-Q                                             Local Address:Port                                                            Peer Address:Port              
  LISTEN      0      128                                                            *:80                                                                         *:*                  
  LISTEN      0      128                                                            *:22                                                                         *:*                  
  LISTEN      0      100                                                    127.0.0.1:25                                                                         *:*                  
  LISTEN      0      128                                                         [::]:22                                                                      [::]:*                  
  LISTEN      0      100                                                        [::1]:25                                                                      [::]:*    
  ```

- 访问测试

  ![image-20230308112117936](https://gitee.com/root_007/md_file_image/raw/master/202303081121998.png)

## 修改nginx文件

- /usr/share/nginx/html/index.html

  ![image-20230308133342039](https://gitee.com/root_007/md_file_image/raw/master/202303081333106.png)

- reload一下nginx

  ```
  [root@localhost html]# nginx -s reload 
  ```

- 访问nginx

  ![image-20230308133515673](https://gitee.com/root_007/md_file_image/raw/master/202303081335731.png)

## location讲解

> **语法规则：**
>
> - = 开头表示精确匹配
> - `^~` 开头表示uri以某个常规字符串开头，理解为匹配url路径即可(非正则)
> - `~` 开头表示区分大小写的正则匹配
> - `~*` 开头表示不区分大小写的正则匹配
> - `!~`和`!~*`分别为区分大小写不匹配及不区分大小写不匹配的正则
> - `/` 通用匹配，任何请求都会匹配到

#### 示例

= 精确匹配

```
[root@localhost conf.d]# vim zxl.conf  

server {
    listen       81;
    server_name  localhost;

    access_log  /etc/nginx/conf.d/logs/access_test.log;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    location = /test.html {
        root   /usr/share/nginx/html;
    }

}
```

^~

```
server {
    listen       81;
    server_name  localhost;
    access_log  /etc/nginx/conf.d/logs/access_test.log;
    error_log /etc/nginx/conf.d/logs/error_test.log;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    location = /test.html {
        root   /usr/share/nginx/html;
    }
    location  ^~ /a/ {
        alias   /usr/share/nginx/html/;
        index  a.html a.htm;
    }


}

```

> 其中 usr/share/nginx/html/a.html需要自己创建文件，内容自定义

~

~*

```
    location  ~* \.(gig|jpg|png)$ {
        return 999;
    }

```

![image-20230308142442894](https://gitee.com/root_007/md_file_image/raw/master/202303081425993.png)

!~

/

#### 补充：

> alias指定的路径结尾要加 "/"
>
> - 用root属性指定的值是要加入到最终路径中的，匹配条件会拼接到路径中
> - 用alias属性指定的值不需要加入到最终路径中

## upstream讲解

​	前提：先安装一下golang环境，为学习upstream提供条件

- golang安装

  ```
   wget https://dl.google.com/go/go1.17.2.linux-amd64.tar.gz
   
   tar -zxf go1.17.2.linux-amd64.tar.gz -C /usr/local
   
  cat >> /etc/profile <<EOF
  #go 环境变量
  export GO111MODULE=on
  export GOROOT=/usr/local/go
  export GOPATH=/home/gopath
  export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/usr/local/go/bin:/home/gopath/bin:/usr/local/go/bin:/home/gopath/bin
  EOF
  
  source /etc/profile
  ```

  示例程序：

  https://gitee.com/root_007/study_demo.git

- 基本语法

  upstream的基本语法如下，一个upstream需要设置一个名称，这个名称可以在server里面当作proxy主机使用。

  ```
  upstream default {
      server  服务监听地址:9000;
      server  服务2:
  }
  ```

  > 其中default是需要自定义的，不要重复，一个upstream可以设置多个server

- 常用算法

  > 默认为轮询
  >
  > 加权轮询: weight:权重，权重值越大，访问到该server次数越多
  >
  > ip_hash: 每个请求按访问 IP的hash结果分配，这样来自同一个 IP的访客固定访问一个后端服务器，有效解决了动态网页存在的 session 共享问题。 
  >
  > 最少连接数：least_conn
  >
  > 随机：random

- 示例轮询

  ```
  upstream rr {
          server 127.0.0.1:9090;
          server 127.0.0.1:9091;
  
  }
  
  server {
      listen       82;
      server_name  localhost;
      access_log  /etc/nginx/conf.d/logs/access_test.log;
      error_log /etc/nginx/conf.d/logs/error_test.log;
  
      location /r/ {
      proxy_pass http://rr/;
      }
  }
  
  
  ```

  > 通过curl进行访问验证
  >
  > curl  http://192.168.133.128:82/ping

  ![image-20230308153908685](https://gitee.com/root_007/md_file_image/raw/master/202303081539774.png)

- 示例加权轮询

  ```
  
  upstream wt {
          server 127.0.0.1:9090 weight=9;
          server 127.0.0.1:9091 weight=1;
  
  }
  
  
  server {
      listen       82;
      server_name  localhost;
      access_log  /etc/nginx/conf.d/logs/access_test.log;
      error_log /etc/nginx/conf.d/logs/error_test.log;
  
      location /wt/ {
      proxy_pass http://wt/;
      }
  
  
  }
  
  ```

  > 使用for循环10请求10次
  >
  > ```
  > [root@localhost conf.d]# for i in {1..10};do
  > > curl http://192.168.133.128:82/wt/ping
  > > done
  > 
  > ```

  ![image-20230308155800052](https://gitee.com/root_007/md_file_image/raw/master/202303081558131.png)

- ip_hash示例

  ```
  upstream ih {
          ip_hash;
          server 127.0.0.1:9090 weight=9;
          server 127.0.0.1:9091 weight=1;
  
  }
  
  ```

## proxy讲解

- proxy_pass

  > 转发代理请求

- proxy_set_head

  > 修改或设置请求头的值，如果值为空字符串，那么这个字段将不会被转发到代理服务器

- proxy_pass_request_headers

  > 指定是否将原始的请求头转发到后端服务，如果需要忽略到所有的原始请求头，可以关闭该值

- proxy_ignore_headers

  > 告知nginx后端服务的响应中的哪些响应头不要去被处理

- proxy_hide_header

  > 在nginx将请求的响应结果给客户端的时候，将后端响应的一些请求头过滤掉

- proxy_pass_header

  > 允许其中某个响应头传递给客户端，与proxy_hide_header是相反的

- proxy_buffering

  > 控制本内容是否对后端服务的响应启用缓冲

- proxy_buffers

  > 有两个参数，第一个是控制缓冲区请求数量，第二个控制缓冲区大小，这个值越大，缓冲的内容越多

- proxy_buffer_size

  > 后端回复结果的首段(包含header部分)是单独缓冲的，该条目定义了这部分缓冲区大小，这个值默认与proxy_buffer的值相同

- proxy_busy_buffers_size

  > 客户端一次只能从一个缓冲区中读取数据，而缓冲是按照队列次序发送给客户端的，该项设置的值就是这个队列的大小

  常用proxy组合

  ```shell
  upstream test1{
  server 127.0.0.1:8000;
  }
  upstream test2{
  server 127.0.0.1:8000;
  }
  server{
  	server_name  test.com;
  	listen 80;
         access_log  /etc/nginx/conf.d/logs/access_test.log;
         error_log /etc/nginx/conf.d/logs/error_test.log;
          proxy_set_header   Host             $host;
          proxy_set_header   X-Real-IP        $remote_addr;
          proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
          proxy_connect_timeout   3s;
          proxy_read_timeout 120s;
          proxy_send_timeout 120s;
  	
          location /user/ {
  		proxy_set_header Connection "";
          	proxy_http_version 1.1;
  		proxy_pass http://test1/;
  		}
          location / {
                  proxy_set_header Connection "";
                  proxy_http_version 1.1;
                  proxy_pass http://test2/;
          }
  }
  
  ```

## nginx常用内置变量

| 名称                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| $remote_addr          | 客户端地址                                                   |
| $server_name          | 请求到达的服务器名;                                          |
| $host                 | 请求信息中的"Host"，如果请求中没有Host行，则等于设置的服务器名; |
| $http_x_forwarded_for | 相当于网络访问路径                                           |

更多参数请参考：https://note.youdao.com/s/SDV25MaS

## nginx使用https

​	需要提前安装openssl

​	yum install -y openssl openssl-devel

- 手动生成证书

  ```
  [root@localhost ssl]# openssl genrsa -out cakey.pem 4096
  Generating RSA private key, 4096 bit long modulus
  .............................................................................++
  ................................................................................................................................................................................++
  e is 65537 (0x10001)
  ```

  ```
  [root@localhost ssl]# openssl req -new -x509 -key cakey.pem -out cacert.pem -days 3650
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [XX]:CN
  State or Province Name (full name) []:HN  
  Locality Name (eg, city) [Default City]:ZZ
  Organization Name (eg, company) [Default Company Ltd]:zdy
  Organizational Unit Name (eg, section) []:devops
  Common Name (eg, your name or your server's hostname) []:zxlabc.devops.com
  Email Address []:
  
  ```

  > 需要填写的内容：
  >
  > Country Name (2 letter code) [XX]:CN       # **国家**
  > State or Province Name (full name) []:HN  # **省**
  > Locality Name (eg, city) [Default City]:ZZ  # **城市**
  > Organization Name (eg, company) [Default Company Ltd]:zdy  # **企业**
  > Organizational Unit Name (eg, section) []:devops  # **部门**
  > Common Name (eg, your name or your server's hostname) []:zxlabc.devops.com  # **域名**
  > Email Address []:  # **申请人邮箱，可以为空**

- 使用脚本生成证书

  脚本位置截图

  ![image-20230309112848910](https://gitee.com/root_007/md_file_image/raw/master/202303091128999.png)

  脚本地址： https://gitee.com/root_007/study_demo.git

  - 使用脚本生成证书

    ```shell
    [root@localhost ssl]# chmod  +x gen.cert.sh 
    ```

    ```shell
    [root@localhost ssl]# ./gen.cert.sh zxlabc.devops.com
    Removing dir out
    Creating output structure
    Done
    Generating a 2048 bit RSA private key
    .........................................................................................+++
    ..............................................................................+++
    writing new private key to 'out/root.key.pem'
    -----
    Generating RSA private key, 2048 bit long modulus
    ........................................................................+++
    ..................................+++
    e is 65537 (0x10001)
    Using configuration from ./ca.cnf
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    countryName           :PRINTABLE:'CN'
    stateOrProvinceName   :ASN.1 12:'Guangdong'
    localityName          :ASN.1 12:'Guangzhou'
    organizationName      :ASN.1 12:'Fishdrowned'
    organizationalUnitName:ASN.1 12:'zxlabc.devops.com'
    commonName            :ASN.1 12:'*.zxlabc.devops.com'
    Certificate is to be certified until Mar  8 03:15:41 2025 GMT (730 days)
    
    Write out database with 1 new entries
    Data Base Updated
    
    Certificates are located in:
    lrwxrwxrwx. 1 root root 11 Mar  8 22:15 /root/ssl/ssl/out/zxlabc.devops.com/root.crt -> ../root.crt
    lrwxrwxrwx. 1 root root 44 Mar  8 22:15 /root/ssl/ssl/out/zxlabc.devops.com/zxlabc.devops.com.bundle.crt -> ./20230308-2215/zxlabc.devops.com.bundle.crt
    lrwxrwxrwx. 1 root root 37 Mar  8 22:15 /root/ssl/ssl/out/zxlabc.devops.com/zxlabc.devops.com.crt -> ./20230308-2215/zxlabc.devops.com.crt
    lrwxrwxrwx. 1 root root 15 Mar  8 22:15 /root/ssl/ssl/out/zxlabc.devops.com/zxlabc.devops.com.key.pem -> ../cert.key.pem
    ```

    证书生成后目录

    ```
    [root@localhost ssl]# ls
    ca.cnf  docs  flush.sh  gen.cert.sh  gen.root.sh  LICENSE  out  README.md
    
    ```

    证书位置

    ```
    /root/ssl/ssl/out/cert.key.pem
    /root/ssl/ssl/out/zxlabc.devops.com/zxlabc.devops.com.crt
    ```

- nginx加载证书

  拷贝证书到自定义目录(/etc/nginx/ssl),**需要自己手动创建该目录哦**

  ```shell
  [root@localhost ssl]# cp  /root/ssl/ssl/out/cert.key.pem  /etc/nginx/ssl/ert.key.pem  -a
  c[root@localhost ssl]# p /root/ssl/ssl/out/zxlabc.devops.com/zxlabc.devops.com.crt  /etc/nginx/ssl/zxlabc.devops.com.crt  -a
  ```

  配置nginx的conf文件

  ```
  server {
      listen       443 ssl;
      server_name  zxlabc.devops.com;
      ssl_certificate_key  ssl/cert.key.pem;
      ssl_certificate ssl/zxlabc.devops.com.crt;
  
  
      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }
  
  
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  
  }
  
  ```

  > 证书：
  >
  > sl_certificate_key  ssl/cert.key.pem;
  >
  > sl_certificate ssl/zxlabc.devops.com.crt;
  >
  > 域名：
  >
  > erver_name  zxlabc.devops.com;
  >
  > 注意：该域名要和生成证书的域名一致，我们用的不是泛域名证书

  修改hosts访问配置的域名

  ```
  [root@localhost conf.d]# vim /etc/hosts
  192.168.133.128  zxlabc.devops.com
  ```

  > 192.168.133.128  该IP是虚拟机的IP

  测试访问：

  ![image-20230309133322548](https://gitee.com/root_007/md_file_image/raw/master/202303091333600.png)

## nginx跨域配置

​	

```
server {
    listen 443;
    listen 80;
    server_name b.xxx.com;
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Credentials true;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE';
    add_header 'Access-Control-Allow-Headers' 'Content-Type';
    location / {
        proxy_pass http://192.168.1.212:8136;
        #proxy_pass http://xd-mfa-mng/;
        include nginx_proxy.conf;
    }
    error_page   500 502 503 504  /502.html;
    location = /50x.html {
        oot   html;
    }
}
```

> 解决跨域问题需要添加字段：
>
> 1. `add_header 'Access-Control-Allow-Origin' '*';`
> 2. `add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE';`
> 3. `add_header 'Access-Control-Allow-Headers' 'Content-Type';`
> 4. `add_header 'Access-Control-Allow-Credentials' 'true';`