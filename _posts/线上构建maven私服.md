#                                    实验室前后端私服部署手册

# 1、线上构建maven私服

- 网络环境构建nexus3镜像，导出nexus3.tar
- 创建挂载文件夹 ： mkdir -p /usr/local/maven/nexus-data
- 修改权限：chmod 777 -R /usr/local/maven/nexus-data
- 运行容器：docker run -d -p 8998:8998 --name nexus -v /usr/local/maven/nexus-data/nexus-data:/nexus-data  grid_sonatype/nexus3:v1
- 查看日志：tail -f /usr/local/maven/nexus-data/log/nexus.log

## 2、私服批量上传

- 进入当前需要部署的仓库根目录 例如：![image-20210408091231440](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\image-20210408091231440.png)

- 构建mavenimport.sh批量上传脚本

  ```
  #!/bin/bash
  # copy and run this script to the root of the repository directory containing files
  # this script attempts to exclude uploading itself explicitly so the script name is important
  # Get command line params
  while getopts ":r:u:p:" opt; do
      case $opt in
          r) REPO_URL="$OPTARG"
          ;;
          u) USERNAME="$OPTARG"
          ;;
          p) PASSWORD="$OPTARG"
          ;;
      esac
  done
  
  find . -type f -not -path './mavenimport\.sh*' -not -path '*/\.*' -not -path '*/\^archetype\-catalog\.xml*' -not -path '*/\^maven\-metadata\-local*\.xml' -not -path '*/\^maven\-metadata\-deployment*\.xml' | sed "s|^\./||" | xargs -I '{}' curl -u "$USERNAME:$PASSWORD" -X PUT -v -T {} ${REPO_URL}/{} ;
  ```

- 运行部署命令

  ```
  ./mavenimport.sh -u admin -p Hndl@2021 -r http://192.168.100.12:8081/repository/grid_repo/
  ```

  

