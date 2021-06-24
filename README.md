# springcloud-nacos-seata

**分布式事务组件seata的使用demo，AT模式，集成nacos、springboot、springcloud、mybatis-plus，数据库采用mysql**

demo中使用的相关版本号，具体请看代码。如果搭建个人demo不成功，验证是否是由版本导致，由于目前这几个项目更新比较频繁，版本稍有变化便会出现许多奇怪问题

* seata 0.8.0
* spring-cloud-alibaba-seata 2.1.0.RELEASE
* spring-cloud-starter-alibaba-nacos-discovery  2.2.1.RELEASE
* springboot 2.2.1.RELEASE
* springcloud  Hoxton.RELEASE

----------

## 1. 服务端配置

### 1.1 Nacos-server

版本为nacos-server-1.1.3，demo采用本地单机部署方式，请参考 [Nacos 快速开始](https://nacos.io/zh-cn/docs/quick-start.html)

### 1.2 Seata-server

seata-server为release版本0.8.0，demo采用本地单机部署，从此处下载 [https://github.com/seata/seata/releases](https://github.com/seata/seata/releases) 并解压

#### 1.2.1 修改conf/registry.conf 配置

设置type、设置serverAddr为你的nacos节点地址。

**注意这里有一个坑，serverAddr不能带‘http://’前缀， 如果不是默认端口，还要加上端口号！！**

~~~java
registry {
  type = "nacos"
  nacos {
    application = "seata-server"
    serverAddr = "10.0.2.32:30112"
    namespace = "a474cd9f-ea9d-45bc-9558-50440012fe83"
    cluster = "default"
    username = "dev"
    password = "123456"
  }

}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "10.0.2.32:30112"
    namespace = "a474cd9f-ea9d-45bc-9558-50440012fe83"
    cluster = "default"
    username = "dev"
    password = "123456"
  }
}

~~~

#### 1.2.2 修改conf/nacos-config.txt 配置

service.vgroup_mapping.${your-service-gruop}=default，中间的${your-service-gruop}为自己定义的服务组名称，服务中的application.properties文件里配置服务组名称。

demo中有两个服务，分别是storage-service和order-service，所以配置如下

~~~properties
service.vgroup_mapping.storage-service-group=default
service.vgroup_mapping.order-service-group=default
~~~

**注意这里, 微服务中使用seata1.0以后版本，使用`vgroupMapping` ，否则会报: no available service**

#### 1.3 启动seata-server

 windows环境：

1. **更改发布python脚本 nacos-config.py，添加nacos的命名空间，叫`tenant `不是namespace**, 以及端口

```python

	 url_prefix = sys.argv[1] + ':30112'
    url_postfix = '/nacos/v1/cs/configs?dataId={}&group=SEATA_GROUP&tenant=a474cd9f-ea9d-45bc-9558-50440012fe83&content={}'.format(str(pair[0]),str(line[line.index('=')+1:])).strip()


```

2. 启动脚本，发布`nacos-config.txt` 到nacos

   ```shell
   py nacos-config.py 10.0.2.32
   ```

3.  启动seata-server：双击seata-server.bat

---



## 2. 应用配置

### 2.1 数据库初始化

~~~SQL
-- seata-server要使用的表 貌似不是必须的！！the table to store GlobalSession data
drop table if exists `global_table`;
create table `global_table` (
  `xid` varchar(128)  not null,
  `transaction_id` bigint,
  `status` tinyint not null,
  `application_id` varchar(32),
  `transaction_service_group` varchar(32),
  `transaction_name` varchar(64),
  `timeout` int,
  `begin_time` bigint,
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`xid`),
  key `idx_gmt_modified_status` (`gmt_modified`, `status`),
  key `idx_transaction_id` (`transaction_id`)
);

-- the table to store BranchSession data
drop table if exists `branch_table`;
create table `branch_table` (
  `branch_id` bigint not null,
  `xid` varchar(128) not null,
  `transaction_id` bigint ,
  `resource_group_id` varchar(32),
  `resource_id` varchar(256) ,
  `lock_key` varchar(128) ,
  `branch_type` varchar(8) ,
  `status` tinyint,
  `client_id` varchar(64),
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`branch_id`),
  key `idx_xid` (`xid`)
);

-- the table to store lock data
drop table if exists `lock_table`;
create table `lock_table` (
  `row_key` varchar(128) not null,
  `xid` varchar(96),
  `transaction_id` long ,
  `branch_id` long,
  `resource_id` varchar(256) ,
  `table_name` varchar(32) ,
  `pk` varchar(32) ,
  `gmt_create` datetime ,
  `gmt_modified` datetime,
  primary key(`row_key`)
);


-- 创建 order库、业务表、undo_log表
create database seata_order;
use seata_order;

DROP TABLE IF EXISTS `order_tbl`;
CREATE TABLE `order_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL,
  `commodity_code` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT 0,
  `money` int(11) DEFAULT 0,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `undo_log`
(
  `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT,
  `branch_id`     BIGINT(20)   NOT NULL,
  `xid`           VARCHAR(100) NOT NULL,
  `context`       VARCHAR(128) NOT NULL,
  `rollback_info` LONGBLOB     NOT NULL,
  `log_status`    INT(11)      NOT NULL,
  `log_created`   DATETIME     NOT NULL,
  `log_modified`  DATETIME     NOT NULL,
  `ext`           VARCHAR(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;


-- 创建 storage库、业务表、undo_log表
create database seata_storage;
use seata_storage;

DROP TABLE IF EXISTS `storage_tbl`;
CREATE TABLE `storage_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `commodity_code` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY (`commodity_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `undo_log`
(
  `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT,
  `branch_id`     BIGINT(20)   NOT NULL,
  `xid`           VARCHAR(100) NOT NULL,
  `context`       VARCHAR(128) NOT NULL,
  `rollback_info` LONGBLOB     NOT NULL,
  `log_status`    INT(11)      NOT NULL,
  `log_created`   DATETIME     NOT NULL,
  `log_modified`  DATETIME     NOT NULL,
  `ext`           VARCHAR(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;

-- 初始化库存模拟数据
INSERT INTO seata_storage.storage_tbl (id, commodity_code, count) VALUES (1, 'product-1', 9999999);
INSERT INTO seata_storage.storage_tbl (id, commodity_code, count) VALUES (2, 'product-2', 0);
~~~

### 2.2 应用配置

见代码

几个重要的配置

1. 每个应用的resource里需要配置一个registry.conf ，demo中与seata-server里的配置相同
2. application.propeties 的各个配置项，注意spring.cloud.alibaba.seata.tx-service-group 是服务组名称，与nacos-config.txt 配置的service.vgroup_mapping.${your-service-gruop}具有对应关系

----------

## 3. 测试

1. 分布式事务成功，模拟正常下单、扣库存

   localhost:9091/order/placeOrder/commit   

2. 分布式事务失败，模拟下单成功、扣库存失败，最终同时回滚

   localhost:9091/order/placeOrder/rollback 





