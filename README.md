# 从 0 开始配置 DOMjudge

使用源码部署 DOMserver，使用 Docker Compose 运行 judgehost，并配置赛事题目、队伍和 ICPC Tools。

## 目录

- [0. 前置说明](#prerequisites)
- [1. 安装 DOMjudge 相关依赖](#dependencies)
- [2. 编译 DOMjudge，配置数据库与服务器](#domserver)
- [3. 配置 PHP](#php)
- [4. 安装 Docker](#docker)
- [5. 创建 judgehost 专用目录与 Compose 文件](#judgehost)
- [6. 赛事筹备](#contest-preparation)
- [7. 赛事展示与赛后滚榜](#contest-tools)

<a id="prerequisites"></a>

## 0. 前置说明

建议使用[MobaXterm](https://mobaxterm.mobatek.net/)连接Linux。在本文中，由于需要下载大量的组件（其实大小也不多），所以从官方源下载就会很慢，但是[该网站](https://linuxmirrors.cn/)就可以一键配置国内的镜像源，包括 apt 和 Docker。

DOMjudge是目前ICPC主流的官方评测平台，[DOMjudge 官网](https://www.domjudge.org/)。DOMjudge目前只能构建在Linux的系统中。

DOMjudge分为DOMserver和Judgeserver，DOMserver负责前端选手提交代码，Jury裁判、管理员统一管理。官方文档中建议DOMserver和Judgeserver分开，但是可能由于各种原因的限制，会导致DOMserver和Judgeserver不得不在同一台Linux中，那么要求机器需要强大的处理能力。

建议每 10–20 支队伍配置一个 judgehost。

> 环境需求：
>
> ![Ubuntu](https://raster.shields.io/badge/Ubuntu-26.01-blue.png?logo=ubuntu&logoColor=white) ![DOMjudge](https://raster.shields.io/badge/DOMjudge-9.0.1-white.png)
>
> 涉及技术栈
>
> ![Ubuntu](https://img.shields.io/badge/Ubuntu-yellow?logo=ubuntu&logoColor=white) ![Vim](https://img.shields.io/badge/Vim-purple?logo=Vim&logoColor=white) ![C++](https://img.shields.io/badge/C++-blue?logo=cplusplus&logoColor=white) ![C](https://img.shields.io/badge/C-blue?logo=c&logoColor=white) ![Java](https://img.shields.io/badge/Java-gray?logo=Java&logoColor=white) ![Python](https://img.shields.io/badge/Python-red?logo=Python&logoColor=white) ![PHP](https://img.shields.io/badge/PHP-pink?logo=PHP&logoColor=white) ![Mysql](https://img.shields.io/badge/Mysql-4479A1?logo=Mysql&logoColor=white) ![MariaDB](https://img.shields.io/badge/MariaDB-003545?logo=MariaDB&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?logo=Docker&logoColor=white) ![Nginx](https://img.shields.io/badge/Nginx-009639?logo=NGINX&logoColor=white)

<a id="dependencies"></a>

## 1. 安装 DOMjudge 相关依赖

```bash
sudo apt update
sudo apt install -y acl zip unzip mariadb-server nginx \
  php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
  php-curl php-json php-xml php-zip composer chrony \
  make gcc g++ debootstrap libcgroup-dev lsof procps \
  libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev vim \
  pkg-config ssh curl php-bcmath
```

同时可以在[DOMjudge 官方下载页面](https://www.domjudge.org/download)下载到DOMjudge的源代码。

```bash
sudo mv domjudge-9.0.1.tar.gz /opt/domjudge-9.0.1.tar.gz
sudo tar -zxvf domjudge-9.0.1.tar.gz
```

<a id="domserver"></a>

## 2. 编译 DOMjudge，配置数据库与服务器

```bash
cd /opt/domjudge-9.0.1
sudo ./configure --prefix=/opt/domjudge --with-domjudge-user=root --with-baseurl=127.0.0.1
sudo make domserver # 这一步会弹出一个询问，直接yes。
sudo make install-domserver
```

在这时我们需要先配置Mysql中的一些设置

```bash
sudo vim /etc/mysql/conf.d/mysql.cnf
```

你会看到一行 `[mysql]`，删掉。接着粘贴以下内容：

```ini
[mysqld]
max_connections = 1000
max_allowed_packet = 512MB
innodb_log_file_size = 2560MB
```

保存退出。

```bash
sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
```

找到 `max_allowed_packet = 1G`，取消这一行的注释。同时在`[mariadbd]`一行下再添加一行：

```ini
innodb_snapshot_isolation = OFF
```

保存退出。

```bash
sudo systemctl restart mysql
```

然后继续配置服务器。

> [!WARNING]
> 注意：在下面的配置中涉及到`php`的版本，只需要`php -v`查看即可，然后把下面的`8.x`版本换成Linux自己的

```bash
cd /opt/domjudge/domserver
sudo bin/dj_setup_database -s install
sudo ln -s /opt/domjudge/domserver/etc/nginx-conf /etc/nginx/sites-enabled/domjudge
sudo ln -s /opt/domjudge/domserver/etc/domjudge-fpm.conf /etc/php/8.x/fpm/pool.d/domjudge.conf
sudo service php8.x-fpm reload
cd /etc/nginx/sites-enabled
sudo rm default
sudo service nginx reload
cd /opt/domjudge/domserver
sudo chown www-data:www-data -R webapp/public/*
sudo systemctl daemon-reload
```

<a id="php"></a>

## 3. 配置 PHP

```bash
cd /etc/php/8.x/fpm/pool.d
sudo vim domjudge.conf
```

打开配置文件后找到 `php_admin_value[memory_limit]` 一栏，改成：

```ini
php_admin_value[memory_limit] = 1024M
```

再找到 `php_admin_value[date.timezone]` 一栏，改成：

```ini
php_admin_value[date.timezone] = Asia/Shanghai
```

保存退出。

```bash
sudo service php8.x-fpm reload
```

现在在Windows上输入`http://你的Linux的IP/domjudge`就可以看到主界面了，默认管理员账号为：`admin`，密码在通过如下代码获取：

```bash
sudo cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

<a id="docker"></a>

## 4. 安装 Docker

这里使用`Docker`进行配置。
首先在 Linux 中下载配置Docker。前文提到的 [LinuxMirrors](https://linuxmirrors.cn/)中就含有自动配置的脚本。

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.huaweicloud.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.huaweicloud.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
```

验证 Docker 可用：

```bash
sudo docker run --rm hello-world
```

<a id="judgehost"></a>

## 5. 创建 judgehost 专用目录与 Compose 文件

```bash
sudo docker pull domjudge/judgehost:latest
```

为便于管理，单独创建一个目录存放 judgehost 的 Compose 配置：

```bash
sudo mkdir -p /opt/domjudge/judgeserver
sudo cd /opt/domjudge/judgeserver
sudo vim docker-compose.yml
```

`docker-compose.yml`的文件模板如下：

```yaml
services:
  judgehost-0:
    image: domjudge/judgehost:latest
    container_name: judgehost-0
    restart: unless-stopped
    privileged: true
    cgroup: host
    network_mode: "host"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      - DAEMON_ID=0
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - JUDGEDAEMON_PASSWORD=你的judgehost密码
      - DOMSERVER_BASEURL=http://localhost/domjudge/
```

关键配置说明：

- 开头的`judgehost-0`为评测机的编号每台不同的评测机的评测机不同，其中`container_name`也是如此
- `environment`下的`DAEMON_ID`这是核心编号。简单来说，一个评测机对应于电脑的一个核心，如果该Linux存在多个核心，那么就需要多创建评测机，并且每个`DAEMON_ID`都不相同才行。可以用`nproc`代码查看核心数量
- `JUDGEDAEMON_PASSWORD`后面是你的`judgehost`密码，可以通过命令`sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret`查看到

保存退出。

```bash
sudo docker compose up -d
```

用以下命令确认在运行：

```bash
sudo docker compose ps
sudo docker ps --filter "name=judgehost"
```

<details>
<summary>示例：一台主机运行 4 个 judgehost</summary>

一台主机，5个CPU，CPU0用作系统，剩下4个给judgehost

```yaml
services:
  judgehost-1:
    image: domjudge/judgehost:latest
    container_name: judgehost-1
    restart: unless-stopped
    privileged: true
    cgroup: host
    network_mode: "host"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      - DAEMON_ID=1
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - JUDGEDAEMON_PASSWORD=你的judgehost密码
      - DOMSERVER_BASEURL=http://localhost/domjudge/
  judgehost-2:
    image: domjudge/judgehost:latest
    container_name: judgehost-2
    restart: unless-stopped
    privileged: true
    cgroup: host
    network_mode: "host"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      - DAEMON_ID=2
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - JUDGEDAEMON_PASSWORD=你的judgehost密码
      - DOMSERVER_BASEURL=http://localhost/domjudge/
  judgehost-3:
    image: domjudge/judgehost:latest
    container_name: judgehost-3
    restart: unless-stopped
    privileged: true
    cgroup: host
    network_mode: "host"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      - DAEMON_ID=3
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - JUDGEDAEMON_PASSWORD=你的judgehost密码
      - DOMSERVER_BASEURL=http://localhost/domjudge/
  judgehost-4:
    image: domjudge/judgehost:latest
    container_name: judgehost-4
    restart: unless-stopped
    privileged: true
    cgroup: host
    network_mode: "host"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      - DAEMON_ID=4
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - JUDGEDAEMON_PASSWORD=你的judgehost密码
      - DOMSERVER_BASEURL=http://localhost/domjudge/
```

</details>

<a id="contest-preparation"></a>

## 6. 赛事筹备

我们需要出题，出题的话可以在[Polygon](https://polygon.codeforces.com/)平台上配置题目，然后出完了之后在Packages界面下载`Full`的`Linux`版本，接着使用[Polygon to DOMjudge](https://github.com/cn-xcpc-tools/Polygon2DOMjudge)脚本去转化成`DOMjudge`可识别的题目，导入即可。

导入队伍与队员，这里需要准备三张表格，`groups.tsv` 、`teams.tsv` 和 `accounts.tsv`。分别代表，比赛的小组（正式队伍、打星队伍和公司队伍等等）、比赛队伍表和登录账号表。
其中比赛小组表本身组类型较少，直接手加也很轻松，这里重点写另外两种。

> [!WARNING]
> 这里所有的 `.tsv` 文件必须用 `UTF-8` 的形式保存与导入。

`teams.tsv`

| 1             | 2         | 3             | 4      | 5                     | 6                  | 7         |
| ------------- | --------- | ------------- | ------ | --------------------- | ------------------ | --------- |
| teams         | 1         |               |        |                       |                    |           |
| `external ID` | `ICPC ID` | `category ID` | `name` | `Affiliation(学校)`     | `Short Name(学校简称)` | `Country` |
| 1001          |           | participants  | Sahara | Chang Chun University | CCU                | CHN       |

第一行是列号，第二行开始才是正文，第二行为固定格式，第三行我想表达的是这一列接下来的含义是什么，第四行是例子。

`accounts.tsv`

| 1        | 2      | 3             | 4          |
| -------- | ------ | ------------- | ---------- |
| accounts | 1      |               |            |
| `Role`   | `Name` | `External ID` | `Password` |
| team     | `***`  | 1001          | mqNnCK4UH7 |

这里同理。但是需要注意的是，这里的`External ID`必须在上面的`teams.tsv`里面存在才行，并且已经导入了才行。而这个`External ID`也作为账户的登录密码（可以是字符串类型的）。

<a id="contest-tools"></a>

## 7. 赛事展示与赛后滚榜

[ICPC Tools](https://tools.icpc.global/)提供了非常多的东西可以给我们使用，我们需要下载`Contest Data Server(CDS)`、`Resolver(滚榜用)`、`Presentation Admin` 和 `Presentation Client`，注意下载的时候这几个应用的版本必须相同才行。

### 配置 CDS

在CDS目录中的`wlp\usr\servers\cds\config`文件下有个叫`cdsConfig.xml`的文件，其中含有以下代码：

```xml
<cds>
  <!-- Configure your local contest and data folder (used to find or store config files, resources, etc.) -->
  <contest path="/home/myuser/contests/test" recordReactions="false">
    <!-- Configure your CCS via REST -->
    <!--
    <ccs url="http://<IP>/domjudge/api/v4/contests/<external id>" user="" password=""/>
    -->
  </contest>
  <!--
  <domain>
    <user name="presAdmin"/>
    <user name="presentation"/>
  </domain>
  -->
</cds>

```

删掉了一些，首先上面`path`是存放的地址，这个视CDS启动的系统是`Windows`还是`Linux`自行修改。
我们注意到有一行`ccs`标签，把里面的`url`改成比赛的对应的`IP(或者域名)`和`external id`，`user`和`password`可以是`DOMjudge`的默认`admin`管理员账户，也可以自行创建一个管理员的账户，然后取消这一段的注释即可。

在这之前我们需要有个`JDK17`并且配好环境变量，注意要这么配置：

| 环境变量 | 值 |
| --- | --- |
| `JAVA_HOME` | `C:\路径\到\JDK17` |
| `Path` | `%JAVA_HOME%\bin` |

然后我们就可以启动运行脚本了，脚本在`wlp\bin\server.bat`地址，然后在该目录下的`CMD`中输入

```bat
server.bat run cds
```

在浏览器中输入`https://127.0.0.1:8443`就可以看到`CDS`的界面了。
右上角的登录账号密码位于`wlp\usr\servers\cds\config\accounts.yaml`文件内。

### 启动 Presentations

然后我们的Presentations也是，先进入Presentations的目录，然后先设置中文显示，再启动Presentations。账号密码也都在`accounts.yaml`里。

```bat
set "ICPC_FONT=Microsoft YaHei"
client.bat https://127.0.0.1:8443/api/contests/<external id> <type为public的账号> <type为public的密码>

```

其中上面的`external id`需要的是DOMjudge中比赛的`external id`。
启动界面后，在CDS的浏览器网页中，左侧边框选择`Presentation Admin`，然后就可以设置这里面的所有信息了。
如果是提交队列的话就是`Judge queue`。

### 滚榜

在滚榜前需要在`DOMjudge`中把这场比赛给`Finalize`了先才行。
然后在`resolver`文件夹里面打开`CMD`执行如下代码：

```bat
set "ICPC_FONT=Microsoft YaHei"
resolver.bat https://127.0.0.1:8443/api/contests/<external id> <admin账号> <admin密码>

```

即可。
