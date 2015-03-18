---
title: "软件开发管理环境配置-中小团队"
description: "Setup gitlab server and redmine for small team"
category: "development"
tags: [Project Management, Configuration Management, git, ]
---
{% include JB/setup %}

一.概述
=======
- 用户管理使用LDAP管理， 选用产品ApacheDS，安装在ServerLDAP
- 文档和代码管理使用Git， 选用产品GitLAB， 安装在ServerCM， 并和LDAP集成
- 项目管理和Bug管理使用Redmine， 安装在ServerCM，并和LDAP集成
- 项目中使用FTP管理大型文件


二.具体安装配置
=============

2.1 LDAP配置
-------------

1. 安装JRE
2. 下载 ApacheDS-2.0.9, 并安装

       缺省端口：10389
       管理员信息：uid=admin,ou=system, 密码：secret （ApacheDS缺省设置）
       也可以另外添加一个管理员帐号： uid=useradmin,ou=system, 密码：xxxxxx (自己随意指定）

3. 下载 Apache Directory Studio， 添加

       ou=employees,dc=hello,dc=com
       uid=test1,ou=employees,dc=hello,dc=com
       uid=test2,ou=employees,dc=hello,dc=com

2.2 GitLAB 配置
----------------
1. 安装 Debian7 OS on ServerCM： 下载iso文件 , 用UltraISO写到U盘，然后用U盘启动即可安装即可
2. 下载 gitlab rpm 包, 安装到 /opt/gitlab

       运行环境： http://ServerCM
       记住Gitlab 超级管理员密码： root/xxxxxx （自己指定）

3. 配置 /etc/gitlab/gitlab.rb， 使用 ldap accounts:
       
       main: # 'main' is the GitLab 'provider ID' of this LDAP server
          label: 'ldap-team'
          host: 'ServerLDAP'
          port: 10389
          uid: 'uid'
          method: 'plain' # "tls" or "ssl" or "plain"
          bind_dn: 'uid=useradmin,ou=system'
          password: 'xxxxxx'
          active_directory: false
          allow_username_or_email_login: false
          base: 'ou=employees,dc=hello,dc=com'
        
4. Configurate /etc/gitlab/gitlab.rb to use SMTP to send notification email

       gitlab_rails['smtp_enable'] = true
       gitlab_rails['smtp_address'] = "smtpserver.com.cn"     #修改成所需SMTP环境
       gitlab_rails['smtp_port'] = 25
       gitlab_rails['smtp_user_name'] = "aaaaaa@hello.com"
       gitlab_rails['smtp_password'] = "xxxxxxx"
       gitlab_rails['smtp_domain'] = "hello.com"
       gitlab_rails['smtp_authentication'] = "login"

2.3 Redmine 配置
----------------
1. Install mysql

       修改root密码， 并创建redmine用户（mysql用户） -->The user is for redmine web application.

2. Deployment redmine to /opt/redmine3

       记住redmine admin用户密码

3. Install unicorn and nginx to power redmine （也可使用Passenger等其他Server）

       运行环境： http://ServerCM:3000 , --> 具体配置请参见Unicorn和Nginx集成文档

4. 配置LDAP， 利用admin用户登录Redmine的页面，进行配置即可


2.4 LDAP-Account-Manager配置
-----------------------------------

为了方便团队成员添加用户（例如新员工加入等）， 使用ldap-account-manager管理ApacheDS的员工帐号

1. 安装ldap-account-manager,lam to manage LDAP accounts
   - install php
   - download lam tar.bz 包， 解压缩到/opt/lam

2. 在 nginx 添加一个Site，监听在 3300端口
   - 直接在lam页面上添加一个LDAP Profile，在页面上配置即可


2.5 Self LDAP Change Password
-----------------------------------

为了方便员工自己修改密码，部署LTB - Self Service Prassword

1. 选择LTB - Self Service Prassword，  部署到Nginx， 和LAM 使用同一个文件， /passwd Location

        location /lam {
                index index.html;
                alias /opt/lam;
                autoindex off;

                location ~ \.php$ {
                        fastcgi_split_path_info ^(.+\.php)(/.+)$;
                        fastcgi_pass unix:/var/run/php5-fpm.sock;
                        fastcgi_index index.php;
                        include fastcgi_params;
                }

                location ~ /lam/(tmp/internal|sess|config|lib|help|locale) {
                        deny all;
                        return 403;
                }
        }

        location /passwd {
                index index.php;
                alias /opt/self_ldap_passwd;
                autoindex off;
                location ~ \.php$ {
                        fastcgi_split_path_info ^(.+\.php)(/.+)$;
                        fastcgi_pass unix:/var/run/php5-fpm.sock;
                        fastcgi_index index.php;
                        include fastcgi_params;
                }

        }

2. 编辑/opt/self_ldap_passwd/conf/confic.inc.php 中的 LDAP 连接信息

       $ldap_url = "ldap://ServerLDAP:10389";
       $ldap_binddn = "uid=useradmin,ou=system";
       $ldap_bindpw = "xxxxxx";
       $ldap_base = "dc=hello,dc=com";
       $ldap_login_attribute = "uid";


2.6 FTP配置
---------------

项目中的大文件签入Git会造成git速度下降， 所以大的二进制文件使用FTP管理

1. 下载FileZilla Server, 安装
2. 配置FTP Server

     可以下载FileZilla Server Inferface（GUI Tool）来配置FileZilla FTP Server
     连接信息： 127.0.0.1  端口：14147， 密码：xxxxx （自己指定）
	 
     [*Troubleshooting*]
	 
     使用FileZilla Server 作为FTP服务器， 刚开始login和list非常慢，
     我的尝试解决办法：
	 
     1. 首先使用服务器管理器的“windows高级安全防火墙”添加端口， 没有解决问题
     2. 使用控制面板中的windows防火墙吧“FileZilla Server”和“FileZilla Admin Server 都加进去， 然后全部启用，就好了， 记录一下， 以后备查。

ftp的根目录 d:\ftp_home

     权限配置为： 所有人匿名可读可上传， 可匿名上传到临时目录， 由文档管理人员整理到正式目录
     临时目录：uploadings,   -->供用户零时上传的目录，
     正式目录：projects， dept_documents， tools， softwares


2.7 其他
--------

其他可以配置， Code Review  （To be done)


