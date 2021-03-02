#### 自动化部署

- ##### 准备

  - 安装Docker
  - 安装GitLab
  - 准备一个jdk

- 安装docker CI

  - ```sh
    mkdir -p /usr/lcoal/docker/runners
    mkdir -p /usr/local/docker/runner/enviroment
    
    #将准备好的jar复制到  /usr/local/docker/runner/enviroment
    cd /usr/local/docker/runner/enviroment
    vi Dockerfile
    ```

  - ```dockerfile
    FROM gitlab/gitlab-runner:v11.0.2
    
    
    RUN echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse' > /etc/apt/sources.list && \
        echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse' >> /etc/apt/sources.list && \
        echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse' >> /etc/apt/sources.list && \
        echo 'deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse' >> /etc/apt/sources.list && \
        apt-get update -y && \
        apt-get clean
    
    
    RUN apt-get -y install apt-transport-https ca-certificates curl software-properties-common && \
        curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add - && \
        add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" && \
        apt-get update -y && \
        apt-get install -y docker-ce
    
    
    COPY daemon.json /etc/docker/daemon.json
    
    
    WORKDIR /usr/local/bin
    RUN curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
    
    
    
    RUN chmod +x /usr/local/bin/docker-compose
    
    
    RUN mkdir -p /usr/local/java
    WORKDIR /usr/local/java
    COPY jdk-8u144-linux-x64.tar.gz /usr/local/java
    RUN tar -zxvf jdk-8u144-linux-x64.tar.gz && \
        rm -fr jdk-8u144-linux-x64.tar.gz
    
    
    RUN mkdir -p /usr/local/maven
    WORKDIR /usr/local/maven
    RUN  wget http://mirrors.cnnic.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz
    
    
    RUN tar -zxvf apache-maven-3.5.4-bin.tar.gz && \
        rm -fr apache-maven-3.5.4-bin.tar.gz
    
    
    
    
    ENV JAVA_HOME /usr/local/java/jdk1.8.0_144
    ENV MAVEN_HOME /usr/local/maven/apache-maven-3.5.4
    ENV PATH $PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
    
    
    WORKDIR /
    ```

  - 配置加速器和从仓库地址

    - ```shell
      vi  /usr/local/docker/runner/environment/daemon.json
      ```

    - ```
      {
        "registry-mirrors" : [
          "http://ovfftd6p.mirror.aliyuncs.com",
          "http://registry.docker-cn.com",
          "http://docker.mirrors.ustc.edu.cn",
          "http://hub-mirror.c.163.com"
        ],
        "insecure-registries" : [
          "registry.docker-cn.com",
          "docker.mirrors.ustc.edu.cn"
        ],
        "debug" : true,
        "experimental" : true
      }
      ```

  - 构建镜像

    - ```
      docker build -t gitlab-runner .
      ```

  - 启动镜像

    - ```
      docker run -d --name gitlab-runner --restart always \
         -v /usr/local/docker/runner/config:/etc/gitlab-runner \
         -v /var/run/docker.sock:/var/run/docker.sock \
         gitlab-runner
      ```

      - 查找token  一定找对项目

        - ![gitlab-ci](../image\gitlab-ci.png)

      - 注册runner

        - ```shell
          docker exec -it gitlab-runner gitlab-runner register
          
          
          # 输入 GitLab 地址
          Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
          http://36.112.173.234:18090/
          
          
          # 输入 GitLab Token  在项目的设置中  CI 
          Please enter the gitlab-ci token for this runner:
          zrPDEebaiyjrycJuqx28
          
          
          # 输入 Runner 的说明
          Please enter the gitlab-ci description for this runner:
          可以为空
          
          
          # 设置 Tag，可以用于指定在构建规定的 tag 时触发 ci
          Please enter the gitlab-ci tags for this runner (comma separated):
          可以为空
          
          
          # 选择 runner 执行器，这里我们选择的是 shell
          Please enter the executor: virtualbox, docker+machine, parallels, shell, ssh, docker-ssh+machine, kubernetes, docker, docker-ssh:
          shell
          
          
          #配置完后
          docker restart gitlab-runner
          ```

      - 配置runner免密登陆GitLab

        - ```shell
          docker exec -it gitlab-runner bash
          
          #切换用户
          su gitlab-runner
          
          生成公钥私钥
          ssh-keygen 
          
          cat /home/gitlab-runner/.ssh/id_rsa.pub
          #回车后将公钥复制到gitlab得SSH密钥中
          
          #授权用户权限 去访问 /var/run/docker.sock
          chmod -R 777 /var/run/docker.sock
          ```

  - 创建jdk镜像

    - ```
      FROM centos:7
      #2、指明该镜像的作者和电子邮箱
      MAINTAINER hanyukun
      #3、在构建镜像时，指定镜像的工作目录，之后的命令都是基于此工作目录，如果不存在，则会创建目录
      WORKDIR /usr/local/docker
      #4、一个复制命令，把jdk安装文件复制到镜像中，语法 ADD SRC DEST ,ADD命令具有自动解压功能
      ADD jdk1.8.0_144 /usr/local/docker/jdk1.8.0_144
      #5、配置环境变量，此处目录为tar.gz包解压后的名称，需提前解压知晓：
      ENV JAVA_HOME=/usr/local/docker/jdk1.8.0_144
      ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
      ENV PATH=$JAVA_HOME/bin:$PATH
      #6、设置启动命令
      CMD ["java","-version"]
      ```

      ```
      docker build -t jdk8 .
      ```

      

  - 项目地址上的根目录  .gitlab-ci.yml文件

    - ```
      stages:
        - build
        - run
        - clean
      
      
      build:
        stage: build
        script:
          - /usr/local/maven/apache-maven-3.5.4/bin/mvn clean package
          - cp crm-main/target/app.jar docker
          - cd docker
          - docker build -t app .
      
      
      run:
        stage: run
        script:
          - cd docker
          - docker-compose down
          - docker-compose up -d
      
      
      clean:
        stage: clean
        script:
          - docker rmi $(docker images -q -f dangling=true)
      ```

  - /docker目录上创建 **Dockerfile**

    - ```
      FROM jdk8
      VOLUME /tmp
      ADD app.jar app.jar
      ENTRYPOINT ["nohup","java","-jar","app.jar","--spring.profiles.active=dev","&"]
      
      ```

  - /docker目录创建docker-compose.yml

    - ```
      version: '3.1'
      services:
        itoken-config:
          restart: always
          image: app
          ports:
            - 82:82
      ```

- 注意事项

  - ```
    #如果修改了映射端口，请注意 gitlab中gitlab.yml文件的拉取代码端口正确
    #本人映射的代码位置  /root/gitlab/data/gitlab-rails/etc/gitlab.yml
    #重启docker会失效
    #修改后 进入gitlab容器中   
    docker exec -it gitlab /bin/bash
    gitlab-ctl restart
    ```

    

