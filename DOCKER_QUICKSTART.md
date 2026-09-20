# CityHub 本地一键启动（Docker Compose）

> 本文档目标：**在你自己的电脑上从零把项目跑起来**，全程只需安装 Docker Desktop + JDK + Maven，
> MySQL / Redis / RocketMQ 三个中间件全部由 `docker-compose.yml` 拉起，**不需要在本机安装**。
>
> 适用环境：Windows / macOS / Linux 均可，命令以 Windows + Git Bash / PowerShell 为主。

---

## 目录

- [1. 项目与技术栈](#1-项目与技术栈)
- [2. 前置条件](#2-前置条件)
- [3. 开始之前需要配置的 3 件事](#3-开始之前需要配置的-3-件事)
- [4. 启动依赖容器](#4-启动依赖容器)
- [5. 初始化数据库](#5-初始化数据库)
- [6. 启动应用](#6-启动应用)
- [7. 验证是否跑通（冒烟测试）](#7-验证是否跑通冒烟测试)
- [8. 配置详解：为什么这么配](#8-配置详解为什么这么配)
- [9. 日常运维命令](#9-日常运维命令)
- [10. 常见问题排查](#10-常见问题排查)
- [11. 停止与彻底清理](#11-停止与彻底清理)

---

## 1. 项目与技术栈

CityHub 是一个「类大众点评 + 优惠券秒杀」的后端 Demo 项目，**只有后端 API**（项目内没有 `static/` 与 `templates/`），
所有接口通过 HTTP 直接调用（Postman / Apifox / curl 均可）。

| 组件 | 版本 | 用途 |
|---|---|---|
| Spring Boot | 3.2.5 | 主框架 |
| Java | 17（`pom.xml` 中 `java.version=17`） | 实测 JDK 21 亦可运行 |
| MyBatis-Plus | 3.5.5 | ORM |
| MySQL | 8.0 | 权威数据源 |
| Redis | 7.0 | 缓存 / 分布式锁 / 秒杀库存 / GEO / BitMap |
| RocketMQ | 5.1.4 | 秒杀订单异步落库、延迟关单 |
| Redisson | 3.13.6 | 分布式锁 |
| Caffeine | — | 本地缓存 |

**容器与端口总览**

| 容器名 | 镜像 | 宿主机端口 → 容器端口 | 数据卷 | 说明 |
|---|---|---|---|---|
| `cityhub-mysql` | `mysql:8.0` | **13306** → 3306 | `mysql-data` | 特意避开本机可能已存在的 3306 |
| `cityhub-redis` | `redis:7.0` | 6379 → 6379 | `redis-data` | 已开启 AOF |
| `cityhub-rmq-namesrv` | `apache/rocketmq:5.1.4` | 9876 → 9876 | — | 注册中心 |
| `cityhub-rmq-broker` | `apache/rocketmq:5.1.4` | 10911 / 10909 → 同 | `rmq-broker-store` | 消息存储 |
| *(应用本身)* | 本机 JVM | 8081 | — | 不在容器里，用 `mvn spring-boot:run` 启动 |

> 所有账号密码统一为 `123456`，已与 `src/main/resources/application.yaml` 对齐，无需改动。

---

## 2. 前置条件

| 依赖 | 要求 | 检查命令 |
|---|---|---|
| **Docker Desktop** | 已安装并**处于运行状态** | `docker version` |
| **JDK** | 17 及以上 | `java -version` |
| **Maven** | 3.6 及以上（或用 IDEA 自带的 Maven） | `mvn -v` |
| **Git** | 任意 | `git --version` |

**端口占用自查**（启动前跑一遍，确认这些端口没被别的程序占用）：

```bash
netstat -ano | findstr ":13306 :6379 :9876 :10911 :10909 :8081"
```

Windows 上没有输出即表示这些端口都空闲。

> ⚠️ **如果你本机已经装了 MySQL 并占用 3306，不影响本项目** —— 容器里的 MySQL 映射到宿主机的 **13306**，
> 两者互不干扰，你可以同时用 Navicat / MySQL Workbench 连本机 3306，用命令行连容器 13306。

---

## 3. 开始之前需要配置的 3 件事

### 3.1 打开 Docker Desktop

开始菜单搜索 **Docker Desktop** 并打开，等任务栏鲸鱼图标**变绿**（首次启动约 1~2 分钟）。

验证：

```bash
docker ps
```

能打印出容器列表（哪怕是空表头）就说明 daemon 已就绪。
若报 `failed to connect to the docker API at npipe://...`，说明还没启动完成，再等一会儿。

### 3.2 配置镜像加速（**中国大陆网络下强烈建议**）

`apache/rocketmq`、`mysql`、`redis` 三个镜像都在 Docker Hub，直连 `registry-1.docker.io` 在国内经常超时。

**方式 A（推荐）：给 Docker Engine 加 registry-mirrors**

打开 Docker Desktop → 右上角齿轮 **Settings** → **Docker Engine**，
在 JSON 里加上 `registry-mirrors` 字段（注意保持 JSON 合法，逗号别写错）：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live",
    "https://hub.rat.dev"
  ]
}
```

点击 **Apply & Restart**，等 Docker 重启完成。

**方式 B：先显式从镜像站拉取，再打回原 tag**

如果方式 A 配置后仍然拉不动，可以手动拉：

```bash
docker pull docker.m.daocloud.io/apache/rocketmq:5.1.4
docker tag  docker.m.daocloud.io/apache/rocketmq:5.1.4 apache/rocketmq:5.1.4

docker pull docker.m.daocloud.io/library/mysql:8.0
docker tag  docker.m.daocloud.io/library/mysql:8.0 mysql:8.0

docker pull docker.m.daocloud.io/library/redis:7.0
docker tag  docker.m.daocloud.io/library/redis:7.0 redis:7.0
```

打完 tag 后 `docker-compose.yml` 会直接使用本地已有镜像，不再走网络。

> 💡 镜像站地址会变化，如果上面几个都不可用，可以换成你手边能用的加速地址，
> 只要能 `docker pull` 下来并打上 `apache/rocketmq:5.1.4` 这个 tag 即可。

### 3.3 确认命令形式（`docker compose` 还是 `docker-compose`）

新版 Docker Desktop 自带 `compose` 插件，命令是 **`docker compose`（中间是空格）**；
老一些的安装包则是独立的 **`docker-compose`（中间是连字符）**。

```bash
docker compose version   # 有输出 → 用 `docker compose`
docker-compose version   # 有输出 → 用 `docker-compose`
```

**下文统一写作 `docker-compose`，如果你的是新版插件形式，把命令里的连字符换成空格即可，两者完全等价。**

---

## 4. 启动依赖容器

在项目根目录（`docker-compose.yml` 所在目录）执行：

```bash
docker-compose up -d --wait
```

> ⚠️ **`--wait` 不要省略。**
>
> | 命令 | 行为 |
> |---|---|
> | `docker-compose up -d` | **立刻返回**，此时 MySQL 往往还在初始化 → 应用紧接着启动会**连不上数据库、启动失败** |
> | `docker-compose up -d --wait` | **阻塞到所有服务真正就绪（`healthy`）才返回**（约 5~10 秒）→ 返回后可直接启动应用 ✅ |

首次执行会拉取镜像 + 初始化，通常需要 **1~3 分钟**（取决于网络）；之后再启动只需几秒。

查看状态（`--wait` 返回后通常已经全绿，此步可选）：

```bash
docker-compose ps
```

期望结果：四个容器都是 `Up`，其中 `cityhub-mysql` 和 `cityhub-redis` 会显示 `(healthy)`。

```
NAME                  STATUS
cityhub-mysql         Up (healthy)
cityhub-redis         Up (healthy)
cityhub-rmq-broker    Up
cityhub-rmq-namesrv   Up
```

> ⏳ **`cityhub-mysql` 第一次启动可能要 30~60 秒才变 `healthy`**（要初始化数据目录）。
> 用了 `--wait` 就不必自己盯着——命令返回时它已经就绪了。

**确认 RocketMQ Broker 启动成功**：

```bash
docker logs cityhub-rmq-broker 2>&1 | findstr "boot success"
```

出现 `The broker[broker-a, 127.0.0.1:10911] boot success` 即表示 Broker 就绪（通常需要 20~30 秒）。

---

## 5. 初始化数据库

`docker-compose.yml` 里的 `MYSQL_DATABASE: dingping` 只会**建一个空库**，
表结构和数据需要你自己导入一次（数据卷是持久的，**导一次就够，以后不用再导**）。

**Git Bash / macOS / Linux：**

```bash
docker exec -i cityhub-mysql mysql -uroot -p123456 dingping < src/main/resources/db/hmdp.sql
```

**PowerShell：**

```powershell
Get-Content .\src\main\resources\db\hmdp.sql -Raw | docker exec -i cityhub-mysql mysql -uroot -p123456 dingping
```

**CMD：**

```cmd
docker exec -i cityhub-mysql mysql -uroot -p123456 dingping < src\main\resources\db\hmdp.sql
```

**验证导入结果**（应该能看到 12 张表）：

```bash
docker exec cityhub-mysql mysql -uroot -p123456 -e "use dingping; show tables;"
```

```
docker exec cityhub-mysql mysql -uroot -p123456 -N -e "select count(*) from tb_user;" dingping
```

`tb_user` 有数据即导入成功。

> 🔌 **想用图形化工具连容器里的 MySQL？**
> 新建连接：主机 `127.0.0.1`，端口 **`13306`**，用户 `root`，密码 `123456`。
> 你本机的 3306 不受影响，两套 MySQL 可以同时开着。

---

## 6. 启动应用

应用**跑在宿主机上**（不在容器里），这样方便打断点、看日志、改代码热重启。

**方式一：命令行（PowerShell / CMD）**

```powershell
cd <项目根目录>          # 例如：cd D:\projects\CityHub
mvn spring-boot:run
```

**方式二：IntelliJ IDEA**

打开项目 → 等 Maven 依赖导入完成 → 运行启动类 `com.hmdp.HmDianPingApplication`。

**启动成功的标志**（日志末尾出现）：

```
Tomcat started on port(s): 8081 (http)
Started HmDianPingApplication in x.xxx seconds
```

并且**没有** Redis / MySQL / RocketMQ 的连接异常。

> ⚠️ **如果日志里的端口不是 8081**，说明有别的程序抢占了 8081 或环境变量干扰，
> 可以显式指定：`mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Dserver.port=8081"`

> ⚠️ **依赖始终下不动，怀疑本机 `~/.m2/settings.xml` 里配了内网私服、或把中央仓库镜像屏蔽了？**
> 临时用一份干净的配置绕过即可 —— 新建 `settings-clean.xml`，只写一个阿里云镜像：
>
> ```xml
> <settings>
>   <mirrors>
>     <mirror>
>       <id>aliyunmaven</id>
>       <mirrorOf>central</mirrorOf>
>       <url>https://maven.aliyun.com/repository/public</url>
>     </mirror>
>   </mirrors>
> </settings>
> ```
>
> 然后运行 `mvn -s .\settings-clean.xml spring-boot:run`。

---

## 7. 验证是否跑通（冒烟测试）

下面是一套最小验证链路，全部通过就说明项目已经完整跑通。

### 7.1 免登录接口（不传 token）

```bash
curl http://localhost:8081/shop/1
curl http://localhost:8081/shop-type/list
curl "http://localhost:8081/blog/hot?current=1"
```

返回 `{"success":true,...}` 即为正常。

### 7.2 登录拿 token

本项目登录采用「手机号 + 短信验证码」，验证码会写入 Redis（**不会**在接口响应里回传）。

**第 1 步 · 发送验证码：**

```bash
curl -X POST "http://localhost:8081/user/code?phone=13800138000"
```

**第 2 步 · 从 Redis 里读出验证码：**

```bash
docker exec cityhub-redis redis-cli -a 123456 --no-auth-warning GET login:code:13800138000
```

> 手机号不存在时系统会**自动注册**，所以随便编一个号也能登录。

**第 3 步 · 登录换 token：**

```bash
curl -X POST "http://localhost:8081/user/login" \
  -H "Content-Type: application/json" \
  -d "{\"phone\":\"13800138000\",\"code\":\"上一步读到的验证码\"}"
```

返回：

```json
{ "success": true, "data": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
```

`data` 就是 token。**后续所有需要登录的接口都要带上请求头 `authorization: <token>`**
（注意是小写的 `authorization`）。

### 7.3 秒杀全链路（最能说明项目跑通了）

**第 1 步 · 创建一张秒杀券：**

```bash
curl -X POST http://localhost:8081/voucher/seckill \
  -H "Content-Type: application/json" \
  -d '{
    "shopId": 1,
    "title": "测试秒杀券",
    "subTitle": "示例",
    "rules": "示例规则",
    "payValue": 1000,
    "actualValue": 5000,
    "type": 1,
    "status": 1,
    "stock": 100,
    "beginTime": "2026-01-01T00:00:00",
    "endTime": "2030-12-31T23:59:59"
  }'
```

返回里的 `data` 就是 `voucherId`（下文记作 `{VID}`）。

**第 2 步 · 检查库存已预热到 Redis：**

```bash
docker exec cityhub-redis redis-cli -a 123456 --no-auth-warning GET seckill:stock:{VID}
```

应该返回 `100`。说明建券时已同步把库存预热进 Redis。

**第 3 步 · 下单：**

```bash
curl -X POST http://localhost:8081/voucher-order/seckill/{VID} \
  -H "authorization: <你的 token>"
```

返回 `{"success":true,"data":<订单号>}`。

此时立刻查 Redis 库存，应该已经减 1：

```bash
docker exec cityhub-redis redis-cli -a 123456 --no-auth-warning GET seckill:stock:{VID}
```

**第 4 步 · 再点一次同一个券** —— 应该被拦下：

```json
{ "success": false, "errorMsg": "不能重复下单" }
```

**第 5 步 · 确认订单已异步落库**（MQ 消费完成后写 DB，等待 2~3 秒）：

```bash
docker exec cityhub-mysql mysql -uroot -p123456 -e \
  "select id,user_id,voucher_id,status,create_time from tb_voucher_order where voucher_id={VID};" dingping
```

能看到刚才那笔订单，说明「Redis 下单 → 发 MQ → 消费者落库」整条异步链路是通的。

**第 6 步 · 连点 6 次以上** → 从第 6 次开始返回 `429 Too Many Requests`，说明限流生效（默认每 IP 每秒 5 次）。

---

## 8. 配置详解：为什么这么配

这一节解释 `docker-compose.yml` 和 `broker.conf` 里几处**看起来多余、实际必须**的配置。
如果你在自己环境里遇到问题，多半就出在这几个点上。

| 配置 | 位置 | 为什么必须这么写 |
|---|---|---|
| `13306:3306` | mysql | 本机可能已装 MySQL 占用 3306。映射到 13306 可以两套并存，连容器时连 13306，`application.yaml` 里也对应改成 13306。 |
| **`TZ: Asia/Shanghai` + `--default-time-zone=+08:00`** | mysql | **关键坑**。容器默认时区是 **UTC**，而应用跑在宿主机上用本地时间。`tb_voucher_order.create_time` 这类字段用的是 DB 的 `DEFAULT CURRENT_TIMESTAMP`，于是 DB 时间比应用时间**慢 8 小时**，`OrderTimeoutTask` 拿 `LocalDateTime.now()` 去比就会认为**每笔订单一落库就已超时**，超时关单窗口从 1 分钟缩成 30 秒。必须把容器时区对齐宿主机。 |
| `--requirepass 123456` | redis | 与 `application.yaml` 的 `spring.data.redis.password` **必须一致**，否则应用连不上 Redis。 |
| `--appendonly yes` | redis | 打开 AOF 持久化，Redis 重启后缓存数据不丢。 |
| `JAVA_OPT_EXT: -Xms256m -Xmx256m` | namesrv / broker | RocketMQ 官方镜像默认堆内存是 **8G**，不调小会把宿主机内存吃满。 |
| **`brokerIP1 = 127.0.0.1`** | `broker.conf` | **关键坑**。Broker 默认把容器内网 IP（如 `172.x.x.x`）注册到 NameServer，宿主机上的 Spring Boot 拿到这个地址后无法连接。显式写 `127.0.0.1`，配合 10911 端口映射才能连通。 |
| **`-XX:-UseContainerSupport`** | broker | **关键坑**。官方镜像内置 Temurin **JDK 8u372**，该版本把 `jdk.internal.platform`（cgroup v2 支持）反向移植进了 JDK 8，在 Docker Desktop / WSL2 环境下 `CgroupV2Subsystem` 初始化会抛 `NullPointerException`，进而导致 `StoreUtil` 静态初始化失败。表现是：消息能存进 Broker，但消费者一拉取 Broker 就抛异常，**订单永远无法落库**。关闭容器指标采集即可绕过。 |
| **`user: root`** | broker | **关键坑**。官方镜像以 `uid=3000(rocketmq)` 运行，但镜像内并不存在 `/home/rocketmq/store` 目录；挂载命名卷后 Docker 会把它创建为 `root:root`，非 root 用户写不进去，Broker 直接启动失败。改为 root 运行即可（仅本地开发依赖容器，无外网暴露）。 |
| `autoCreateTopicEnable = true` | `broker.conf` | 开发环境允许自动创建 Topic，省去手动 `mqadmin updateTopic` 的步骤。 |
| `depends_on: rocketmq-namesrv` | broker | 保证 Broker 在 NameServer 之后启动。 |
| `healthcheck` | mysql / redis | `docker-compose ps` 里能看到 `(healthy)`，方便判断依赖是否真的可用，而不是只看到「容器在跑」。 |
| `restart: unless-stopped` | 全部 | 电脑重启或 Docker 重启后容器自动拉起，不用每次手动 `up`。 |

---

## 9. 日常运维命令

```bash
# 查看容器状态
docker-compose ps

# 查看某个容器日志
docker logs -f cityhub-rmq-broker
docker logs --tail 50 cityhub-mysql

# 重启单个服务
docker-compose restart redis

# 进 Redis 命令行
docker exec -it cityhub-redis redis-cli -a 123456

# 进 MySQL 命令行
docker exec -it cityhub-mysql mysql -uroot -p123456 dingping

# 查看 RocketMQ Topic 列表
docker exec cityhub-rmq-broker sh -c \
  "export NAMESRV_ADDR=rocketmq-namesrv:9876; /home/rocketmq/rocketmq-5.1.4/bin/mqadmin topicList"

# 查看消费者堆积情况（Diff 长期不降说明消费卡住）
docker exec cityhub-rmq-broker sh -c \
  "export NAMESRV_ADDR=rocketmq-namesrv:9876; /home/rocketmq/rocketmq-5.1.4/bin/mqadmin consumerProgress -g seckill-order-consumer"
```

---

## 10. 常见问题排查

| 现象 | 原因 | 处理 |
|---|---|---|
| `docker-compose up` 卡在 Pulling / `i/o timeout` | 国内网络连不上 Docker Hub | 见 [3.2 配置镜像加速](#32-配置镜像加速中国大陆网络下强烈建议) |
| `cityhub-mysql` 一直不是 `healthy` | 数据目录初始化中 | 等 30~60 秒；若超过 2 分钟，`docker logs cityhub-mysql` 看报错 |
| `cityhub-rmq-broker` 起不来、日志有 `Permission denied` | 命名卷属主问题 | 确认 `docker-compose.yml` 里 broker 有 `user: root` |
| 容器都 Up，但应用报 **Redis 连接失败** | 密码不一致 | `application.yaml` 的 `spring.data.redis.password` 必须是 `123456` |
| 应用启动报 **RocketMQ 连接超时** | Broker 没起来，或 `brokerIP1` 不对 | `docker logs cityhub-rmq-broker \| findstr "boot success"`；确认 `broker.conf` 里是 `brokerIP1 = 127.0.0.1` |
| 应用报 `Access denied for user 'root'` | 数据库连错端口 | 确认 `application.yaml` 的 URL 里是 **13306** 而不是 3306 |
| **应用启动报 `Communications link failure` / 连不上 MySQL** | 容器还没就绪就启动了应用（用了 `up -d` 而不是 `up -d --wait`） | 改用 `docker-compose up -d --wait`，等命令返回后再启动应用 |
| **秒杀返回成功，但 `tb_voucher_order` 一直没数据** | 消费者拉取消息时 Broker 抛异常（本机为 cgroup v2 兼容性 Bug） | 确认 broker 的 `JAVA_OPT_EXT` 里有 `-XX:-UseContainerSupport`，然后 `docker-compose up -d rocketmq-broker` 重建 |
| 秒杀返回 `-1` / 库存不足 | Redis 里没有库存预热 | 重新创建一张券，建券时会自动预热；或检查 Redis 是否连得上 |
| 秒杀返回 401 | token 过期 | token 有效期 30 分钟，重新登录即可 |
| 请求返回 429 | 触发限流（默认每 IP 每秒 5 次） | 正常现象，等 1 秒再请求 |
| Redis 里有脏数据 | 之前调试残留 | `docker exec cityhub-redis redis-cli -a 123456 FLUSHALL`（**确认没有别的项目在用这个 Redis**） |
| **订单刚下单就被关单、库存立刻回补** | MySQL 容器时区不是本地时区，DB 时间比应用时间慢 8 小时，订单被判超时 | 确认 mysql 服务有 `TZ: Asia/Shanghai` 和 `--default-time-zone=+08:00`，然后 `docker-compose up -d mysql` 重建 |
| `docker-compose` 提示 `Found orphan containers` | 有别的项目的容器用了同一个 compose 项目名 | 无关紧要；确要清理可加 `--remove-orphans` |

---

## 11. 停止与彻底清理

**停止（保留所有数据，下次直接 `up` 即可）：**

```bash
docker-compose stop
```

**删除容器（数据卷保留，下次 `up` 后数据还在）：**

```bash
docker-compose down
```

**连数据一起清空（⚠️ 数据库、Redis 缓存、MQ 消息全部删除，需重新导入 SQL）：**

```bash
docker-compose down -v
```

---

## 附：从零开始的完整命令清单

```bash
# 0. 前提：Docker Desktop 已启动，已配置镜像加速

# 1. 启动全部依赖并等待就绪（首次 1~3 分钟；--wait 会阻塞到 healthy 才返回）
docker-compose up -d --wait

# 2. 确认状态（可选；--wait 返回时已全部就绪）
docker-compose ps

# 3. 确认 RocketMQ Broker 就绪
docker logs cityhub-rmq-broker 2>&1 | findstr "boot success"

# 4. 导入表结构与初始数据（只需一次）
docker exec -i cityhub-mysql mysql -uroot -p123456 dingping < src/main/resources/db/hmdp.sql

# 5. 启动应用
mvn spring-boot:run

# 6. 验证
curl http://localhost:8081/shop/1
```
