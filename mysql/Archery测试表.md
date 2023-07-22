Archery 测试表



```sql
create database archery_data_test;
ALTER DATABASE archery_data_test DEFAULT CHARACTER SET utf8mb4;
```



```sql
CREATE TABLE res_user (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) NOT NULL COMMENT '用户id', 
  `username` varchar(45) NOT NULL COMMENT '真实姓名',
  `email` varchar(30) NOT NULL COMMENT '用户邮箱',
  `nickname` varchar(45) NOT NULL COMMENT '昵称',
  `avatar` int(11) NOT NULL COMMENT '头像',
  `birthday` date NOT NULL COMMENT '生日',
  `sex` tinyint(4) DEFAULT '0' COMMENT '性别',
  `short_introduce` varchar(150) DEFAULT NULL COMMENT '一句话介绍自己，最多50个汉字',
  `user_resume` varchar(300) NOT NULL COMMENT '用户提交的简历存放地址',
  `user_register_ip` int NOT NULL COMMENT '用户注册时的源ip',
  `create_time` timestamp NOT NULL COMMENT '用户记录创建的时间',
  `update_time` timestamp NOT NULL COMMENT '用户资料修改的时间',
  `user_review_status` tinyint NOT NULL COMMENT '用户资料审核状态，1为通过，2为审核中，3为未通过，4为还未提交审核',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_user_id` (`user_id`),
  KEY `idx_username`(`username`),
  KEY `idx_create_time`(`create_time`, `user_review_status`)
) ENGINE = InnoDB  COMMENT = '网站用户基本信息';
```

```sql
CREATE TABLE res_user2 (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) NOT NULL COMMENT '用户id', 
  `username` varchar(45) NOT NULL COMMENT '真实姓名',
  `phone` varchar(20) NOT NULL COMMENT '手机号',
  `create_time` timestamp NOT NULL default  COMMENT '用户记录创建的时间',
  `update_time` timestamp NOT NULL COMMENT on update current_time '用户资料修改的时间',
  `email` varchar(30) NOT NULL COMMENT '用户邮箱',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_user_id` (`user_id`),
  KEY `idx_username`(`username`)
) ENGINE = InnoDB  COMMENT = '网站用户基本信息';

```



```sql
ALTER TABLE res_user ADD phone VARCHAR(20);
```

```
insert res_user2(id, user_id, username, email, phone) values
    (null, '1', 'ZhangSan', 'zhangsan@163.com', '18233331111'),
    (null, '2', 'WangWu', 'wangwu@163.com', '18322221111');
```

