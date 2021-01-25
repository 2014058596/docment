- #### 什么是canal

  - canal，译意为水道/管道/沟渠，主要用途是基于 **MySQL 数据库增量日志解析**，提供**增量数据订阅和消费**。

  ![canal-1](..\image\canal-1.png)

- **Canal能做什么**

  - 数据库镜像

  - 数据库实时备份

  - 索引构建和实时维护

  - 业务cache(缓存)刷新

  - 带业务逻辑的增量数据处理

  - 如何搭建canal

    - mysql服务端配置

      - 准备一台mysql（本文采用docker版本的mysql）

      - 修改mysql配置，启动bin-log日志

        ```shell
        #进入mysql容器
        docker exec -it mysql /bin/bash
        vi /etc/mysql/mysql.conf.dmysqld.cnf
        #增加如下配置
        # 打开binlog
        log-bin=mysql-bin
        # 选择ROW(行)模式
        binlog-format=ROW
        # 配置MySQL replaction需要定义，不要和canal的slaveId重复
        server_id=1
        ```

      - 检查mysql 日志是否开启

        ```shell
        #查看日志开启
        show variables  like 'log_bin';
        #查看binlog日志文件列表
        show binary logs;
        #查看正在写入的binlog文件
        show master  status ;
        ```

      - ```mysql
        创建mysql用户
        
        CREATE USER canal IDENTIFIED BY 'canal';  
        GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
        FLUSH PRIVILEGES;
        ```

  - 搭建canal-admin

    - ```mysql
      --sql初始化化脚本
      -- ----------------------------
      -- Table structure for canal_adapter_config
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_adapter_config`;
      CREATE TABLE `canal_adapter_config` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `category` varchar(45) NOT NULL,
        `name` varchar(45) NOT NULL,
        `status` varchar(45) DEFAULT NULL,
        `content` text NOT NULL,
        `modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      -- ----------------------------
      -- Table structure for canal_cluster
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_cluster`;
      CREATE TABLE `canal_cluster` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `name` varchar(63) NOT NULL,
        `zk_hosts` varchar(255) NOT NULL,
        `modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      -- ----------------------------
      -- Table structure for canal_config
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_config`;
      CREATE TABLE `canal_config` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `cluster_id` bigint(20) DEFAULT NULL,
        `server_id` bigint(20) DEFAULT NULL,
        `name` varchar(45) NOT NULL,
        `status` varchar(45) DEFAULT NULL,
        `content` text NOT NULL,
        `content_md5` varchar(128) NOT NULL,
        `modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        UNIQUE KEY `sid_UNIQUE` (`server_id`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      -- ----------------------------
      -- Table structure for canal_instance_config
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_instance_config`;
      CREATE TABLE `canal_instance_config` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `cluster_id` bigint(20) DEFAULT NULL,
        `server_id` bigint(20) DEFAULT NULL,
        `name` varchar(45) NOT NULL,
        `status` varchar(45) DEFAULT NULL,
        `content` text NOT NULL,
        `content_md5` varchar(128) DEFAULT NULL,
        `modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        UNIQUE KEY `name_UNIQUE` (`name`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      -- ----------------------------
      -- Table structure for canal_node_server
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_node_server`;
      CREATE TABLE `canal_node_server` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `cluster_id` bigint(20) DEFAULT NULL,
        `name` varchar(63) NOT NULL,
        `ip` varchar(63) NOT NULL,
        `admin_port` int(11) DEFAULT NULL,
        `tcp_port` int(11) DEFAULT NULL,
        `metric_port` int(11) DEFAULT NULL,
        `status` varchar(45) DEFAULT NULL,
        `modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      -- ----------------------------
      -- Table structure for canal_user
      -- ----------------------------
      DROP TABLE IF EXISTS `canal_user`;
      CREATE TABLE `canal_user` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `username` varchar(31) NOT NULL,
        `password` varchar(128) NOT NULL,
        `name` varchar(31) NOT NULL,
        `roles` varchar(31) NOT NULL,
        `introduction` varchar(255) DEFAULT NULL,
        `avatar` varchar(255) DEFAULT NULL,
        `creation_date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
      
      SET FOREIGN_KEY_CHECKS = 1;
      
      -- ----------------------------
      -- Records of canal_user
      -- ----------------------------
      BEGIN;
      INSERT INTO `canal_user` VALUES (1, 'admin', '6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9', 'Canal Manager', 'admin', NULL, NULL, '2019-07-14 00:05:28');
      COMMIT;
      ```

    - ```
      docker pull canal/canal-admin:v1.1.4
      
      //或者 直接docker 命令 启动
      sudo docker run -it --name canal-admin \
          -e server.port=8089 \
          -e canal.adminUser=admin \
          -e canal.adminPasswd=admin \
          -e spring.datasource.address=192.168.1.249:3306 \
          -e spring.datasource.database=bxkc_crm \
          -e spring.datasource.username=canal \
          -e spring.datasource.password=canal \
          --net=host \
          -m 1024m \
          -d canal/canal-admin:v1.1.4
          
      ```

  - 启动canal

    - ```
      docker run -it --name=canal-server \
          -e canal.admin.manager=192.168.1.15:8089 \
          -e canal.admin.port=11110 \
          -e canal.port=11121 \
          -e canal.metrics.pull.port=11122 \
          -e canal.admin.user=admin \
          -e canal.admin.passwd=4ACFE3202A5FF5CF467898FC58AAB1D615029441 \
          -h 192.168.1.15 \
          --net=host \
          -d canal/canal-server:v1.1.4
      ```

  - 启动成功后，访问192.168.1.15:8098 输入账号 admin 密码 123456 查看server是否注册成功

  - ![canal-2](..\image\canal-2.png)

  - 然后新增一个instance 修改配置如下

    - ```properties
      #################################################
      ## mysql serverId , v1.0.26+ will autoGen
      canal.instance.mysql.slaveId=86
      
      # enable gtid use true/false
      canal.instance.gtidon=false
      
      # position info
      canal.instance.master.address=192.168.1.249:3306
      canal.instance.master.journal.name=
      canal.instance.master.position=
      canal.instance.master.timestamp=
      canal.instance.master.gtid=
      
      # rds oss binlog
      canal.instance.rds.accesskey=
      canal.instance.rds.secretkey=
      canal.instance.rds.instanceId=
      
      # table meta tsdb info
      canal.instance.tsdb.enable=true
      #canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
      #canal.instance.tsdb.dbUsername=canal
      #canal.instance.tsdb.dbPassword=canal
      
      #canal.instance.standby.address =
      #canal.instance.standby.journal.name =
      #canal.instance.standby.position =
      #canal.instance.standby.timestamp =
      #canal.instance.standby.gtid=
      
      # username/password
      canal.instance.dbUsername=canal
      canal.instance.dbPassword=canal
      canal.instance.connectionCharset = UTF-8
      # enable druid Decrypt database password
      canal.instance.enableDruid=false
      #canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==
      
      # table regex
      canal.instance.filter.regex=.*\\..*
      # table black regex
      canal.instance.filter.black.regex=
      # table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
      #canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
      # table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
      #canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch
      
      # mq config
      canal.mq.topic=example
      # dynamic topic route by schema or table regex
      #canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
      canal.mq.partition=0
      # hash partition config
      #canal.mq.partitionsNum=3
      #canal.mq.partitionHash=test.table:id^name,.*\\..*
      #################################################
      
      ```

      

