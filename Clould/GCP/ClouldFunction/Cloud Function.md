# 开发

## 代码结构

![image-20220228141551762](Cloud Function.assets/image-20220228141551762-16460289527733.png) 

## 文件

### index.js

入口文件

```javascript
const archiveData = require('./job.service')

exports.clouldfunction = async (event) => {
    console.log("event", event);
    let pubsubMessage = event.data;
    let body = Buffer.from(pubsubMessage, 'base64').toString();
    if (!!body) {
        body = JSON.parse(body);
        console.log("body", JSON.stringify(body));

        await archiveData(body);
    } else {
        console.log('No event data.');
    }
}
```





## PubSub

### 接收

> 触发CF。配置订阅的Topic，tipic被触发时将执行该CF。

```javascript
let pubsubMessage = event.data;
let body = Buffer.from(pubsubMessage, 'base64').toString();
if (!!body) {
    body = JSON.parse(body);
}
```

### 发送

> 该topic被触发（app_27354_npd-archive-completion）订阅该topic的所有CF都将执行。

```javascript
const publishMessage = async (eventObj, bool) => {
    const msgObj = {
        ContractInventoryKey: eventObj.ContractInventoryKey,
        ContractNbr: eventObj.ContractNbr,
        ModuleNm: 'CS',
        Result: bool ? 'Completed' : 'Failed'
    }
    console.log('publish Msg body', msgObj);
    try {
        const topicName = 'app_27354_npd-archive-completion';
        const pubsub = new PubSub();
        const dataBuffer = Buffer.from(JSON.stringify(msgObj));
        const messageId = await pubsub.topic(topicName).publish(dataBuffer);
        console.log(`${messageId} -  Message published.`);
    } catch (err) {
        console.error("sendMsg error: ", err)
}
```



# 测试

## 本地

> 本地没有PubSub环境，只能改写code使用HTTP模拟CF被触发。本地无法测试PubSub的发送结果。**调式**程序方法同API项目相同

#### 添加文件

增加三个文件并更改部分code，使CF能够接收HTTP 请求，然后用postman模拟CF被触发。

**.env** / **sa** 文件同API项目中相同

![image-20220228142015323](Cloud Function.assets/image-20220228142015323.png) 

**boot.js** 文件需更改。使code变为可接收HTTP请求的Express App

![image-20220228143048788](Cloud Function.assets/image-20220228143048788-16460298512774-16460298677195.png)

#### 改写入口Code

**index.js** event.body 即为 HTTP post请求中的body

```javascript
exports.clouldfunction = async (event) => {
    console.log("event", event);
    let body = event.body;
    if (!!body) {
        await archiveData(body);
    } else {
        console.log('No event data.');
    }
}
```



#### 安装本地测试所需的包

**nodemon dotenv express**

![image-20220228144048601](Cloud Function.assets/image-20220228144048601-16460304500636.png) 

#### 启动程序

**npm run watch**  通过boot.js为主文件启动程序

![image-20220228144149082](Cloud Function.assets/image-20220228144149082-16460305111067.png) 



#### Postman 触发CF

![image-20220228170622446](Cloud Function.assets/image-20220228170622446-16460391843038-16460391911349.png)

```json
http://localhost:8081
{
    "ContractInventoryKey": "100604be-eeb8-4ef0-bafd-28bb23f87d47",
    "ContractNbr":"9940212385"
}
```



## DEV/STAGE

### Pipeline触发CF

**参照 Pipeline** **[Execute Archive Contract Command](https://dev.azure.com/accenturecio07/NxtGenMMC_27354/_apps/hub/ms.vss-ciworkflow.build-ci-hub?_a=edit-build-definition&id=4561&view=Tab_Tasks)**

**Edit Task** 

1. 根据想要执行的环境在**Active Account and Envi**的script中找到对应的环境名、SA文件名

   <img src="Cloud Function.assets/image-20220228181454695-164604329675510.png" alt="image-20220228181454695" style="zoom:67%;" /> 

2. 创建对应环境的Task

   <img src="Cloud Function.assets/image-20220228182906899-164604414887211.png" alt="image-20220228182906899" style="zoom: 67%;" />

3. 修改variables 

   <img src="Cloud Function.assets/image-20220228184743694-164604526594512.png" alt="image-20220228184743694" style="zoom: 80%;" /> 

4. 修改测试所需数据和要触发的Tipic（一个Topic可能被多个CF订阅，运行Pipeline多个CF将被触发）

   <img src="Cloud Function.assets/image-20220228223529395.png" alt="image-20220228223529395" style="zoom: 67%;" /> 

5. 执行Pipeline

### 在GCP中查看Log

[GCP]([Functions – Cloud Functions – npd-27354-oriondev – Google Cloud Platform](https://console.cloud.google.com/functions/list?project=npd-27354-oriondev-95184682&supportedpurview=project)) 中找到对应CF，查看Log信息

![image-20220228230333952](Cloud Function.assets/image-20220228230333952-16460606152002.png) 



# 发布

> 通过Azure Pipeline 将Code 发布至GCP中并创建新的CF（自动）

## **添加文件** 

<img src="Cloud Function.assets/image-20220228231350087-16460612319543.png" alt="image-20220228231350087" style="zoom:80%;" /> 

## CI

### **azure-pipelines.yml** 

> 创建Pipeline时将配置为根据该yml文件设置pipeline

自动触发Pipeline的code branch

自动触发Pipeline的文件改动

```javascript
trigger:
  batch: true
  branches:
    include:
    - development
    - release_V2
    - master_V2
    - release
    - R22.2_Development
   paths:
    include:
    - Infra/002-215-cf-archivecontractingstatus/*
```

根据项目名称更改

```javascript
variables:
  TF_SRC_PATH: './Infra/002-215-cf-archivecontractingstatus'
  CFName: 'mmc-cf-archivecontractingstatus'
  NodeJSVer: '12.x'
  Archive_Src_Folder: 'dist_archive'
  WebPack_Output_Folder: 'Infra/002-215-cf-archivecontractingstatus/src'
```

### **创建Pipeline**

> 在[Azure](https://dev.azure.com/accenturecio07/NxtGenMMC_27354/_build) 中创建新pipeline

1. <img src="Cloud Function.assets/image-20220301110838601.png" alt="image-20220301110838601" style="zoom: 67%;" />

2. <img src="Cloud Function.assets/image-20220301111003569.png" alt="image-20220301111003569" style="zoom:50%;" /> 
3. <img src="Cloud Function.assets/image-20220301111206758.png" alt="image-20220301111206758" style="zoom:50%;" />  
4. <img src="Cloud Function.assets/image-20220301111239976.png" alt="image-20220301111239976" style="zoom:50%;" /> 
5. <img src="Cloud Function.assets/image-20220301111340525.png" alt="image-20220301111340525" style="zoom:80%;" /> 
6. <img src="Cloud Function.assets/image-20220301111426738.png" alt="image-20220301111426738" style="zoom: 80%;" /> 





## CD

### **main.tf** 

根据项目名称修改

<img src="Cloud Function.assets/image-20220301105926968.png" alt="image-20220301105926968" style="zoom: 67%;" />  



### 创建Release

> 在[Azure](https://dev.azure.com/accenturecio07/NxtGenMMC_27354/_release?view=all&_a=releases&definitionId=288) 创建Release ，参考[cf-archivecontractingstatus-cd](https://dev.azure.com/accenturecio07/NxtGenMMC_27354/_releaseDefinition?definitionId=288&_a=environments-editor-preview) 

1. clone 现有 Release

   <img src="Cloud Function.assets/image-20220301134722705.png" alt="image-20220301134722705" style="zoom: 50%;" /> 

2. 新增Artifacts，选择之前创建好的CI的路径

   <img src="Cloud Function.assets/image-20220301135818799.png" alt="image-20220301135818799" style="zoom: 80%;" /> 

3. 修改自动触发规则，**配置成功后如pipeline执行成功将自动执行release**

   <img src="Cloud Function.assets/image-20220301135944186.png" alt="image-20220301135944186" style="zoom: 80%;" /> 

4. 修改Deployed approver （参照[cf-archivecontractingstatus-cd](https://dev.azure.com/accenturecio07/NxtGenMMC_27354/_releaseDefinition?definitionId=288&_a=environments-editor-preview)  有些环境不需要设置approver）

   <img src="Cloud Function.assets/image-20220301140223948.png" alt="image-20220301140223948" style="zoom: 67%;" /> 

5. 修改变量 

   $(CF_JOBARCHIVE_TRIGGER_EVENT) 对应 Variable groups 中各个环境中的值，此变量为该CF所订阅的Topic需要根据CF具体功能需求所更改

   <img src="Cloud Function.assets/image-20220301134921770.png" alt="image-20220301134921770" style="zoom:80%;" />

3. 检查Variable groups 中各个环境中是否都已经定义了 Pipeline variables 中所使用的变量（$(CF_JOBARCHIVE_TRIGGER_EVENT)）

   <img src="Cloud Function.assets/image-20220301135242644.png" alt="image-20220301135242644" style="zoom:67%;" /> 
