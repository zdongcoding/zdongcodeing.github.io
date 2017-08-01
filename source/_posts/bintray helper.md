---
title: 上传bintray简单说明
date: 2017-06-21
copyright: true
tags: bintray
categories: android
---


update to bintray

> 一键帮助你上次 bintary

> 举个例子 
  ```
    compile 'com.github.zdongcoding:bintrayhelper:0.1.0'
    含义： groupid : artifactId : version
   ```
直接上手就是干
>  然后您使用了这个项目中的[Config.gradle](https://github.com/zdongcoding/simplebintray/blob/master/SimpleBintray.gradle) 
 您就可以跳转 **准备工作** 这项
接入方法如下 ： 
  ``` 
    <!--projet 中的 build.gradle 文件中加入 -->
    classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3'
    classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5'
    <!--lib中的build.gradle-->
    apply from: 'https://raw.githubusercontent.com/zdongcoding/simplebintray/master/SimpleBintray.gradle'
    //需要终于一点 apply from: 需要放到 ext(项目详细)后面 否则报错
  ```
<!-- more -->
# 准备工作
* projet 中的 build.gradle 文件中加入 
    ```
    classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3'
    classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5'
    ```
* 在您需要打包arr并上次bintray model 中 build.gradle 添加如下

    ```
        apply plugin: 'com.jfrog.bintray'
        apply plugin: 'com.github.dcendents.android-maven'
    ```
    基本配置
    ```
        if (project.hasProperty("android")) { // Android libraries
            task sourcesJar(type: Jar) {
                classifier = 'sources'
                from android.sourceSets.main.java.srcDirs
            }

           task javadoc(type: Javadoc) {
                source = android.sourceSets.main.java.srcDirs
                classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
            }
        } else { // Java libraries
            task sourcesJar(type: Jar, dependsOn: classes) {
                classifier = 'sources'
                from sourceSets.main.allSource
            }
        }

        task javadocJar(type: Jar, dependsOn: javadoc) {
            classifier = 'javadoc'
            from javadoc.destinationDir
        }

        artifacts {
        //    archives javadocJar
            archives sourcesJar
        }

    ```
* *  继续添加
  ```
     version = libraryVersion   //① 版本号
        install {
            repositories.mavenInstaller {
                // This generates POM.xml with proper parameters
                pom {
                    project {
                        packaging 'aar'
                        // Add your description here
                        name libraryDescription    //②项目描述
                        url siteUrl
                        // Set your license
                        licenses {
                            license {
                                name 'The Apache Software License, Version 2.0'
                                url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                            }
                        }
                        artifactId artifact_Id    // ③artifactId
                        developers {
                            developer {
                                id developerId       //④ 开发者信息
                                name developerName      //④ 开发者信息
                                email developerEmail    //④ 开发者信息
                            }
                        }
                        scm {
                            connection gitUrl       //④ 开发者信息
                            developerConnection gitUrl  //④ 开发者信息
                            url siteUrl     //④ 开发者信息
                        }
                    }
                }
            }
        }

      

        // Bintray
        Properties properties = new Properties()
        properties.load(project.rootProject.file('local.properties').newDataInputStream())
        group = publishedGroupId   //⑥ groupid
        bintray {
            user = properties.getProperty("bintray.user")  //  bintray账号
            key = properties.getProperty("bintray.apikey")   //  bintray apikey

            configurations = ['archives']
            pkg {
                repo = bintrayRepo
                name = bintrayName
                desc = libraryDescription
                websiteUrl = siteUrl
                vcsUrl = gitUrl
                issueTrackerUrl = issueUrl
                licenses = allLicenses
                labels = alllabels
                publish = true
                publicDownloadNumbers = true
        //        githubReleaseNotesFile="README.md"
        //        githubRepo = 'bintray/gradle-bintray-plugin' //Optional Github repository
        //        githubReleaseNotesFile = 'README.md' //Optional Github readme file
                version {
                    desc = libraryDescription
                    gpg {
                        sign = true //Determines whether to GPG sign the files. The default is false
                        passphrase = properties.getProperty("bintray.gpg.password")
                        //Optional. The passphrase for GPG signing'
                    }
                }
            }
        }
  ```
    **以上部分都是配置到 model的 build.gradle 文件中**

#    开始操作
> 在local.properties 编写 (一般local.properties不开源的)
   ``` 
     bintray.user= xxxxxxx
     bintray.apikey=xxxxx
   ```
>  首先你需要bintray 账号 [bintray](https://bintray.com)

*  查找  bintray.apikey  
  
 ![第一步](resource/API-1.png)
 ![第二步](resource/API-2.png)

#  项目信息
 ```
 ext{
    bintrayRepo = 'maven'  //创库名称
    bintrayName = 'Bintrayhelper'  //项目名称
    publishedGroupId = 'com.github.zdongcoding'   //groupid   这个和package 没有关系
    libraryName = 'Bintrayhelper'    //
    artifact = 'Bintrayhelper'
    artifact_Id = 'bintrayhelper'
    libraryVersion = '0.1.0'    //版本号
    maturity ='Stable'     //成熟度  stable  development offical experimental
    libraryDescription = '测试  update bintray' //项目描述
    alllabels = ['android','bintray']

    /*项目地址或者issue地址*/
    siteUrl = 'https://github.com/zdongcoding/bintrayhelper'
    gitUrl = 'https://github.com/zdongcoding/bintrayhelper.git'
    issueUrl='https://github.com/zdongcoding/bintrayhelper/issues'
    
    /*一些开发者信息*/
    developerId = 'zdongcoding'
    developerName = 'zoudong'
    developerEmail = 'zoudongq1990@gmail.com'

     /*一些开源信息*/
    licenseName = 'The Apache Software License, Version 2.0'
    licenseUrl = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
    allLicenses = ["Apache-2.0"]
}
 ```

以上基本就配置好了 我们开始上传吧
  ```
    gradlew install  //第一步
    成功后执行第二步
    gradlew bintrayUpload   //第二步
  ```
  或者 ![](resource/complete-1.png)

 > model---->other---install
       
  成功后执行第二步

 > model---->publishing----bintrayUpload

 都成功后  去bintray查看

 ![](resource/upload-0.png)
 ![](resource/upload.png)
 ![](resource/upload-2.png)
 
 # 发布到jcenter
 > 到了这步 说明您已经上传成功了， 然后去bintray 中找到您上传的库 右侧有个  add to jcenter 按钮  
  如下图：
   ![add to jcenter](resource/addtojcenter.png)

   Request to include the package 'Bintrayhelper' in 'jcenter'  
   之后半天应该差不多就能收到回执消息
    
   ![success to jcenter](resource/add_success.png)

  您就可以在AS 中 添加依赖了   
> compile 'com.github.zdongcoding:bintrayhelper:0.1.0'