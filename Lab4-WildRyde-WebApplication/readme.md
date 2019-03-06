# Serverless Web 应用（动静分离）

## 前言
在本次实验中，您将部署一个简单的Web应用程序，使用户可以从Wild Rydes车队请求独角兽乘坐。该应用程序将向用户提供基于HTML的用户界面，用于指示他们想要获取的位置，并在后端与RESTful Web服务接口对接以提交请求并派遣附近的独角兽。

应用程序体系结构使用AWS Lambda，Amazon API Gateway，Amazon S3，Amazon DynamoDB。 S3托管静态Web资源，包括在用户浏览器中加载的HTML，CSS，JavaScript和图像文件。在浏览器中执行的JavaScript从使用Lambda和API Gateway构建的公共后端API发送和接收数据。最后，DynamoDB提供了一个持久层，可以通过API的Lambda函数存储数据。

请参阅下图，以了解完整的体系结构。目前本实验不涉及 Cognito 这部分。  

![架构](./img/Picture1.png)

该研讨会分为多个模块，您必须在继续下一个模块之前完成每个模块。
* 静态网站托管
* 无服务器后台
* RESTful API 集成

## Amazon S3静态网站托管

在此模块中，您将配置Amazon Simple Storage Service（S3）以托管Web应用程序的静态资源。在后续模块中，您将使用JavaScript为这些页面添加动态功能，以调用使用AWS Lambda和Amazon API Gateway构建的远程RESTful API。

### 架构概览
该模块的架构非常简单。所有静态Web内容（包括HTML，CSS，JavaScript，图像和其他文件）都将存储在Amazon S3中。然后，您的最终用户将使用Amazon S3公开的公共网站URL访问您的网站。您无需运行任何Web服务器或使用其他服务即可使您的站点可用。

![架构](./img/Picture2.png)

出于本模块的目的，您将使用我们提供的Amazon S3网站端点URL。它的格式为http：// {your-bucket-name} .s3-website- {region} .amazonaws.com.cn。对于大多数实际应用程序，您需要使用自定义域来托管您的站点。如果您对使用自己的域感兴趣，请按照Amazon S3文档中使用自定义域设置静态网站的说明进行操作。

### 创建S3存储桶

请选择对应的区域，以下示例宁夏区域进行实验，请保证您所部署的所有AWS资源都会在宁夏区域部署。在开始之前，请确保从AWS控制台右上角的下拉列表中选择您所在的区域。

![选择区域](./img/Picture3.png)
  
Amazon S3可用于托管静态网站，而无需配置或管理任何Web服务器。在此步骤中，您将创建一个新的S3存储桶，用于托管Web应用程序的所有静态资产（例如，HTML，CSS，JavaScript和图像文件）。  
  
使用控制台或AWS CLI创建Amazon S3存储桶。请记住，您的存储桶名称必须在所有地区和客户中具有全球唯一性。我们建议使用像`aws-serverless-wildrydes-<your name>`这样的名称。如果您收到存储桶名称已存在的错误，请尝试添加其他数字或字符，直到找到未使用的名称。

Step-By-Step 操作指引  
1. 在AWS管理控制台中，选择服务，然后在存储和内容分发下选择S3。
2. 选择创建存储桶
3. 为您的存储桶提供全局唯一名称，例如`aws-serverless-wildrydes-<your name>`
4. 从下拉列表中选择您选择用于此实验的区域 – 中国（宁夏）
5. 选择对话框左下角的“创建”，而不选择要从现有复制设置的存储桶

![s3](./img/Picture4.png)

### 上传网页文件  
将此模块的网站资产上传到S3存储桶。您可以使用AWS管理控制台（需要Google Chrome浏览器）或者AWS CLI来完成此步骤。如果您已在本地计算机上安装并配置了AWS CLI，我们建议您使用该方法。否则，如果您安装了最新版本的Google Chrome，请使用控制台。  
  
通过控制台上传实验文件  
1. 通过如下链接下载本次实验的文件内容  
[WildRydes-aws-serverless-workshop.zip](./WildRydes-aws-serverless-workshop.zip)  
1. 解压缩后把tutorial文件夹（不包含tutorial文件夹）内的所有内容通过网页控制台上传到刚才创建的S3存储桶内,可以直接全选文件夹内的所有内容，通过鼠标拖拽到上传界面(请不要通过“上传文件”按钮的方式上传文件)。请保证内容里的所有文件（包括文件夹）都在上传范围内。  
![upload](./img/Picture5.png)

3. 点击上传等到文件全部导入S3存储桶内

### 添加存储桶策略允许公开访问  
您可以使用存储桶策略定义谁可以访问S3存储桶中的内容。存储桶策略是JSON文档，用于指定允许哪些主体对存储桶中的对象执行何种操作。   
  
您需要向新的Amazon S3存储桶添加存储桶策略，以允许匿名用户查看您的网站。默认情况下，只有有权访问您的AWS账户的经过身份验证的用户才能访问您的存储桶。  
  
请参阅如下示例，该策略将授予对匿名用户的只读访问权限。此示例策略允许Internet上的任何人查看您的内容。  
1. 选择存储桶，选择权限选项卡，然后选择存储桶策略。
2. 在存储桶策略编辑器中粘贴如下代码示例，确保在Resource中替换您存储桶的名字  
3. 点击保存

![policy](./img/Picture6.png)

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws-cn:s3:::您的存储桶名称/*"
            }
        ]
    }

### 开启静态网站托管
默认情况下，S3存储桶中的对象可通过结构为  
`http://aws-serverless-wildrydes-<name>.s3-website.cn-northwest-1.amazonaws.com.cn`
的URL提供。要从根URL（例如/index.html）提供资源，您需要在存储桶上启用网站托管。有关使用此功能的更多详细信息，请参阅AWS网站文档。这将使您的对象在存储桶的AWS区域特定网站端点可用。请参阅Amazon Simple Storage Service网站端点以查找适用于您所在地区的域。
您还可以为您的网站使用自定义域名。例如，`http://www.wildrydes.com` 托管在S3上。本研讨会不涉及设置自定义域名，但您可以在我们的文档中找到详细说明。
开启静态网站托管操作指引
1. 选择存储桶，选择属性选项卡，找到静态网站托管的选项
2. 点击静态网站托管，选择“使用此存储桶托管网站”
3. 在索引文档里填入“index.html”作为默认主页
4. 点击保存

![s3](./img/Picture7.png)

检查网站托管内容  
完成这些步骤后，您应该可以通过访问S3存储桶的网站端点URL来访问您的静态网站。  
选择静态网站托管页里终端节点，点击打开网页  
   
打开后可以看到我们网站托管的内容，恭喜您，已经成功托管了一个静态网站在AWS上！！！  

![s3](./img/Picture8.png)
    
    
   
下一节：[搭建无服务器后台](./readme2.md)
