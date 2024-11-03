# docker compose部署superset

## 部署说明

本说明文档针对的是cenos 7.9版本。

* 第一步：创建一个用户来部署superset，例如用户名就叫superset。

  ```bash
  useradd -m superset
  ```

* 第二步：给创建的superset用户sudo权限，且执行sudo命令时不用输入密码。
* 

* 第三步：下载代码：可以在部署的服务器上直接git clone或者GitHub下载后上传到服务器指定目录

  ```bash
  git clone --depth=1  https://github.com/apache/superset.git
  ```

  通过使用 `--depth` 选项，可以告诉 Git 仅下载最近的几次提交。`--depth=1` 表示仅下载最新的一次提交。这样，克隆的仓库将不包含完整的提交历史，从而大大减少了下载数据的大小，加快了克隆速度，并降低了磁盘空间的占用。

* 第四步：进入第一步下载的superset目录里，在docker-compose-image-tag.yml所在的目录执行如下语句：

  ```bash
  # 启动superset服务，-d表示以后台模式运行
  sudo docker compose -f docker-compose-image-tag.yml up -d
  # 查看容器服务的启动情况
  sudo docker compose ps 
  ```

* 第五步：浏览器打开http://ip:8088，ip对应部署服务器的ip地址，然后输入默认用户名admin/admin登录即可。



## 安装database驱动

比如要连接clickhouse，需要安装clickhouse-driver

https://superset.apache.org/docs/configuration/databases#clickhouse

在superset的代码主目录的./docker目录下新建一个requirements-local.txt文件，里面内容如下：

```bash
echo "clickhouse-connect>=0.6.8" >> ./docker/requirements-local.txt
docker compose -f docker-compose-image-tag.yml down 
# 执行up命令会下载requirements-local.txt里的clickhouse driver
docker compose -f docker-copose-image-tag.yml up -d 
```



## 设置开机自启动

通过systemd来设置superset开机自启动

* 执行docker compose命令的用户加到docker组，执行docker命令就不用在前面加sudo

  ```bash
  sudo usermod -aG docker superset
  ```

* 创建superset.service文件，路径是：/etc/systemd/system/superset.service

  ```bash
  [Unit]
  Description=Superset service with Docker Compose
  Requires=docker.service
  After=docker.service network.target
  
  [Service]
  Type=oneshot
  RemainAfterExit=yes
  User=superset
  # 如果你的服务需要一个特定的组，可以取消注释并修改下一行
  # Group=superset
  WorkingDirectory=/home/superset/superset/superset-master
  # 确保这里的路径正确指向了你的Docker和Docker Compose的安装位置
  ExecStart=/usr/bin/docker compose -f docker-compose-image-tag.yml up -d
  ExecStop=/usr/bin/docker compose -f docker-compose-image-tag.yml down
  
  [Install]
  WantedBy=multi-user.target
  ```

* 让superset.servivce生效

  ```bash
  sudo systemctl daemon-reload
  ```

* 设置开机自启动

  ```bash
  sudo systemctl enable superset # 设置开机自启动
  sudo systemctl is-enabled superset # 查看设置是否生效
  ```

* 使用systemctl命令来进行服务启停

  ```bash
  sudo systemctl stop superset
  sudo systemctl start superset
  sudo systemctl restart superset
  docker compose ps ## docker compose命令要在yml文件的当前目录执行
  ```

* 然后就可以关机再开机来验证superset是否正常启动了。



## 日常运维

* 服务启停

  ```bash
  # 停止容器
  docker compose stop
  # 启动容器
  docker compose start
  # 停止并移除容器，移除由 docker-compose up 创建的网络，但保留卷。
  sudo systemctl docker compose down
  # 除了停止并移除容器、移除网络外，还会移除所有相关的数据卷。
  sudo systemctl docker compose up
  ```

  

* 查看容器运行情况

  ```bash
  docker compose ps
  ```

  

* 查看容器日志：参数service_name是容器的服务名，对应的是docker compose ps里的SERVICE这一列的值

  ```bash
  docker compose logs service_name
  ```

  

## 常见问题

* docker compose stop/start和docker compose down/up区别

  stop/start是停止/启动容器，down/up是删除容器及容器里的网络/重建容器及容器里的网络

* docker compose down和docker compose down -v区别

  后者的v是指volume，会删除容器里的数据卷，如果容器里的数据要保留要切记先备份再操作。前者没有-v是不会删除容器里的数据。

* superset通过docker compose部署，db这个service的日志里显示 ` ls: cannot open directory '/docker-entrypoint-initdb.d/': Permission denied`

  可以看docker-compose-image-tag.yml里db的配置如下：

  ```bash
  db:
      env_file:
        - path: docker/.env # default
          required: true
        - path: docker/.env-local # optional override
          required: false
      image: postgres:15
      container_name: superset_db
      restart: unless-stopped
      volumes:
        - db_home:/var/lib/postgresql/data
        - ./docker/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
  ```

  可以看出/docker-entrypoint-initdb.d对应的是宿主机的./docker/docker-entrypoint-initdb.d路径，修改这个路径的权限如下即可：

  ```bash
  chmod 755 -R ./docker/docker-entrypoint-initdb.d/
  ```

* docker compose -f docker-compose-image-tag.yml提示：`services.db.env_file.0 must be a string`。

  这是因为docker compose版本比较旧，不支持yml的新语法，需要升级docker和docker compose，参考下面的步骤来。

* 查看docker和docker compose版本

  ```bash
  docker info
  docker version
  docker compose version
  ```

* docker更新

  centos 7.9已经不支持清华的yum源了。

  ```bash
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  sudo yum update docker
  ```

  

* docker compose更新：docker compose是docker的一个plugin插件，按照如下方式指定docker compose版本进行更新

  ```bash
  sudo mkdir -p /usr/local/lib/docker/cli-plugins
  sudo curl -SL https://github.com/docker/compose/releases/download/v2.30.1/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
  sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
  ```

  

## 开源地址

文章和示例代码开源地址在GitHub: https://github.com/jincheng9/disributed-system-notes

公众号：coding进阶

个人网站：https://jincheng9.github.io/



## References

*  https://docs.docker.com/engine/reference/builder/ 
*  https://eddycjy.com/posts/go/gin/2018-03-24-golang-docker/