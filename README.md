# 分布式事务

## 一、准备数据
执行资料中提供的`seata_demo.sql`文件，导入数据。

其中包含4张表。

Order表：

```mysql
CREATE TABLE `order_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL COMMENT '用户id',
  `commodity_code` varchar(255) DEFAULT NULL COMMENT '商品码',
  `count` int(11) unsigned DEFAULT '0' COMMENT '购买数量',
  `money` int(11) unsigned DEFAULT '0' COMMENT '总金额',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

商品库存表：

```mysql
CREATE TABLE `storage_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `commodity_code` varchar(255) DEFAULT NULL COMMENT '商品码',
  `count` int(11) unsigned DEFAULT '0' COMMENT '商品库存',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `commodity_code` (`commodity_code`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

用户账户表：

```mysql
CREATE TABLE `account_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL COMMENT '用户id',
  `money` int(11) unsigned DEFAULT '0' COMMENT '用户余额',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

还有用来记录Seata中的事务日志表undo_log，其中会包含`after_image`和`before_image`数据，用于数据回滚：

```mysql
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

## 二、服务介绍

- eureka-server：注册中心，端口8761
- order-service：订单服务，提供根据数据创建订单的功能，端口8082（发起订单，调用库存和用户服务）
  - account-service：用户服务，提供操作用户账号余额的功能，端口8083
  - storage-service：仓储服务，提供扣减商品库存功能，端口8081

## 三、测试事务

接下来，我们来测试下分布式事务的现象。

下单的接口是：

- 请求方式：POST
- 请求路径：/order
- 请求参数：form表单，包括：
    - userId：用户id
    - commodityCode：商品码
    - count：购买数量
    - money：话费金额
- 返回值类型：long，订单的id