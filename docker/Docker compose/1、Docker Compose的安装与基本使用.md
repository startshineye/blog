
### 1、多容器的App的问题
  因为docker compose是docker的命令行工具；我们是需要安装的，
在桌面操作系统：比如：mac、window上的机器，我们的docker-compose是默认自动安装好的。  
![](../images/07.png)  

```renderscript
docker-compose --version
docker-compose version 1.29.2, build 5becea4c
```

我们的docker compose是参考如下:https://docs.docker.com/compose/  
我们参考如下:https://docs.docker.com/compose/install/compose-plugin/ 安装docker-compose；  

###### 1、To download and install the Compose CLI plugin, run:
```renderscript
 DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
 mkdir -p $DOCKER_CONFIG/cli-plugins
 curl -SL https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
```

  安装完之后，我们发现docker-compose支持的command是非常多的，有如下种类：
![](../images/08.png)   

一般我们使用docker-compose都是结合我们的yml文件来进行的。  

我们先看下如下的配置文件:

```renderscript
version: '3'

services:

  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: root
    networks:
      - my-bridge

  mysql:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-bridge

volumes:
  mysql-data:

networks:
  my-bridge:
    driver: bridge
```

#### 1、docker-compose up
   docker-compose up是将我们的yml文件里面的容器启动起来。
   默认情况下我们敲命令:docker-compose up 是直接寻找我们当前目录下的docker-compose.yml文件，他的作用等同于:
```renderscript
docker-compose -f docker-compose.yml up
```
 我们使用上面的docker-compose.yml启动容器的时候，我们发现了如下的信息。  
 

