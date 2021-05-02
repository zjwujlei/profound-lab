Android Pipline代码扫描搭建
===========================

Android侧预计将代码静态扫描、代码单元测试及UI自动化测试作为一个Pipline，做代码自动化扫描。

### Android 环境搭建

### 代码静态扫描

静态代码扫描使用Android Lint，团队根据Android侧代码规范及常问题编写了Lint扫描规则作为依赖库导入，无需额外配置环境。

项目中依赖WYLint库
```
    implementation 'com.profound.android:lint:1.0.0'//根据实际情况组织GAV
```

通过以下命令执行lint扫描

```
./gradlew lint
```

### 单元测试

单元测试使用junit实现，无需额外配置环境。

项目中依赖单元测试相关库
```
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:0.5'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:2.2.2'
}
```

通过以下命令执行lint扫描

```
./gradlew testDebugUnitTest
```

###### 测试用例的简单编写

### UI自动化

UI自动化测试使用Appium框架+python实现。

linux下安装appium需要用如下命令：

```
sudo apt-get update
sudo apt-get install nodejs
sudo apt-get install npm
npm install -g appium
```

配置 python驱动


### Pipline 脚本

```groovy
#!groovy
pipeline{
    agent {
        label 'xxx.xxx.xx.xxx'//IP
    }
    parameters {
        string(name:'repoBranch', defaultValue: '${defaultBranch}', description: 'git分支名称')
        string(name:'lineCoverage', defaultValue: '20', description: '单元测试代码覆盖率要求(%)，小于此值pipeline将会失败！')
    }
    tools{
        gradle 'Gradle2.14.1'
        jdk 'jdk8'
    }
    environment{
        REPO_URL='xxx'//git地址
        //git服务全系统只读账号，无需修改
        CRED_ID='private-token' //填入token
        QA_EMAIL='email' //填入email
    }
    //pipeline运行结果通知给触发者
    post{
        success{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL }",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER}) result",
                            body: "${BUILD_USER}'s pineline '${JOB_NAME}' (${BUILD_NUMBER}) run success\n请及时前往${env.BUILD_URL}进行查看"
                }
            }
        }
        failure{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL }",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER}) result",
                            body: "${BUILD_USER}'s pineline  '${JOB_NAME}' (${BUILD_NUMBER}) run failure\n请及时前往${env.BUILD_URL}进行查看"
                }
            }

        }
        unstable{
            script {
                wrap([$class: 'BuildUser']) {
                    mail to: "${BUILD_USER_EMAIL }",
                            subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER})结果",
                            body: "${BUILD_USER}'s pineline '${JOB_NAME}' (${BUILD_NUMBER}) run unstable\n请及时前往${env.BUILD_URL}进行查看"
                }
            }
        }
    }

    stages{
        stage('源码获取'){
            steps{
                echo "starting fetchCode from ${REPO_URL}......"
                // Get some code from a GitHub repository
                git credentialsId:CRED_ID, url:REPO_URL, branch:params.repoBranch
            
                sh "cp /home/xx/xxx/android/local.properties  ${WORKSPACE}/local.properties"//拷贝服务器中的local.properties文件
            }
        }
        stage('单元测试'){
            steps{
                sh "./gradlew testDebugUnitTest"
            }
        }
        stage('静态检查'){
            steps{
                sh "./gradlew lint"
            }
        }
        
        stage('UI自动化测试 ') {
            steps{
                echo "starting performanceTest......"
                //    TODO
            }
        }
        
        stage('通知人工验收'){
            steps{
                script{
                    wrap([$class: 'BuildUser']) {
                        mail to: "${QA_EMAIL}",
                                subject: "PineLine '${JOB_NAME}' (${BUILD_NUMBER})人工验收通知",
                                body: "${BUILD_USER}提交的PineLine '${JOB_NAME}' (${BUILD_NUMBER})进入人工验收环节\n请及时前往${env.BUILD_URL}进行测试验收"
                    }
                }
            }
        }
    }
}
```








