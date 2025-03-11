# Gen-BI项目开发文档

## 一、需求分析

1. 智能分析：用户输入的分析目标和原始数据，便可自动生成图表和分析结论
2. 图表管理
3. 图表生成的异步化（消息队列：因为一个图片生成可能比较长，因此会有多人同时图表生成，因此使用消息队列的异步化处理）
4. 对接AI能力

## 二、项目开发日志

### 2025/3/7 第一期

#### 本期计划：

1. 确立项目的立项、技术选型、架构图、业务流程
2. 使用模板对前端项目和后端项目初始化
3. 前端开发
   1. 快速开发登录功能
   2. 图表分析页面的开发
   3. 图表管理页面的开发
4. 后端开发
   1. 库表设计
   2. 图表管理开发
   3. 文件上传接口开发

#### 详细日志：

##### 一、前端项目搭建

1. 使用NVM安装管理不同版本的node.js（本次为18.20.4）

   ~~~
   nvm list 			//查看本地安装的所有版本
   nvm install 14.15.4 //安装特定版本node.js
   nvm use 14.15.4		//使用特定版本
   nvm install 14.15.4 //卸载特定版本
   nvm ls available	//列出远程服务器node.js所有可用版本 
   ~~~

2. 前往Ant Design Pro官网，根据使用教程初始化并搭建前端项目

   ~~~bash
   # 使用pro-cli来快速初始化脚手架
   npm i @ant-design/pro-cli -g
   pro create Gen-BI-frontend
   
   # 选择umi4版本
   ? 🐂 使用 umi@4 还是 umi@3 ? (Use arrow keys)
   ❯ umi@4
     umi@3
    
   # 选择simple脚手架便于二次开发
   ? 🚀 要全量的还是一个简单的脚手架? (Use arrow keys)
   ❯ simple
     complete
   # 安装项目各种依赖
   cd Gen-BI-frontend
   tyarn
   ~~~

3. 为前端给项目减负，去掉不必要文件

   1. 使用git代码托管
   2. 移除国际化功能（运行i18n-remove命令）
   3. 并在src/components/index.ts文件中export处移除SelectLang

##### 二、后端项目搭建

1. 借用鱼皮的万能后端开发模板进行初始化

2. 数据库库表设计

   ~~~mysql
   -- 用户表
   create table if not exists user
   (
       id           bigint 	  auto_increment 		  	comment 'id' primary key,
       userAccount  varchar(256)                           not null comment '账号',
       userPassword varchar(512)                           not null comment '密码',
       userName     varchar(256)                           null comment '用户昵称',
       userAvatar   varchar(1024)                          null comment '用户头像',
       userRole     varchar(256) default 'user'            not null comment '用户角色：user/admin',
       createTime   datetime     default CURRENT_TIMESTAMP not null comment '创建时间',
       updateTime   datetime     default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
       isDelete     tinyint      default 0                 not null comment '是否删除',
       index 	idx_userAccount   (userAccount)
   ) comment '用户' collate = utf8mb4_unicode_ci;
   ~~~

   ~~~mysql
   -- 图表表
   create table if not exists chart
   (
       id           bigint auto_increment comment 'id' primary key,
       goal				 text  null comment '分析目标',
       chartData    text  null comment '图表数据',
       chartType	   varchar(128) null comment '图表类型',
   	  genChart		 text	 null comment '生成的图表数据',
       genResult		 text	 null comment '生成的分析结论',
       userId       bigint null comment '创建用户 id',
       createTime   datetime     default CURRENT_TIMESTAMP not null comment '创建时间',
       updateTime   datetime     default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
       isDelete     tinyint      default 0                 not null comment '是否删除'
   ) comment '图表信息表' collate = utf8mb4_unicode_ci;
   ~~~

3. 后端基础开发

   - 执行MySQL语句来建表   

   - 使用MyBatis X插件生成代码

     - 选上MyBatis Plus-3 、Lombok、Comment、Actual Column

   - 迁移生成的代码 

   - 复制老的增删改查代码，根据表来重构

   - 根据接口文档来测试

     ~~~apl
     http://localhost:8101/api/doc.html 
     ~~~

     

##### 三、前后端串连

- 使用 ant design pro 自带的 openapi 工具，根据后端的 swagger 接口文档数据自动生成对应的请求 service 代码。

- 注意：前端须更改对应的请求地址为你的后端地址，方法：在 app.tsx 里修改 request.baseURL

  ~~~tsx
  baseURL:"http://localhost:8101"
  ~~~


### 2025/3/11 第二期

#### 本期计划：

1. 前端模板优化，删除多余部分。
2. 开发前后端的登录、注册页面。
3. 学习使用AI生成BI图表的完整流程。
4. 开发AI智能分析功能。
5. 基本跑动文件处理的业务。
   1. 开发前后端的文件上传功能
   2. 开发Excel处理功能。
6. 开发图表管理功能。

#### 详细日志：

##### 一、删除部分

- oneapi.json（示例API）
- mock本地模拟数据文件夹
- locales国际化文件夹
- services.swagger文件夹
- tests文件夹
- types文件夹
- jest.config.ts(单元测试框架)