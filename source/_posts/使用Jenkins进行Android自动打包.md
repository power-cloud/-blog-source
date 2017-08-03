---
title: 使用Jenkins进行Android自动打包
date: 2017-08-03 11:27:38
tags: [Jenkins,Android]
---

之前App在提交测试和最终部署的过程中App打包一直是由开发人员来完成的，由于项目比较大， 再加上Android打包本身就比较慢，所以每次打包还是很耗时的。并且按照严格的研发流程来讲，开发人员应该只负责提交代码，测试和部署过程中的打包都不应该由开发人员来完成，所以我就想着给测试和运维人员搭建一个可以自动打包的环境。后来在网上看到很多网友分享使用Jenkins进行Android自动打包的文章，几经尝试终于把环境搭建起来了。

<!--more-->
## Jenkins安装
Jenkins作为一个开源的持续集成工具，不仅可以用来进行Android打包，也可以用来进行iOS打包、NodeJs打包、Jave服务打包等。官方地址为:[https://jenkins.io/](https://jenkins.io/)。Jenkins是使用Java开发的，官方提供一个war包，并且自带servlet容器，可以独立运行也可以放在Tomcat中运行。我们这里使用独立运行的方式。运行命令为：
```code
java -jar jenkins.war
```
运行成功，打开浏览器访问`http://locahost:8080`,首次运行会要求输入管理员密码，Jenkins在首次运行时生成的，会在控制台打印出来或者按照页面提示的文件路径查看管理员密码。控制台输出的密码:
```code
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

b7004e63acb940368e62a5dacaa2b246

This may also be found at: /Users/dmx/.jenkins/secrets/initialAdminPassword
```
第一次运行的页面

![Jenkins第一次运行](/images/jenkins_first_load.png)

输入密码之后点击continue选择要安装的插件

![Jenkins安装插件](/images/jenkins_install_plugin.png)

由于Jenkins的插件之间存在依赖关系，并且Jenkins不会帮我们自动安装依赖的插件，所以插件安装过程比较容易出错，所以我们建议自己选择要安装的插件，不选择Jenkins建议安装的插件。点击`Select plugins to install `进入下一个页面

![Jenkins选择插件](/images/jenkins_select_plugin.png)

首先先把默认选中的插件都取消掉，然后选择我们要安装的插件，对于Android打包来讲一般需要的插件有
- Git plugin
- Gradle Plugin
- Email Extension Plugin
- description setter plugin
- build-name-setter
- user build vars plugin
- Post-Build Script Plug-in
- Branch API Plugin
- SSH plugin
- Scriptler
- Git Parameter Plug-In
- Gitlab plugin

如果插件安装过程中由于依赖关系造成安装失败，可以根据错误信息先安装依赖的插件再重新安装需要的插件。

插件安装完成之后按照提示创建一个管理员账号即可使用，登录之后进行首页面。

![Jenkins首页](/images/jenkins_main.png)
## 配置环境变量
需要配置的环境变量有Android Home、JDK目录、Gradle目录。首先点击*系统管理*=>*系统设置*，选中`Environment variables`，然后新增Android Home环境变量

![Android Home配置](/images/jenkins_android_home.png)

然后在*系统管理*=>*Global Tool Configuration*中配置JDK目录和Gradle目录

![Gradle配置](/images/jenkins_gradle.png)

JDK和Gradle建议提前下载好放到服务器上，不要使用自动安装，Jenkins自动下载安装非常慢

## 配置打包脚本
Jenkins配置完成之后需要我们来完善我们的gradle脚本让它能够满足我们的打包要求，既能支持在Jenkins中打包，也能支持我们使用Android Studio进行打包。首先我们需要一个变量`IS_JENKINS`用来标识当前是在Jenkins中打包还是在Android Studio中打包，在不同环境下打包时证书的路径和APK生成的路径不同，我们定义一个函数来获取证书路径，然后在gradle中指定打包时使用的证书
```code
def getMyStoreFile(){
    if("true".equals(IS_JENKINS)){
        return file("使用Jenkins打包时的证书路径")
    }else{
        return file("使用Android Studio打包时证书路径")
    }
}
android{
  signingConfigs {
        release {
            keyAlias '*****'
            keyPassword '****'
            storeFile getMyStoreFile()
            storePassword '****'
        }
    }
    buildTypes{
      debug{
        ....
        signingConfig signingConfigs.release
      }
      release{
        ....
        signingConfig signingConfigs.release
      }
    }
    ....
}
```
然后配置不同打包环境下apk的生成路径
```code
   android.applicationVariants.all { variant ->
        variant.outputs.each { output ->
            //新名字
            def newName
            //输出文件夹
            def outDirectory
            //是否为Jenkins打包，输出路径不同
            if ("true".equals(IS_JENKINS)) {
                //BUILD_PATH为服务器输出路径
                outDirectory = BUILD_PATH
                newName = "你的应用名称" + "-" + defaultConfig.versionName + "-" + BUILD_TYPE + ".apk"
            } else {
                outDirectory = output.outputFile.getParent()
                newName = "你的应用名称" + "-" + defaultConfig.versionName + "-" + BUILD_TYPE + ".apk"
            }
            output.outputFile = new File(outDirectory, newName)
        }
    }
```
最总完成的gradle脚本为
```code
apply plugin: 'com.android.application'
repositories {
    flatDir {
        dirs 'libs'
    }
}

dependencies {
   ....
}
def getMyStoreFile(){
    if("true".equals(IS_JENKINS)){
        return file("使用Jenkins打包时的证书路径")
    }else{
        return file("使用Android Studio打包时证书路径")
    }
}
android {
      signingConfigs {
        release {
            keyAlias '*****'
            keyPassword '****'
            storeFile getMyStoreFile()
            storePassword '****'
        }
    }
    compileSdkVersion Integer.parseInt(project.ANDROID_BUILD_SDK_VERSION)
    buildToolsVersion project.ANDROID_BUILD_TOOLS_VERSION
    dexOptions {
        jumboMode true
    }
    defaultConfig {
        applicationId project.APPLICATION_ID
        minSdkVersion Integer.parseInt(project.ANDROID_BUILD_MIN_SDK_VERSION)
        targetSdkVersion Integer.parseInt(project.ANDROID_BUILD_TARGET_SDK_VERSION)
        versionName project.APP_VERSION
        versionCode Integer.parseInt(project.VERSION_CODE)
        ndk {
            abiFilters "armeabi", "armeabi-v7a", "arm64-v8a", "mips", "mips64", "x86", "x86_64"
        }
        // Enabling multidex support.
        multiDexEnabled true
    }
    buildTypes {
        debug {
            minifyEnabled false
            shrinkResources false
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        release {
            // 移除无用的resource文件
            shrinkResources true
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    android.applicationVariants.all { variant ->
        variant.outputs.each { output ->
            //新名字
            def newName
            //输出文件夹
            def outDirectory
            //是否为Jenkins打包，输出路径不同
            if ("true".equals(IS_JENKINS)) {
                //BUILD_PATH为服务器输出路径
                outDirectory = BUILD_PATH
                newName = "你的app名字" + "-" + defaultConfig.versionName + "-" + BUILD_TYPE + ".apk"
            } else {
                outDirectory = output.outputFile.getParent()
                newName = "你的app名字" + "-" + defaultConfig.versionName + "-" + BUILD_TYPE + ".apk"
            }
            output.outputFile = new File(outDirectory, newName)
        }
    }
    flavorDimensions("channel")
    productFlavors {
        yingyongbao { dimension "channel" }
    }
    productFlavors.all {
        flavor -> flavor.manifestPlaceholders = [CHANNEL_VALUE: name]
    }
    packagingOptions {
        exclude 'META-INF/DEPENDENCIES.txt'
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
        exclude 'META-INF/NOTICE'
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/DEPENDENCIES'
        exclude 'META-INF/notice.txt'
        exclude 'META-INF/license.txt'
        exclude 'META-INF/dependencies.txt'
        exclude 'META-INF/LGPL2.1'
    }

}
```
gradle脚本中使用了在gradle.properties中定义的变量，gradle.properties内容如下
```code
org.gradle.daemon=true
org.gradle.parallel=true
manifestmerger.enabled=true
android.useDeprecatedNdk=true
org.gradle.configureondemand=true
org.gradle.jvmargs=-Xmx4096m -XX\:MaxPermSize\=4096m -XX\:+HeapDumpOnOutOfMemoryError -Dfile.encoding\=UTF-8

ANDROID_BUILD_MIN_SDK_VERSION=14
ANDROID_BUILD_TOOLS_VERSION=25.0.1
ANDROID_BUILD_TARGET_SDK_VERSION=22
ANDROID_BUILD_SDK_VERSION=24
VERSION_CODE=176
APPLICATION_ID=你的applicationId

#jenkins中用到的变量
NODEJS_ADDRESS=app要访问的服务器地址
API_VERSION=api版本号
APP_VERSION=app版本号
IS_JENKINS=false
BUILD_PATH=apk输出路径
BUILD_TYPE=Debug

```
## 创建Job
经过上面对gradle的配置我们已经做好了准备工作，现在需要在Jenkins上新建一个任务来完成对上面脚本的调用。

在Jenkins中点击*新建*，输入Job名字，由于Jenkins会根据Job名字生成目录所以建议使用英文不要使用中文，然后选择*构建一个自由风格的软件项目*，然后点击*OK*进入配置页面

![Job配置](/images/jenkins_config_job.png)

Job配置一共分为六个部分：General、源码管理、构建触发器、构建、构建后操作。

### General
General中可以配置Job的基本信息，名字、描述等信息，我们需要关注的是关于构建的配置，如果服务器资源比较紧张可以选择*丢弃旧的构建*，然后选中*参数化构建过程*，这样就能够在打包的时候输入一些必要的参数，比如App版本号、打包类型、服务器地址、渠道等信息，这些输入参数会在构建过程中替换掉gradle.properties中定义的变量。Jenkins中支持的参数类型有Boolean、Choice（下拉选择形式的）、String、Git(需要安装插件)。网上其他文章中提到的`Dynamic Parameter Plug-in`由于安全性问题已经不再支持。下面看一下我们需要添加参数:

![](/images/jenkins_param1.png)

BUILD_TYPE表示构建版本是Release版还是Debug版，这样可以区分App是正式版本还是内容测试版本。JS_JENKINS表示这是从Jenkins打包的，默认值为true

![](/images/jenkins_param2.png)

PRODUCT_FLAVORS表示App的渠道，我们目前只设置了应用宝这个一个渠道，如果渠道包多的话这样打包效率比较低，需要一个专门进行多渠道打包的工具。APP_VERSION表示APP的版本号，这里添加这个参数是为了能够让运维人员在App发布时能够指定发布的版本号。

![](/images/jenkins_param3.png)

GIT_TAG用于在打包时选择使用仓库上哪个分支或者TAG，其中Parameter Type可以选择Tag、Branch、Branch or Tag或者revision，这里我们选择Branch or Tag

![](/images/jenkins_param4.png)

NODEJS_ADDRESS表示服务器地址，这里可以配置上测试环境、生产环境地址，在打包时选择要哪个后台服务。

![](/images/jenkins_param5.png)

REMARK用来描述本次打包的版本，比如这次打包使用来验证哪个问题等等，要不然单凭版本号很难想起当时打包这个版本是用来干什么的。

### 源码管理

我们公司使用Gitlab进行代码管理，这里选择git，然后输入仓库地址，并在Branch Specifier绑定GIT_TAG变量，这样GIT_TAG会自动读取仓库上的分支和TAG列表。

![源码配置](/images/jenkins_scm.png)

### 构建触发器

构建触发器用来配置什么时候触发构建，一般做法有手动触发、定时触发、或者提交代码时触发。提交代码触发需要在gitlab中添加webhook，我们这里使用手动触发所以这里不做配置

### 构建环境

通过选中`Set Build Name`设置构建名称，我们这里设置名称为
```code
#${BUILD_NUMBER}_${BUILD_USER}_${APP_VERSION}_${BUILD_TYPE}
```
在Jenkins中${}表示引用变量,其中BUILD_NUMBER为构建编号，为Jenkins提供的变量；BUILD_USER为构建人，即当前登录用户，需要选中`Set jenkins user build variables`；APP_VERSION为App版本号;BUILD_TYPE为构建类型。一个实际的构建名称为`#14_admin_1.2_Release`,表示第14次构建，构建人为admin,构建的App版本为1.2Release版本

![](/images/jenkins_build_env.png)

### 构建

![](/images/jenkins_build1.png)

选中`invoke gradle`通过调用gradle脚本进行构建，选择在系统管理中配置的gradle的版本，这里为gradle4.0

然后在Tasks输入打包命令
```code
clean assemble${PRODUCT_FLAVORS}${BUILD_TYPE}
```
首先执行clean，然后执行assemble进行打包。以PRODUCT_FLAVORS选择yingyongbao,BUILD_TYPE为Release为例，则实际执行的命令为
```code
clean assembleYingyongbaoRelease
```
然后选中`	Pass job parameters as Gradle properties`这样才能将我们自定义参数在打包时传递到gradle脚本中

这样我们就能成功打包出apk了

## 实现二维码下载

为了能够更方便的使用，我们还应该提供一个二维码功能，这样手机扫描之后就能下载安装。一般做法有两个：一是选择将打包出来的apk上传到第三方平台；另一个是本地搭建一个服务，实现静态文件服务器的功能。我们这里选择在本地服务器搭建一个静态文件服务，同时将文件地址生成一个二维码展示出来。

![](/images/jenkins_build2.png)

在Excute Shell中输入在构建完成之后执行的脚本，根据apk路径生成一个二维码
```code
node /opt/jenkins_node/qr.js http://10.1.170.154:3000/apk/yundiangong-${APP_VERSION}-${BUILD_TYPE}.apk /opt/jenkins_node/apk/yundiangong-${APP_VERSION}-${BUILD_TYPE}.png
```
即通过node 执行/opt/jenkins_node（需要根据自己实际的目录设置）下的qr.js文件，同时传递两个参数，第一个参数文件apk文件访问路径，我在gradle打包脚本中设置apk输出路径为/opt/jenkins_node/apk目录，通过静态文件服务的访问地址`http://10.1.170.154:3000/apk/yundiangong-${APP_VERSION}-${BUILD_TYPE}.apk`（10.1.170.154为我们公司内部服务器，需要根据自己情况设置）；第二个参数为生成二维码的保存路径，同样为/opt/jenkins_node/apk目录，这样静态文件服务既可以提供apk下载，也可以提供二维码下载。

然后通过设置build description显示二维码功能,通过定义一个html片段，需要在*系统管理*=>*Configure Global Security*中将`Markup Formatter`选择为`Safe HTML`
```code
<img src="http://10.1.170.154:3000/apk/yundiangong-${APP_VERSION}-${BUILD_TYPE}.png" ><br> <a target="_blank" href="http://10.1.170.154:3000/apk/yundiangong-${APP_VERSION}-${BUILD_TYPE}.apk">点击下载</a><p>${REMARK}</p>
```
这样构建成功之后会展示一个二维码，同时提供一个点击下载的链接，并且还会展示该构建版本的描述信息

我们使用nodeJs实现一个静态文件服务，通过nodejs启动一个http服务，然后通过解析请求返回对应的apk文件。代码如下
```code
const http = require('http')
const path = require('path')
const url = require('url')
const fs = require('fs')
const mime = require('mime')

const port = '3000'
const server = http.createServer((req, res) => {
  if (req.url === '/') {
    res.end('Hello World')
    return
  }
  if (req.url === '/favicon.ico') return //不响应favicon请求

  // 获取url->patnname 即文件名
  let pathname = path.join(__dirname, url.parse(req.url).pathname)
  pathname = decodeURIComponent(pathname) // url解码，防止中文路径出错
  if (fs.existsSync(pathname)) {
    if (!fs.statSync(pathname).isDirectory()) {
      // 以binary读取文件
      fs.readFile(pathname, 'binary', (err, data) => {
        if (err) {
          res.writeHead(500, { 'Content-Type': 'text/plain' })
          res.end(JSON.stringify(err))
          return false
        }
        res.writeHead(200, {
          'Content-Type': `${mime.lookup(pathname)};charset:UTF-8`
        })
        res.write(data, 'binary')
        res.end()
      })
    } else {
      res.statusCode = 404;
      res.end('Directory Not Support')
    }

  } else {
      res.statusCode = 404;
      res.end('File Not Found')
  }
});
server.listen(port);
```

生成二维码的小程序也是使用nodejs实现，通过使用qr-image模块实现生成二维码功能
```code
const qr=require('qr-image')
const  args = process.argv.splice(2);
const filePath=args[0]//源文件地址
const distPath=args[1]//目标文件地址
const img=qr.image(filePath,{size:5})//生成二维码图片
img.pipe(require('fs').createWriteStream(distPath));//保存图片
```
代码完整地址为：[https://github.com/dumingxin/jenkinsNode.git](https://github.com/dumingxin/jenkinsNode.git),首先需要安装nodejs，然后在代码目录执行`npm install`
执行`node web.js`启动静态文件服务即可。如果想后台运行可以使用pm2启动web.js

最后打包成功之后的效果

![](/images/jenkins_final.png)