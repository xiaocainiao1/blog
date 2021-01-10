#                        jenkins+gitlab+docker+maven多模块



### 1. 关于jenkins使用过程中的一些心得

- jenkins的很多功能都是依赖于插件实现的，找不到某项功能的时候先查看是不是没有安装相应的插件

- jenkins支持模块，会根据父pom找到里面的模块，会多出一个模块按钮

  ![image-20210109210338250](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109210338250.png)

点进去可以打包或者编译，但是没有其他的相关操作，最终的其他处理还需要在顶层配置

- maven的选项配置是个好东西

### 2. 测试环境的配置

![image-20210109210958497](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109210958497.png)

![image-20210109211104045](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109211104045.png)

![image-20210109211122175](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109211122175.png)

![image-20210109211138406](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109211138406.png)

![image-20210109211152851](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210109211152851.png)



**最关键的地方**,root pom :代表父工程pom文件，Goals and options:配置,要发布哪个子模块，pl后面就写那个子模块 ，pl意思是把相关配置项目一起打成jar包。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201112143904999.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNzAzMTgx,size_16,color_FFFFFF,t_70#pic_center)

### 3. 流水线关键脚本

```
pipeline {
    agent any
    environment {
        registryUrl= "172.17.66.165:8009" //搭建docker私有仓库（Harbor/nexus）或者 用DockerHub 又或者用云平台的“容器镜像服务”
        registry_user= "grid"
        registry_pass= "grid@123"
    }
    options {
        timestamps()  //设置在项目打印日志时带上对应时间
        disableConcurrentBuilds()  //不允许同时执行流水线，被用来防止同时访问共享资源等
        timeout(time: 5, unit: 'MINUTES')  // 设置流水线运行超过n分钟，Jenkins将中止流水线
        buildDiscarder(logRotator(numToKeepStr: '10'))   // 表示保留n次构建历史
    }

    //gitlab  webhook触发器
    //代码推到gitlab上后，所有子项目将被触发构建，不可取，待优化启用
    //triggers{
    //   gitlab( triggerOnPush: true,                       //代码有push动作就会触发job
    //       triggerOnMergeRequest: true,                   //代码有merge动作就会触发job
    //        branchFilterType: "NameBasedFilter",          //只有符合条件的分支才会触发构建 “All/NameBasedFilter/RegexBasedFilter”
    //        includeBranchesSpec: "${Branch_name}")      //基于branchFilterType值，输入期望包括的分支的规则
    //}

    stages{
        stage('Print Message') {   //打印信息
            steps {
                echo '打印信息'
                echo "Project_Pipeline_name: ${JOB_NAME}"
                echo "Project_module_name: ${PROJECT_NAME}"
                echo "workspace: ${WORKSPACE}"
                echo "branch: ${Branch_name}"    //gitlab分支名
                echo "build_id: ${BUILD_ID}"
                echo "target_action: ${action}"
                echo "registryUrl: ${registryUrl}"
                echo "image_repository: ${registryUrl}/${Project_name}"
           }
        }
        stage ('Checkout'){   //拉取代码
            steps{
                echo '拉取代码'
                script {
                    if ( action == 'deploy' ) {    //判断当action == 'deploy' 时，才执行此stage
                        checkout([$class: 'GitSCM', branches: [[name: '${Branch_name}']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                            userRemoteConfigs: [[credentialsId: '40ba7c18-dd55-4d21-9853-af38b353671c', //gitlab登录令牌，如何设置自行搜索方法
                            url: 'http://172.17.66.164:8098/grid/grid-boot.git']]])    //gitlab项目clone地址
                    }
                }
            }
        }
        stage('Packaging project') {        //mvn打包
            steps {
                echo 'mvn打包子项目'
                script {
                    if ( action == 'deploy' ) {
                        try {    //此步骤执行失败，将终止流水线工作
                            sh 'source /etc/profile && mvn clean install -pl ${Project_name} -am -amd -P${Profiles_name} -Dmaven.test.skip=true'
                            //“-pl”指定子项目名称 ； “-P”指定使用哪个环节的配置文件（-Ptest即表示使用文件application-test.yml配置文件打包）
                        } catch (err) {
                            echo 'Packaging project failed & End of Pipeline!!!'
                            //echo '可能原因：（1）上一次失败构建或中断构建项目后，工作目录${WORKSPACE}没有清理（可选择手动执行rm -rf ${WORKSPACE}/*）'
                        }
                    }
                }
            }
        }
		stage('Sonar-canner') {   //sonar-scanner代码检查
            steps {
                echo '代码检查'
                dir ('./') {       //指定工作目录（默认为${WORKSPACE}）
                   script {
				     try { 
					   if ( action == 'deploy' ) {
                        sh 'sonar-scanner'  //执行命令开始扫描代码(前提要maven编译生成classes文件)
                       }
					 } catch (err) {
					     echo 'Packaging project failed & End of Pipeline!!!'
                         echo echo '可能原因：（1）mvn打包子项目失败；（2）上一次失败构建或中断构建项目后，工作目录${WORKSPACE}没有清理（手动执行rm -rf ${WORKSPACE}/*）'
					 }
                    }
                }
           }
	   }
        stage('Build & Push Image to Harbor/nexus') {      //构建，推送镜像
            steps {
                echo '构建，推送镜像到docker镜像仓库'
                dir ('./') {       //指定工作目录（默认为${WORKSPACE}）
                    script {
                        if ( action == 'deploy' ) {
                            try {                               
                                //方法一：
                                sh 'docker login  --username=${registry_user} --password=${registry_pass}   ${registryUrl}'
                                sh 'cp ${Project_name}/target/*.jar ./'
                                sh 'docker build  -t ${registryUrl}/${Project_name}:${Profiles_name}-${BUILD_ID} -f ./Dockerfile_config/${Project_name}/Dockerfile . '
                                sh 'docker push ${registryUrl}/${Project_name}:${Profiles_name}-${BUILD_ID}'
                                //方法二：
                                //sh 'docker login  --username=${registry_user} --password=${registry_pass}   ${registryUrl}'
                                //sh 'cp ${Project_name}/target/*.jar ./ '
                                //def app = docker.build('${registryUrl}/${Project_name}:${Profiles_name}-${BUILD_ID} -f ./Dockerfile_config/${Project_name}/Dockerfile')
                                //app.push('${Profiles_name}-${BUILD_ID}')

                                //sh 'docker rmi ${registryUrl}/${Project_name}:${Profiles_name}-${BUILD_ID}'
                            } catch (err) {
                                echo 'Build Image failed & End of Pipeline!!!'
                                echo '可能原因：（1）mvn打包子项目失败；（2）上一次失败构建或中断构建项目后，工作目录${WORKSPACE}没有清理（手动执行rm -rf ${WORKSPACE}/*）'
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy to the Target server') {      //部署到目标服务器
            steps {
                echo '部署到目标服务器'
                script {
                    timeout(time: 60, unit: 'SECONDS') {    // 设置远程部署超过n秒，将终止该步骤
                        sh 'bash  ./Deploy.sh  ${Project_name}  ${registryUrl}/${Project_name}:${Profiles_name}  ${Profiles_name}  ${action}  ${BUILD_ID} ${rollback_id}'   //${1,2,3,4,5,6}
                }
            }
        }
		}
        //此步骤在调试Jenkinsfile时可以注释以便了解目录结构
        //亦可以忽略这步骤（有磁盘空间，任性）
        // stage('Delete Workspace') {         //清理工作目录（从jenkins上清除刚拉取的代码及mvn编译打包的内容，节省磁盘空间）
        //     steps {
        //         echo "清理工作目录: ${WORKSPACE}"
        //         deleteDir()     //表示删除当前目录(${WORKSPACE})下内容，通常用在构建完毕之后清空工作空间
        //     }
        // }
    }
 }
```

