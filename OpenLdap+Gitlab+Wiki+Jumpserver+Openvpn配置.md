## OpenLdap 对接内部系统(Gitlab+Wiki+Jumpserver+Openvpn)配置

LDAP 全称轻量级目录访问协议（英文：Lightweight Directory Access Protocol），是一个运行在 TCP/IP 上的目录访问协议。目录是一个特殊的数据库，它的数据经常被查询，但是不经常更新。其专门针对读取、浏览和搜索操作进行了特定的优化。目录一般用来包含描述性的，基于属性的信息并支持精细复杂的过滤能力。比如 DNS 协议便是一种最被广泛使用的目录服务。

LDAP 中的信息按照目录信息树结构组织，树中的一个节点称之为条目（Entry），条目包含了该节点的属性及属性值。条目都可以通过识别名 dn 来全局的唯一确定1，可以类比于关系型数据库中的主键。比如 dn 为 uid=ada,ou=People,dc=xinhua,dc=org 的条目表示在组织中一个名字叫做 Ada Catherine 的员工，其中 uid=ada 也被称作相对区别名 rdn。

一个条目的属性通过 LDAP 元数据模型（Scheme）中的对象类（objectClass）所定义，下面的表格列举了对象类 inetOrgPerson（Internet Organizational Person）中的一些必填属性和可选属性。

OpenLDAP 是 LDAP 协议的一个开源实现。LDAP 服务器本质上是一个为只读访问而优化的非关系型数据库。它主要用做地址簿查询（如 email 客户端）或对各种服务访问做后台认证以及用户数据权限管控。（例如，访问 Samba 时，LDAP 可以起到域控制器的作用；或者 Linux 系统认证 时代替 /etc/passwd 的作用。）

#### 1、LDAP 术语

Entry (or object) 条目(或对象):LDAP中的每个单元都认为是条目。

dn:条目名称。

ou:组织名称。

dc:域组件。例如，likegeeks.com是这样写的:dc=likegeeks,dc=com。

cn:通用名称，如人名或某个对象的名字



#### 2、使用Docker部署启动Openldap
```
# 环境配置：
OpenLdap服务器地址：100.111.21.68

# 创建文件夹：
mkdir -p /data/openldap/{config,database}

# 拉取openldap镜像
shell > docker pull osixia/openldap:1.2.2

# 启动openldap服务
shell> docker run -d --name ldap-service --hostname ldap-service -p 389:389 -p 689:689 -v /data/openldap/database:/var/lib/ldap -v /data/openldap/config:/etc/ldap/slapd.d --env LDAP_ORGANISATION="zypnote.com" --env LDAP_DOMAIN="zypnote.com" --env LDAP_ADMIN_PASSWORD="zyppassword" --env LDAP_TLS=false --detach osixia/openldap:1.2.2

# 启动phpldapadmin图形管理工具
shell > docker pull osixia/phpldapadmin:0.7.2

shell > docker run --name phpldapadmin-service -p 6443:443 -p 6680:80 --hostname phpldapadmin-service --link ldap-service:zypnote.com --env PHPLDAPADMIN_LDAP_HOSTS=zypnote.com --env PHPLDAPADMIN_HTTPS=false --detach osixia/phpldapadmin:0.7.2


Docker镜像地址：https://github.com/osixia/docker-openldap


```
#### 3、访问phpldapAdmin：
```
http://100.111.21.68:6680

账号：cn=admin,dc=zypnote,dc=com
密码：zyppassword


```

----- 

## OpenLdap+Gitlab配置
```
环境说明：
GitLab: 10.2.5
OpenLdap: 1.2.2

```
#### 1、修改Gitlab配置文件
```
shell> vim /home/git/gitlab/config/gitlab.yml

ldap:
enabled: true
servers:
    label: 'LDAP'
    host: '100.111.21.68'
    port: 389
    uid: 'cn'
    encryption: 'plain' # "start_tls" or "simple_tls" or "plain"
    verify_certificates: false
    ca_file: ''
    ssl_version: ''

    bind_dn: 'cn=admin,dc=zypnote,dc=com'
    password: 'zyppassword'
    timeout: 10
    active_directory: false
    allow_username_or_email_login: true
    block_auto_created_users: false
    base: 'dc=zypnote,dc=com'
    user_filter: ''
    attributes:
      username: ['cn', 'uid', 'userid', 'sAMAccountName']
      email: ['mail', 'email', 'userPrincipalName']
      name:       'cn'
      first_name: 'givenName'
      last_name:  'sn'


配置参数说明：
host：ldap服务器地址
port：ldap服务端口
uid：以哪个属性作为验证属性，可以为uid、cn等，我们使用uid
method：如果开启了tls或ssl则填写对应的tls或ssl，都没有就填写plain
bind_dn：search搜索账号信息的用户完整bind（需要一个有read权限的账号验证通过后搜索用户输入的用户名是否存在）
password：bind_dn用户的密码，bind_dn和password两个参数登录ldap服务器搜索用户
active_directory：LDAP服务是否是windows的AD，我们是用的openldap，这里写false
allow_username_or_email_login：是否允许用户名或者邮箱认证，如果是则用户输入用户名或邮箱都可
base：从哪个位置搜索用户，例如允许登录gitlab的用户都在ou gitlab里，name这里可以写ou=gitlab,dc=domain,dc=com
filter：添加过滤属性，例如只过滤employeeType为developer的用户进行认证（employeeType=developer）
重启gitlab服务，看到页面已经有ldap的登录选项了


```


#### 2、配置phpldapadmin
```
1.在浏览器中打开http://IP/phpldapadmin

2. 点击【Login】按钮，输入管理员密码。

3.点击【创建新条目】.

4. 点击【Generic: Postfix Group】.

5. 输入【Users】, 点击【创建对象】,创建一个组

6. 点击【提交】

7. 下一步添加用户，点击刚才所创建的组【users】

8. 点击【创建一个子条目】

9. 点击【Generic: User Account】按钮。

10.根据自己的情况，添加信息然后点击【创建对象】

11. 点击【提交】

12.提交完成后，点击新增的用户，点击右侧【增加新的属性】

13.选择属性【Email】

14. 添好Email地址

15.点击【Update Object】

参考连接：https://www.cnblogs.com/xiaomifeng0510/p/9564688.html

```


---------
## Openldap+Confluence(wiki)

#### 1、使用管理员登录Confluence
- 依次点击：一般配置-->用户&安全-->用户目录–添加目录-->LDAP-->Openldap
```
名称：LDAP服务器
目录类型：OpenLDAP
主机名：100.111.21.68
端口：389
账号：cn=admin,dc=zypnote,dc=com
密码：zyppassword


基础DN：dc=zypnote,dc=com
附加用户DN：不用写
附加组DN：不用写

- LDAP权限：只读，且为本地组
（从LDAP服务器上检索到的用户、用户组及成员，且无法在Confluence中修改。你可以将LDAP的用户添加到维护在Confluence内部目录的用户组中。）

默认组成员：confluence-users
（首次登陆系统后，将添加的组成员列表，且每个成员以逗号分开。如果不存在该组，则会自动创建这个组。这里意思是openldap的用户登录confluence的时候默认加入到哪个组里面，confluence-users组是confluence的默认普通用户权限组，登录后会自动加入到这个组里面）

- 设置用户模式
用户名属性：cn
用户名RDN属性：cn
用户名字属性：givenName
用户姓氏属性：sn
用户显示名属性：displayName
用户邮箱: Email
用户密码属性: Password
用户密码加密：MD5


```

#### 2、配置完成后，点击快速测试

```
- 如果测试成功，会提示：
测试成功 
这里仅测试此服务器可连接且认证信息正确。保存设置后您可点击 浏览目录 页面的 “测试” 连接可以进行更全面的测试.

```


-----
## OpenLDAP+Jumpserver
#### 1、登录jumpserver设置ldap
```
- 系统设置 - LDAP设置
LDAP地址：ldap://100.111.21.68:389
绑定DN：cn=admin,dc=zypnote,dc=com
密码：zyppassword
用户OU：dc=zypnote,dc=com
用户过滤器：(cn=%(user)s)
LDAP属性映射：{"username": "cn", "name": "sn", "email": "mail"}
启用LDAP认证：打勾

```
#### 2、配置完成后，重启jumpserver
- 重启jumpserver容器
- 使用ldap用户直接登录系统

---------------------------

## Openldap+Openvpn

#### 环境：
- openvpn-2.4.6
- openvpn-auth-ldap-2.0.3
- 

1、安装openvpn需要的依赖包
```
yum install openvpn-auth-ldap


```
2、编辑配置文件
```
#编辑ldap文件
vi /etc/openvpn/auth/ldap.conf

<LDAP>
        URL             ldap://100.111.21.68:389
        BindDN          cn=admin,dc=zypnote,dc=com
        Password        zyppassword
        Timeout         15
        TLSEnable       no
        FollowReferrals no
</LDAP>

<Authorization>
        BaseDN          "dc=zypnote,dc=com"
        SearchFilter    "cn=%u"
        RequireGroup    false

        <Group>
                BaseDN          "cn=users,dc=zypnote,dc=com"
                SearchFilter    "cn=vpn"
                MemberAttribute memberUid
        </Group>
</Authorization>



# 编辑openvpn的配置文件/etc/openvpn/server.conf：
shell> vim /etc/openvpn/server.conf

port 1194
proto tcp
dev tun
ca keys/ca.crt
cert keys/vpn01.zypnote.com.crt
key keys/vpn01.zypnote.com.key  # This file should be kept secret
dh keys/dh2048.pem
server 100.15.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.0.0 255.0.0.0"
push "route 100.64.0.0 255.192.0.0"
client-to-client
keepalive 10 120
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
log         /var/log/openvpn/openvpn.log
log-append  /var/log/openvpn/openvpn.log
verb 4

plugin /usr/lib64/openvpn/plugin/lib/openvpn-auth-ldap.so "/etc/openvpn/auth/ldap.conf"
client-cert-not-required



```
3、客户端的配置调整如下：
```

client
dev tun
proto tcp
remote 61.20.94.11 1194
# nobind
persist-key
persist-tun

<ca>
-----BEGIN CERTIFICATE-----
MIIEsDCCA5igAwIBAgIJAOToyUe9fEooMA0GCSqGSI
MQ0wCwYDVQQLEwR0ZWNoMRMwEQYDVQQDEwp2aWRlb2
EQfFmb3QRtwjazDHgaXjoNc9FyVPsUY3iS7wnikXle4wS/rpdjBZWnw4XACnrpyo
2j6s1WkgSTx0h43MbKmipHbe9WRfdVO5MC72KMf7VSXZDCeYE5o8v45H4UcUtxup
MQ0wCwYDVQQLEwR0ZWNoMRMwEQYDVQQDEwp2aWRlb2
EQfFmb3QRtwjazDHgaXjoNc9FyVPsUY3iS7wnikXle4wS/rpdjBZWnw4XACnrpyo
2j6s1WkgSTx0h43MbKmipHbe9WRfdVO5MC72KMf7VSXZDCeYE5o8v45H4UcUtxup
MQ0wCwYDVQQLEwR0ZWNoMRMwEQYDVQQDEwp2aWRlb2
EQfFmb3QRtwjazDHgaXjoNc9FyVPsUY3iS7wnikXle4wS/rpdjBZWnw4XACnrpyo
2j6s1WkgSTx0h43MbKmipHbe9WRfdVO5MC72KMf7VSXZDCeYE5o8v45H4UcUtxup
MQ0wCwYDVQQLEwR0ZWNoMRMwEQYDVQQDEwp2aWRlb2
EQfFmb3QRtwjazDHgaXjoNc9FyVPsUY3iS7wnikXle4wS/rpdjBZWnw4XACnrpyo
2j6s1WkgSTx0h43MbKmipHbe9WRfdVO5MC72KMf7VSXZDCeYE5o8v45H4UcUtxup
7B47Mg==
-----END CERTIFICATE-----
</ca>

# ns-cert-type server

auth-user-pass
remote-cert-tls server
verb 4

```
- auth-user-pass是新加入的配置开启了用户名密码认证


---------------------------------
#### 配置过程中遇到的问题
```
如果报如下错误

因为 Undefined method `provider' for nil:nilclass，所以您无法从 Ldapmain 获得授权。
(也有网友登录的时候报 Could not authorize you from LDAP because “(ldap) account must provide a dn,uid and email address”)

这是因为Gitlab要求有email属性
所以需要添加email

登录phpldapadmin管理，http://ldap-server/phpldapadmin/
给用户添加email属性即可


```

