# 如何使用 Jenkins 插件实现发送自定义邮件



前言：我们的工程使用 Jenkins + Fastlane 作为我们的 CI/CD 工具，一直以来只是使用到了打包+上传内测托管平台的基本功能。最近实现了打包后发送自定义邮件的功能，在此记录一下。

## 需求

自定义邮件内容需要包含以下内容：

- 项目的名称
- 版本号
- 修改内容
- 内测包URL
- 其他基本构建信息

## 准备

Jenkins, Email plugins

#### 1. 安装[Jenkins](https://www.jenkins.io/download/)

网上有很多如何安装Jenkins的文章，限于篇幅，这里就不赘述了。

#### 2. 下载插件

在 Jenkins plugin manager 页面搜索需要的plugins。这里我们需要下载两个和email 相关的插件：`Email Extension Plugin` 和 `Email Extension Plugin`.

![install_plugins](https://github.com/T2Je/T2Je.github.io/blob/main/CI/Images/email_plugins.png?raw=true)



##开始搭建

#### 1. Configure Email 

前往 Manage Jenkins -> Configure System. 滑动找到 email notification 区域， 如果你使用企业邮箱作为你的SMTP server，选择`Use SMTP authentication`输入 `SMTP server` ，点击 `Advanced` ，输入用户名、密码，选择 `Use SSL`，`SMTP port` （465），保存。

![email_extended](https://github.com/T2Je/T2Je.github.io/blob/main/CI/Images/configure_email.png?raw=true)

#### 2. Test Email Configuration

##### 1. 使用 `Configure System` 页面的 test email 功能

![test_email_configure_system](https://github.com/T2Je/T2Je.github.io/blob/main/CI/Images/test_email_configuration.png?raw=true)

##### 2. 使用Pipeline 测试邮件发送

在 Jenkins 首页创建一个新 job，选择 `Pipeline`，在Script  中拷贝下面的代码

```shell
pipeline {
    agent any
    
    stages {
        stage('Ok') {
            steps {
                echo "Ok"
            }
        }
    }
    post {
        always {
            emailext body: 'A Test EMail', recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']], subject: 'Test'
        }
    }
}
```

保存，然后点击 `Build Now`，如果 配置 Email Extended 正确，`Configure System` 里配置的 `Default Recipient` 就会收到测试邮件。

#### 3. 配置Jenkins 里的工程

进入工程中，选择 `Configure` ，滑动到底部，选择 `Add post-build action`，选择 `Editable Email Notification Templates`

<img src="https://github.com/T2Je/T2Je.github.io/blob/main/CI/Images/post_build_action.png?raw=true" alt="add_post_build_action" style="zoom: 33%;" />



查看字段的配置，点击 `Advanced`， 修改`Trigger`, 增加一个 `always` trigger，然后保存。

选择 `Build Now`，build 完成后，检查一下邮箱，应该会收到一封邮件。

#### 4. 修改邮件内容

Jenkins 发送的邮件默认主题和内容都可以在 首页的 `Configure System`里面修改，在可以在每个项目中自定义，这里我选择在项目中修改邮件内容。

在项目的配置页面，滑到底部，找到 `Post-build Actions` -> `Editable Email Notification`

![post_build_action_email](/Users/xiaoyang/Desktop/Blog/Jenkins-Email/post_build_action_email.png)

想要修改邮件内容，可以修改 `Default Content` 内容，可以在这里写纯文本的内容，也可以写 html 内容，不过需要修改 `Content Type` 为 html 或者 `Both html and plain text`。html 的内容，网上有很多模版，感兴趣的可以去搜一下。但是由于构建完成后，只能获取到有限的构建参数，比如  `JOB_NAME`，`BUILD_NUMBER`，`BUILD_URL`等信息，无法获取到需求中提到的，项目版本号，安装包的安装地址等信息。

##### 解决方案：

项目版本号，安装包的安装地址这些信息都是在构建项目的时候产生的，可以将这些信息存下来，用脚本获取这些信息。

###### 方案一

在`Editable Email Notification`  `Pre-send Script` 字段里，可以写脚本，这里主要是用来在邮件发送之前修改邮件内容的。看起来这就是解决方案了。脚本语言是`Groovy`，一种`Java`平台的脚本语言。

在项目构建的脚本中(`Execute shell`)，我添加了一段生成邮件内容的 html 文件的代码，html 文件中加入了项目的版本号，安装包地址等信息，你也可以添加其他你想要的信息。

```shell
touch testReport.html
cat > testReport.html  << EOF
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>第$jenkins_build_number次构建日志</title>
  </head>
  <body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4" offset="0">
    <table width="95%" cellpadding="0" cellspacing="0" style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
      <tr>
        (本邮件由系统自动发出，无需回复！)<br/>
        各位好，以下是$project_name项目构建信息<br/>
      </tr>
      <tr>
          <td><br/>
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" />
          </td>
      </tr>
      <tr>
        <td>
          <ul>
            <li>项目名称：$project_name</li>
            <li>构建版本：$version_number/$build_number</li>
            <li>安装包地址：<a href=$url>$url</li>
          </ul>                   
        </td>
      </tr>
    </table>
  </body>
</html>

EOF
```

在 `Pre-send Script` 中，添加一段 groovy 代码：

```shell
String result = build.result.toString()
if (result.equals("SUCCESS")) {
      def reportPath = build.getWorkspace().child("testReport.html")
      String reportStr = reportPath.readToString()
      msg.setContent(reportStr, "text/html");
}
```

这样我们就可以通过修改 `Execute shell` 来自定义邮件内容了。如果你想在构建失败时发送不同的邮件，只需要在 `Execute shell`  中写一份构建失败的 html 文件就可以了，当然也可以是纯文本的。

###### 方案二

为 Jenkins `Email extension plugin` 配置groovy 模版。

- 设置 `Editable Email Notification` `Content Type` 为 `HTML`或者 `Both HTML and Plain Text` 

- 在 `Content` 中获取 groovy template，如下：

  ```shel
  ${SCRIPT, template="test_groovy_html.template"}
  ```

- 将 test_groovy_html.template 放入 Jenkins 安装根目录的 email-templates 目录下，如：xx/.jenkins/email-templates

邮件groovy template可以在网上找一些，官方也有一些[模版](https://github.com/jenkinsci/email-ext-plugin/tree/master/docs/templates)，可以参考使用。

在模版中，插入一段 groovy 脚本用来读取一个 Json 文件，代码如下：

```groovy
import groovy.json.JsonSlurper
def jsonSlurper = new JsonSlurper()

File propertiesFile = new File("${build.properties.moduleRoot}/testReport.json")
def data = jsonSlurper.parse(propertiesFile)

// get value from key: data["version"]
```

这个Json 文件是构建项目的脚本(`Execute shell`)在构建项目时生成的，里面包含了版本号，安装包地址等信息，代码如下：

```shell
# Writing test report template file required properties into a JSON file
json=$(cat <<-END
    {
        "version": "Your version",
        "url": "Your package url",				
				# ... any information you want
    }
END
)

echo $json > testReport.json
```

👀遇到中文乱码的问题，记得在 HTML Head 中改编码。

```html
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
```

## 结语

这就是我如何实现Jenkins 发送自定义邮件的大概过程，希望能够对你有所帮助与启发。另外，我还想抽空了解下Jenkins Pipeline，不知道使用 Pipeline 实现发送自定义邮件是否更方便，在网上看到一些人这么做，但是没有深入了解。
