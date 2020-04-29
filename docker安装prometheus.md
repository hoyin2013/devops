# docker 安装promethes

- 实验IP:10.128.128.210

## 下载相关镜像

```bash
docker pull prom/node-exporter
docker pull prom/prometheus
docker pull grafana/grafana
```

## 启动node-exporter(采集本机服务器信息)

```bash
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  --net="host" \
  prom/node-exporter

netstat -tanlp | grep 9100
# 访问 http://10.128.128.210:9100/metrics 可以看到采集到的相关信息


```

## 启动prometheus

```bash
# 准备目录
mkdir /opt/prometheus && cd /opt/prometheus
cat <<EOF >> /opt/prometheus/prometheus.yml
global:
  scrape_interval:     60s
  evaluation_interval: 60s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: linux-15.195
    static_configs:
      - targets: ['10.128.128.210:9100']
EOF

# 启动
docker run  -d \
  --name prometheus \
  -p 9090:9090 \
  -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
  prom/prometheus

# 查看启动情况
netstat -tanlp|grep 9090
# web访问url
http://10.128.128.210:9090/graph
# 查看监控主机
http://10.128.128.210:9090/targets

```

## grafana启动

```bash
# 目录准备
mkdir /opt/grafana-storage && chmod -R 777 /opt/grafana-storage

# 启动容器
docker run -d -p 3000:3000 --name=grafana -v /opt/grafana-storage:/var/lib/grafana \
grafana/grafana

# 查看状态
netstat -tanlp |grep 3000

# 访问url http://10.128.128.210:3000/ 默认用户名和密码都为： admin
# 1.设置密码
# 2.add data source -> Prometheus -> Name(Prometheus)|URL(http://10.128.128.210:9090) | Dashboards -> import -> Save & Test

```

## 测试mysqld-exporter

```bash
# docker 安装mysql
# 准备工作目录
mkdir -pv /opt/mysql/{conf,logs,data} && cd /opt/mysql

# 运行容器
cd /opt/mysql && docker run -p 3306:3306 --name mysql-1 \
-v $PWD/conf:/etc/mysql/conf.d \
-v $PWD/logs:/logs \
-v $PWD/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql

# 赋权初始化
docker exec -it 6f mysql -uroot -p123456
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password' PASSWORD EXPIRE NEVER;
flush privileges;

# 创建监控用户
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporter' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

# docker安装mysqld-exporter
docker pull prom/mysqld-exporter

date -R
mv /etc/timezone /etc/timezone-`date +%Y%m%d-%H%M%S`
echo 'Asia/Shanghai' > /etc/timezone
mv /etc/localtime /etc/localtime-`date +%Y%m%d-%H%M%S`
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

docker pull prom/mysqld-exporter
docker rm -f mysqld-exporter
docker run -d \
--name mysqld-exporter \
-v /etc/timezone:/etc/timezone \
-v /etc/localtime:/etc/localtime \
-p 9104:9104 \
-e DATA_SOURCE_NAME="exporter:exporter@(10.128.128.210:3306)/" \
--restart=unless-stopped \
prom/mysqld-exporter
docker logs -f --tail 10 mysqld-exporter

```
promtool check config prometheus.yml

