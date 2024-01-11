---
title: MySQL 常用命令
date: 2020-09-04 15:39:05
tags: [mysql]
---

根据个人使用经验记录常用的 `MySQL` 命令作为备忘清单。

<!-- more -->

## MySQL

|命令|描述|
|--|--|
|` mysql –h localhost –u root -P 3306 –p `|连接数据库|

## 数据库

|命令|描述|
|--|--|
|`` SHOW DATABASES; ``|列出数据库|
|`` CREATE DATABASE IF NOT EXISTS `db_name` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci; ``|创建数据库|
|`` DROP DATABASE `db_name`; ``|删除数据库|
|`` SHOW CREATE DATABASE `db_name`; ``|查看数据库的创建信息|
|`` USE `db_name`; ``|选择数据库|


## 表

|命令|描述|
|--|--|
|`` SHOW TABLES; ``|列出表|
|`` CREATE TABLE `table_name` ( `` </br> `` `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键 id', `` </br> `` `uid` varchar(20) NOT NULL DEFAULT '' COMMENT 'uid', `` </br> `` `status` tinyint(1) NOT NULL DEFAULT '1' COMMENT '状态: 0 无效 1 有效', `` </br> `` `gmt_create` bigint(13) unsigned NOT NULL COMMENT '创建时间', `` </br> `` PRIMARY KEY (`id`), `` </br> `` UNIQUE KEY `unq_idx_uid` (`uid`) `` </br> `` ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='表的描述'; ``|创建表|
|`` DROP TABLE `table_name`; ``|删除表|
|`` SHOW CREATE TABLE `table_name`; ``|查看表的创建信息|
|`` DESC `table_name`; ``|查看表的字段|
|`` ALTER TABLE `table_name` RENAME `new_table_name`; ``|查看表的字段|
|`` ALTER TABLE `table_name` MODIFY `field_name` field_type; ``|修改字段类型|
|`` ALTER TABLE `table_name` ADD `field_name` field_type AFTER `another_field_name`; ``|添加字段|
|`` ALTER TABLE `table_name` DROP `field_name`; ``|删除字段|
|`` ALTER TABLE `table_name` CHANGE `field_name` `new_field_name` new_field_type; ``|修改字段名称|

## 增删改查

|命令|描述|
|--|--|
|`` INSERT INTO `table_name` (field_name1, field_name2) VALUES (value1, value2); ``|新增|
|`` SELECT * FROM `table_name` WHERE `field_name` = value; ``|查询|
|`` UPDATE `table_name` SET `field_name1` = value1 WHERE `field_name2` = value2; ``|更新|
|`` DELETE FROM `table_name` WHERE `field_name` = value; ``|删除|
|`` TRUNCATE `table_name`; ``|清空|

## 事务

|命令|描述|
|--|--|
|` BEGIN; ` </br> ` START TRANSACTION; `|开启事务|
|` START TRANSACTION WITH CONSISTENT SNAPSHOT; `|开启事务并立即创建 read-view|
|` ROLLBACK; `|回滚事务|
|` COMMIT; `|提交事务|
|` SELECT @@autocommit; `|查看 autocommit，默认 1|
|` SET @@autocommit = 0; `|设置为手动提交|
|` SHOW VARIABLES LIKE 'transaction_isolation';` </br> ` SHOW VARIABLES LIKE 'tx_isolation'; `|查看隔离级别|
|` SET transaction_isolation = 'READ-UNCOMMITTED'\|'READ-COMMITTED'\|'REPEATABLE-READ'\|'SERIALIZABLE'; ` </br> ` SET SESSION transaction isolation level READ UNCOMMITTED\|READ COMMITTED\|REPEATABLE-READ\|SERIALIZABLE; `|设置隔离级别|

## 用户

|命令|描述|
|--|--|
|` CREATE USER 'username'@'ip' IDENTIFIED BY 'password'; `|创建用户|
|` GRANT ALL ON db_name.* TO 'username'@'ip'; `|授权|
|` FLUSH PRIVILEGES; `|刷新权限|
|` REVOKE ALL ON db_name.* FROM 'username'@'ip'; `|取消授权|
|` DROP USER 'username'@'ip'; `|删除用户|

## 变量

全局变量、会话变量、用户变量、局部变量。

|命令|描述|
|--|--|
|` SHOW [GLOBAL\|SESSION] VARIABLES; `|查看全部变量|
|` SHOW [GLOBAL\|SESSION] VARIABLES [LIKE 匹配的模式]; ` </br> ` SELECT @@[GLOBAL\|SESSION].variable_name `|查看单个变量|
|` SET [GLOBAL\|SESSION] variable_name = value; ` </br> ` SET @@[GLOBAL\|SESSION].variable_name = value `|设置单个变量|

> `SESSION` 可以省略。
