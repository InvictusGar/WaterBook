# gradle插件上传maven遇到的问题，java.lang.NoClassDefFoundError: Could not initialize class xxx

项目中需要将三方提供的gragle-plugin上传至私有的maven方便使用和管理，顺便了解了下maven相关的知识。

[maven介绍](https://www.trinea.cn/android/maven/)
[pom文件详解](https://www.cnblogs.com/hafiz/p/5360195.html)

问题背景：首先按照如下配置上传

```
configurations {
    resultArchives
}

artifacts{
    resultArchives file: file('*.jar')
}

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: '***') {
                authentication(userName: '***', password: '***')
            }
            snapshotRepository(url: '***') {
                authentication(userName: '***', password: '***')
            }

            pom.project {
                artifactId = ArtifactId
                version = Version
                name = ArtifactId
                packaging = 'aar'
                groupId = 'com.xxx.test'
            }
        }
    }
}
```

上传成功后在主工程引用，sync正常，不过编译时报错
```
java.lang.NoClassDefFoundError: Could not initialize class xxx
```

由于该插件在编译过程中有transform注入过程，所以应该是注入时加载不到，但是在Studio cache中是可以找到这个plugin已经下载下来的了。

一时间没有头绪，后来发现我上传的这个包的pom文件中只有该包的信息；而这个原始包中还包括一些dependency依赖库！
于是在gradle文件中添加：


```
//只是例子
compile "com.alibaba:fastjson:1.2.49"
```

这样上传后的pom文件中也包含了dependency，再在主工程引用后，编译正常。

另外，如果希望编译源码，要按照如下配置：


```
// 指定编码
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

// 打包源码
task sourcesJar(type: Jar) {
    from android.sourceSets.main.java.srcDirs
    classifier = 'sources'
}

artifacts {
    archives sourcesJar
}

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: '***') {
                authentication(userName: '***', password: '***')
            }
            snapshotRepository(url: '***') {
                authentication(userName: '***', password: '***')
            }

            pom.project {
                artifactId = ArtifactId
                version = Version
                name = ArtifactId
                packaging = 'aar'
                groupId = 'com.xxx.test'
            }
        }
    }
}
```