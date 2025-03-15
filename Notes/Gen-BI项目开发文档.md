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

##### 一、前端模板优化：

1. 删除多余文件

   - oneapi.json（示例API）

   - mock本地模拟数据文件夹

   - locales国际化文件夹

   - services.swagger文件夹

   - tests文件夹

   - types文件夹

   - jest.config.ts(单元测试框架)

2. 修改icon和开发登录页

   1. 前往[阿里巴巴矢量图标库](https://www.iconfont.cn/)选择合适的图表修改public/favicon.ico(网站小图标)和logo.svg(页面logo)
   2. 修改Login/index.tsx文件(删除自动登录，手机登录，其他登录方式，添加Github，微信等)
   3. 连通登录页面的前后端
   4. 调整好用户接口和界面优化

##### 二、智能分析业务开发

1. 业务大致流程

   1. 用户输入
      - 分析目标
      - 原始数据(excel)
      - 更精细化地控制图表:比如图表类型、图表名称等
   2. 后端校验
      1. 控制用户的输入是否合法
      2. 成本控制（次数统计、校验、鉴权等）
   3. 把处理后的数据输入给AI模型（调用AI接口），让AI模型提供图表信息和结论文本。
   4. `图表信息`（json代码）和`结论文本`在前端进行显示。

2. 设计后端开发接口：根据用户的输入(文本和文件)，最后返回信息(图表信息、结论文本)。

   1. 在ChartController中增添可接收文件的接口@PostMapping("/gen")

   2. 在dto/chart中增添GenChartByAiRequest请求

      ~~~java
      @Data
      public class GenChartByAiRequest implements Serializable {
      
          /**
           * 图表名称
           */
          private String name;
      
          /**
           * 分析目标
           */
          private String goal;
      
          /**
           * 图表类型
           */
          private String chartType;
      
          private static final long serialVersionUID = 1L;
      }
      ~~~

   3. 开发PostMapping("/gen")接口

      ~~~java
          /**
           * 智能分析
           *
           * @param multipartFile
           * @param genChartByAiRequest
           * @param request
           * @return
           */
          @PostMapping("/gen")
          public BaseResponse<String> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                                   GenChartByAiRequest genChartByAiRequest, HttpServletRequest request) {
              String name = genChartByAiRequest.getName();
              String goal = genChartByAiRequest.getGoal();
              String chartType = genChartByAiRequest.getChartType();
      
              // 校验
              // 如果分析目标为空，就抛出请求参数错误异常，并给出提示
              ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "目标为空");
              // 如果名称不为空，并且名称长度大于100，就抛出异常，并给出提示
              ThrowUtils.throwIf(StringUtils.isNotBlank(name) && name.length() > 100, ErrorCode.PARAMS_ERROR, "名称过长");
      
              // 读取到用户上传的 excel 文件，进行一个处理
              User loginUser = userService.getLoginUser(request);
              // 文件目录：根据业务、用户来划分
              String uuid = RandomStringUtils.randomAlphanumeric(8);
              String filename = uuid + "-" + multipartFile.getOriginalFilename();
              File file = null;
              try {
      
                  // 返回可访问地址
                  return ResultUtils.success("");
              } catch (Exception e) {
                  //    log.error("file upload error, filepath = " + filepath, e);
                  throw new BusinessException(ErrorCode.SYSTEM_ERROR, "上传失败");
              } finally {
                  if (file != null) {
                      // 删除临时文件
                      boolean delete = file.delete();
                      if (!delete) {
                          //  log.error("file delete error, filepath = {}", filepath);
                      }
                  }
              }
          }
      
      ~~~

##### 三、原始数据进行压缩

1. 为了避免调用API接口输入过长，对原始数据进行压缩处理

2. 在utils中创建ExcelUtils.java以实现excel-->csv的功能

   1. 实现上传功能，并将.xlsx格式转为CSV格式的字符串
   2. 对空表格进行过滤的处理

   ~~~java
   package com.yupi.springbootinit.utils;
   
   import cn.hutool.core.collection.CollUtil;
   import com.alibaba.excel.EasyExcel;
   import com.alibaba.excel.support.ExcelTypeEnum;
   import lombok.extern.slf4j.Slf4j;
   import org.apache.commons.lang3.ObjectUtils;
   import org.apache.commons.lang3.StringUtils;
   import org.springframework.web.multipart.MultipartFile;
   
   import java.io.IOException;
   import java.util.LinkedHashMap;
   import java.util.List;
   import java.util.Map;
   import java.util.stream.Collectors;
   
   
   /**
    * Excel 相关工具类
    */
   @Slf4j
   public class ExcelUtils {
       /**
        * excel 转 csv
        *
        * @param multipartFile
        * @return
        */
       public static String excelToCsv(MultipartFile multipartFile) {
   //        File file = null;
   //        try {
   //            file = ResourceUtils.getFile("classpath:网站数据.xlsx");
   //        } catch (FileNotFoundException e) {
   //            e.printStackTrace();
   //        }
           // 读取数据
           List<Map<Integer, String>> list = null;
           try {
               list = EasyExcel.read(multipartFile.getInputStream())
                       .excelType(ExcelTypeEnum.XLSX)
                       .sheet()
                       .headRowNumber(0)
                       .doReadSync();
           } catch (IOException e) {
               log.error("表格处理错误", e);
           }
           // 如果数据为空
           if (CollUtil.isEmpty(list)) {
               return "";
           }
           // 转换为 csv
           StringBuilder stringBuilder = new StringBuilder();
           // 读取表头(第一行)
           LinkedHashMap<Integer, String> headerMap = (LinkedHashMap) list.get(0);
           List<String> headerList = headerMap.values().stream().filter(ObjectUtils::isNotEmpty).collect(Collectors.toList());
           stringBuilder.append(StringUtils.join(headerList, ",")).append("\n");
           // 读取数据(读取完表头之后，从第一行开始读取)
           for (int i = 1; i < list.size(); i++) {
               LinkedHashMap<Integer, String> dataMap = (LinkedHashMap) list.get(i);
               List<String> dataList = dataMap.values().stream().filter(ObjectUtils::isNotEmpty).collect(Collectors.toList());
               stringBuilder.append(StringUtils.join(dataList, ",")).append("\n");
           }
           return stringBuilder.toString();
       }
       
   }
   
   ~~~


##### 四、准备调用AI

在gen接口中的用户输入部分融合加入系统的预设文本进去

~~~java
	StringBuilder userInput = new StringBuilder();
	userInput.append("你是一个数据分析师，接下来我会给你我的分析目标和原始数据，请告诉我分析结
论。").append("\n");
	userInput.append("分析目标：").append(goal).append("\n");

	// 把multipartFile传进来，其他的东西先注释
	String result = ExcelUtils.excelToCsv(multipartFile);
	userInput.append("数据：").append(result).append("\n");
	return ResultUtils.success(userInput.toString());
~~~

### 2025/3/14 第三期

#### 本期计划:

1. 后端
   1. 探索使用AI数据分析的应用方式
   2. 调用Deep Seek的接口
2. 前端
   1. 开发用户表单
   2. 对接后端AI分析接口
   3. 使用Echarts生成图表

#### 详细日志:

##### 后端部分

1. 使用Deep Seek赋能，进行AI数据分析
   1. AI生成结论
   2. AI生成图表代码

2. 调用腾讯云API详细过程
   - 前往腾讯云控制台生成自己的腾讯云密钥：SecretId和SecretKey，并充值10元以调用API
   - 前往腾讯云的[云API](https://console.cloud.tencent.com/api/explorer?Product=lkeap&Version=2024-05-22&Action=ChatCompletions)处，选择DeepSeek API接口，并下载示例SDK工程
   - 在后端Config中增添DeepSeekClientConfig类，作为一个客户端

   ~~~java
   package com.yupi.springbootinit.config;
   
   import com.tencentcloudapi.common.Credential;
   import com.tencentcloudapi.common.profile.ClientProfile;
   import com.tencentcloudapi.common.profile.HttpProfile;
   import com.tencentcloudapi.lkeap.v20240522.LkeapClient;
   import lombok.Data;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   
   @Configuration
   @ConfigurationProperties(prefix = "tencent.deepseek.client")
   @Data
   public class DeepSeekClientConfig {
   
       /**
        * 腾讯云账号 secretID
        */
       private String secretId;
       /**
        * 腾讯云密码 secretKey
        */
       private String secretKey;
   
       /**
        * 初始化客户端
        *
        * @return
        */
       @Bean
       public LkeapClient deepSeekClient() {
           // 实例化一个认证对象，入参需要传入腾讯云账户 SecretId 和 SecretKey，此处还需注意密钥对的保密
           // 代码泄露可能会导致 SecretId 和 SecretKey 泄露，并威胁账号下所有资源的安全性。以下代码示例仅供参考，建议采用更安全的方式来使用密钥，请参见：https://cloud.tencent.com/document/product/1278/85305
           // 密钥可前往官网控制台 https://console.cloud.tencent.com/cam/capi 进行获取
           Credential cred = new Credential(secretId, secretKey);
           // 实例化一个http选项，可选的，没有特殊需求可以跳过
           HttpProfile httpProfile = new HttpProfile();
           httpProfile.setEndpoint("lkeap.tencentcloudapi.com");
           // 实例化一个client选项，可选的，没有特殊需求可以跳过
           ClientProfile clientProfile = new ClientProfile();
           clientProfile.setHttpProfile(httpProfile);
           // 实例化要请求产品的client对象,clientProfile是可选的
           LkeapClient client = new LkeapClient(cred, "ap-guangzhou", clientProfile);
           return client;
       }
   }
   
   ~~~

   - 在配置中，加上腾讯云deep seek接口的账号和密码

      ~~~apl
      tencent:
        deepseek:
          client:
            secretId: 我的账号
            secretKey: 我的密码
      ~~~

   - 新建一个VO层对象，BiResponse作为deepseek的返回实体

   ~~~java
   package com.yupi.springbootinit.model.vo;
   
   import lombok.Data;
   
   /**
    * Bi 的返回结果
    */
   @Data
   public class BiResponse {
   
       private String genChart;
   
       private String genResult;
       // 新生成的图标id
       private Long chartId;
   }
   ~~~

   

   - 新增AiManager，作为一个请求实体对象

   ~~~java
   package com.yupi.springbootinit.manager;
   
   import com.tencentcloudapi.common.exception.TencentCloudSDKException;
   import com.tencentcloudapi.lkeap.v20240522.LkeapClient;
   import com.tencentcloudapi.lkeap.v20240522.models.ChatCompletionsRequest;
   import com.tencentcloudapi.lkeap.v20240522.models.ChatCompletionsResponse;
   import com.tencentcloudapi.lkeap.v20240522.models.Message;
   import com.yupi.springbootinit.common.ErrorCode;
   import com.yupi.springbootinit.exception.BusinessException;
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.stereotype.Service;
   
   import javax.annotation.Resource;
   
   /**
    * 用于对接 AI 平台
    */
   @Service
   @Slf4j
   public class AiManager {
   
       // 使用腾讯云对接 DeepSeek AI
       @Resource
       private LkeapClient deepSeekClient;
   
       /**
        * AI 对话
        *
        * @param message
        * @return
        */
       public String doChat(String message) {
           // 写系统预设
           final String SYSTEM_PROMPT = "你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：\n" +
                   "分析需求：\n" +
                   "{数据分析的需求或者目标}\n" +
                   "原始数据：\n" +
                   "{csv格式的原始数据，用,作为分隔符}\n" +
                   "请根据这两部分内容，按照以下指定格式生成内容（此外不要输出任何多余的开头、结尾、注释）\n" +
                   "【【【【【\n" +
                   "{前端 Echarts V5 的 option 配置对象js代码（输出 json 格式），合理地将数据进行可视化，不要生成任何多余的内容，比如注释}\n" +
                   "【【【【【\n" +
                   "{明确的数据分析结论、越详细越好，不要生成多余的注释}\n" +
                   "【【【【【";
           try {
               // 实例化一个请求对象,每个接口都会对应一个request对象
               ChatCompletionsRequest req = new ChatCompletionsRequest();
               req.setModel("deepseek-v3");
               req.setStream(false);
               // 系统消息
               Message[] messages = new Message[2];
               Message message0 = new Message();
               message0.setRole("system");
               message0.setContent(SYSTEM_PROMPT);
               messages[0] = message0;
               // 用户消息
               Message message1 = new Message();
               message1.setRole("user");
               message1.setContent(message);
               messages[1] = message1;
               req.setMessages(messages);
               // 返回的resp是一个ChatCompletionsResponse的实例，与请求对象对应
               ChatCompletionsResponse resp = deepSeekClient.ChatCompletions(req);
               String content = resp.getChoices()[0].getMessage().getContent();
               return content;
           } catch (TencentCloudSDKException e) {
               e.printStackTrace();
               log.error("AI 对话失败", e);
               throw new BusinessException(ErrorCode.OPERATION_ERROR, "AI 对话失败");
           }
       }
   }
   ~~~

   

   - 重写/gen接口

   ~~~java
       /**
        * 智能分析（同步）
        *
        * @param multipartFile
        * @param genChartByAiRequest
        * @param request
        * @return
        */
       @PostMapping("/gen")
       public BaseResponse<BiResponse> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                                    GenChartByAiRequest genChartByAiRequest, HttpServletRequest request) {
           String name = genChartByAiRequest.getName();
           String goal = genChartByAiRequest.getGoal();
           String chartType = genChartByAiRequest.getChartType();
           // 校验
           ThrowUtils.throwIf(org.apache.commons.lang3.StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "目标为空");
           ThrowUtils.throwIf(org.apache.commons.lang3.StringUtils.isNotBlank(name) && name.length() > 100, ErrorCode.PARAMS_ERROR, "名称过长");
           // 校验文件
           long size = multipartFile.getSize();
           String originalFilename = multipartFile.getOriginalFilename();
           // 校验文件大小
           final long ONE_MB = 1024 * 1024L;
           ThrowUtils.throwIf(size > ONE_MB, ErrorCode.PARAMS_ERROR, "文件超过 1M");
           // 校验文件后缀 aaa.png
           String suffix = FileUtil.getSuffix(originalFilename);
           final List<String> validFileSuffixList = Arrays.asList("xlsx");
           ThrowUtils.throwIf(!validFileSuffixList.contains(suffix), ErrorCode.PARAMS_ERROR, "文件后缀非法");
   
           User loginUser = userService.getLoginUser(request);
           // 限流判断，每个用户一个限流器
   //        redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getId());
           // 分析需求：
           // 分析网站用户的增长情况
           // 原始数据：
           // 日期,用户数
           // 1号,10
           // 2号,20
           // 3号,30
   
           // 构造用户输入
           StringBuilder userInput = new StringBuilder();
           userInput.append("分析需求：").append("\n");
   
           // 拼接分析目标
           String userGoal = goal;
           if (org.apache.commons.lang3.StringUtils.isNotBlank(chartType)) {
               userGoal += "，请使用" + chartType;
           }
           userInput.append(userGoal).append("\n");
           userInput.append("原始数据：").append("\n");
           // 压缩后的数据
           String csvData = ExcelUtils.excelToCsv(multipartFile);
           userInput.append(csvData).append("\n");
   
           String result = aiManager.doChat(userInput.toString());
           // 获取结果
           String[] splits = result.split("【【【【【");
           if (splits.length < 3) {
               throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 生成错误");
           }
           String genChart = splits[1].trim();
           String genResult = splits[2].trim();
           // 插入到数据库
           Chart chart = new Chart();
           chart.setName(name);
           chart.setGoal(goal);
           chart.setChartData(csvData);
           chart.setChartType(chartType);
           chart.setGenChart(genChart);
           chart.setGenResult(genResult);
           chart.setUserId(loginUser.getId());
           boolean saveResult = chartService.save(chart);
           ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");
           BiResponse biResponse = new BiResponse();
           biResponse.setGenChart(genChart);
           biResponse.setGenResult(genResult);
           biResponse.setChartId(chart.getId());
           return ResultUtils.success(biResponse);
       }
   ~~~

   