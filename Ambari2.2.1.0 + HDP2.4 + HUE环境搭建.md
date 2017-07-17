### 一、前言
* 数据团队Hadoop集群目前只有线上环境，测试缺失使得有些流程上不通，例如ETL测试环境、平常依赖HDP组件的开发等等。以及后续线上集群升级测试环境可以预演。
* Ambari采用Apache的版本 [Apache Ambari文档地址](https://cwiki.apache.org/confluence/display/AMBARI/Install+Ambari+2.2.1+from+Public+Repositories)，文档上Ubuntu 14的源地址有误，切记不要`右击`选择复制链接地址，得到的url为2.2.0.0的版本，直接选择复制。同时HDP也有集成，两者差别不大，也可以参考HDP的文档来安装。[HDP Ambari文档地址](https://docs.hortonworks.com/HDPDocuments/Ambari-2.2.1.0/bk_Installing_HDP_AMB/content/ch_Getting_Ready.html)。此次安装参考Apache的文档，比较简单。
* 系统环境：Ubuntu14 + HDP2.4 + Amabri 2.2.1，查看系统信息命令`lsb_release  -a`。
	
	```
	No LSB modules are available.
	Distributor ID:	Ubuntu
	Description:	Ubuntu 14.04.5 LTS
	Release:	14.04
	Codename:	trusty
	```
* 全文都是以root账号进行操作。	

### 二、准备

* 安装mysql
	* ambari、hive、hue都有默认的依赖库，为了统一都改成mysql。连接客户端，创建相关库、用户已经设置权限。
	* `mysql -uroot -proot`登陆。
	* ambari mysql db/user create &nbsp;&nbsp;  [参考](https://docs.hortonworks.com/HDPDocuments/Ambari-2.5.1.0/bk_ambari-administration/content/using_ambari_with_mysql.html)
		
		```
		CREATE DATABASE ambari;
 		
 		CREATE USER 'ambari'@'%' IDENTIFIED BY 'ambari';
 		
 		GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%';
 		
 		CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'ambari';
 		
	 	GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost';
	 	
	 	FLUSH PRIVILEGES;
		
		```
	* hive mysql db/user create &nbsp;&nbsp; [参考](https://docs.hortonworks.com/HDPDocuments/Ambari-2.5.1.0/bk_ambari-administration/content/using_hive_with_mysql.html)
		
		```
		CREATE DATABASE hive;
 		
 		CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
	 	
	 	GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%';
	 	
	 	FLUSH PRIVILEGES;
		```
	
	* hue mysql db/user create
		
		```
		CREATE DATABASE hue;
 		
 		CREATE USER 'hue'@'%' IDENTIFIED BY 'hue';
 		
 		GRANT ALL PRIVILEGES ON *.* TO 'hue'@'%';
 		
 		FLUSH PRIVILEGES;
		```	
* `ssh 免密登陆`，配置如下：
	* Ubuntu 默认已安装了SSH client，此外还需要安装 SSH server，如果SSH server已安装忽略此步骤。`apt-get install openssh-server`
	* 执行`ssh localhost`，此时会让你输入账号密码，到这一步退出，目的是为了自动创建`~/.ssh`文件夹。
	* `cd ~/.ssh/`
	* `ssh-keygen -t rsa` 提示让你输入的信息都按回车即可。
	* `cat ./id_rsa.pub >> ./authorized_keys`  加入授权。
	* 至此，免密已完成。 可以使用`ssh localhost`验证。
	* 如果需要root ssh免密到各个账号以及机器之间互相免密登陆，需要将各自的公钥`id_rsa.pub` copy到各个账号的~/.ssh下的`authorized_keys`文件中，`authorized_keys`不存在则创建一个。

* 搭建私有源，因hortonworks虽然提供了官方源，但国内到外网的网速堪忧，需要搭建自己私有的源。[官方文档地址](https://docs.hortonworks.com/HDPDocuments/Ambari-2.2.1.0/bk_Installing_HDP_AMB/content/_using_a_local_repository.html)，感谢运维同学~

	
### 三 、安装

1. Ambari
	* 将私有源Ambari的ambari.list（`deb http://xxx/hdp2.4.0.0/xxx/2.2.1.0/ Ambari main`）下载地址替换Apache版本写入 `/etc/apt/sources.list.d/ambari.list`文件中。
	* `apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD`
	* `apt-get update`
	* `apt-get install ambari-server`
	* `ambari-server setup`
	
		**setup 这一步主要是设置服务启动的账号，以及依赖的database、jdk等。可以全部选择默认的选项，此次安装除了database之外其他的都选择默认的配置。database信息已经在准备工作中初始化好了，此外还需要使用ambari账号登陆mysql切换到ambari库中，将ambari数据脚本初始化。执行sql语句：SOURCE /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql;**
		
	 	![图1](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/ambari-setup.png)
	* 如果需要修改Ambari访问端口，`vim /etc/ambari-server/conf/ambari.properties` 添加`client.api.port=8090`
	* `ambari-server start`
	* [访问](http://172.17.41.241:8090) (账号：`admin`  密码：`admin`)
	
2. HDP
   * 按照Ambari添加集群向导，新增集群名称。
   * 下一步，Select Stack, 选择安装的HDP版本以及修改官网源为私有源。
   ![图2](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install1.png)
   * 下一步，添加需要安装HDP机器列表的HOST，可以是多个，本次因为节点只有一个所以只有一个。添加Ambari服务所在机器的root账号的私钥。
   ![图3](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install2.png)
   * 下一步，Ambari Server会自动安装 Ambari Agent 到刚才指定的机器列表。安装完成后，Agent 会向Ambari Server注册。
   ![图4](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install3.png)
   * 此次注册的时候出现了警告信息，虽然可以忽略，但尽量去消除这些警告，以免后患，主要碰到两个：
   		* Transparent Huge Pages Issues（这一项才准备工作的时候已经完成，所以警告不在出现，这是在知道情况才会出现，在首次安装的时候不注意这个一定会有这个警告）；
   		
   		 ```
   		此项警告需要关闭Transparent HugePages.
   		执行：echo never > /sys/kernel/mm/transparent_hugepage/enabled
			 echo never > /sys/kernel/mm/transparent_hugepage/defrag
   		参考：https://answers.splunk.com/answers/188875/how-do-i-disable-transparent-huge-pages-thp-and-co.html
   		```
   		* The following services shuld be up Service ntp Not running on
   		
   		 ```
   		此项警告需要安装ntp.
   		1、apt-get install ntp -y
   		2、service ntp restart
   		参考：https://docs.hortonworks.com/HDPDocuments/Ambari-2.5.1.0/bk_ambari-installation/content/enable_ntp_on_the_cluster_and_on_the_browser_host.html
   		http://www.ehowstuff.com/how-to-install-and-configure-ntp-server-on-ubuntu-14-04/
   		```
   		![图5](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install3.png)
   		![图6](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install5.png)
   * 修复之后，点击Return Checks.
   * 选择需要安装的组件点击下一步，然后需要修改Hive依赖的database，都是准备工作中已经做完的。
   ![图7](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install6.png)
   * 有些步骤在此不再累赘描述，默认选择下一步就好。如果有自己的源安装过程会很顺利，如果是官网的源会很痛苦，经常断掉。
   ![图8](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/ambari/hdp-install11.png)
   
3. HUE
   * [参考1](http://gethue.com/how-to-build-hue-on-ubuntu-14-04-trusty/) [参考2](http://gethue.com/hue-3-12-the-improved-editor-for-sql-developers-and-analysts-is-out/)
   * 执行`apt-get install python2.7-dev make libkrb5-dev libxml2-dev libffi-dev libxslt-dev libsqlite3-dev libssl-dev libldap2-dev python-pip`安装hue依赖库。
   * [下载hue](http://gethue.com/downloads/hue-3.12.0.tgz) 
   * 解压，修改`desktop/conf/hue.ini`配置文件中的database选项
 		
 		```
 		[[database]]
    	# Database engine is typically one of:
    	# postgresql_psycopg2, mysql, sqlite3 or oracle.
    	#
    	# Note that for sqlite3, 'name', below is a path to the filename. For other backends, it is the database name
    	# Note for Oracle, options={"threaded":true} must be set in order to avoid crashes.
    	# Note for Oracle, you can use the Oracle Service Name by setting "host=" and "port=" and then "name=<host>:<port>/<service_name>".
    	# Note for MariaDB use the 'mysql' engine.
    	engine=mysql
    	host=localhost
    	port=3306
    	user=hue
    	password=hue
    	# Execute this script to produce the database password. This will be used when 'password' is not set.
    	## password_script=/path/script
    	## name=desktop/desktop.db
    	name = hue
    	## options={}
    	# Database schema, to be used only when public schema is revoked in postgres
    	## schema=
 		```
 	* 执行 `make apps`
 	* 执行 `build/env/bin/./supervisor -d`启动
