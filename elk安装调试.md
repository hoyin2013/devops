
# docker版elk安装调试

## docker安装elk

1. 修改运行内存

```bash
# 运行elasticsearch需要vm.max_map_count至少需要262144内存
# 切换到root用户修改配置sysctl.conf
vi /etc/sysctl.d/elasticsearch.conf
# 在尾行添加以下内容
vm.max_map_count=262144
# 执行命令
sysctl -p

# 1. 运行docker 参考：[手册](https://elk-docker.readthedocs.io/#usage)
docker run -tid -p 5601:5601 -p 9200:9200 -p 5044:5044 --name elk sebp/elk

```

## logstash调试

### 本地测试

```bash

# 配置文件只定义filter中的match
cat /etc/logstash/conf.d/test.conf
input{
   stdin{
   }
}

filter {
    grok {
        match => {
             "message" => "%{TIMESTAMP_ISO8601:timestamp} %{SPACE} %{DATA:user} %{SPACE} %{IPV4:ip} %{SPACE} %{DATA:database} %{SPACE} %{DATA:ms} %{SPACE} %{INT:com} %{SPACE} %{GREEDYDATA:sql}"
        }
    }
}

output{
   stdout{
    }
}

# 启动测试
/opt/logstash/bin/logstash  --path.settings /etc/logstash  -f /etc/logstash/conf.d/test.conf


```

### 配置logstash抓取tcp端口(不推荐这种，因为还要手工制作模板等内容，调试麻烦）

#### 重新启动容器，修改配置

docker cp elk:/etc/logstash/config.d ./

```bash
# 修改配置参数
input{
  tcp{
   port => 1514
   mode => "server"
   type => "tcplog"
    }
}

filter {
    grok {
        match => {
             "message" => "%{TIMESTAMP_ISO8601:timestamp} %{SPACE} %{DATA:user} %{SPACE} %{IPV4:ip} %{SPACE} %{DATA:database} %{SPACE} %{DATA:ms} %{SPACE} %{INT:com} %{SPACE} %{GREEDYDATA:sql}"
        }
    }
}

output {
  elasticsearch {
    hosts => ["localhost"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
  }
}

# 重新创建容器
docker rm -f elk
cd /home/docker-elk/logstash
docker run -tid -p 5601:5601 -p 9200:9200 -p 1514:1514 -v $PWD/conf.d:/etc/logstash/conf.d --name elk sebp/elk
```

### 模拟socket数据发送

- 方式一
`echo "2020-04-21 18:32:12      paas    192.168.16.15   ds_seata_store           0ms             0      SET NAMES utf8mb4" | nc 127.0.0.1 1514`

- 方式二
`telnet 127.0.0.1 1514`

### 在线测试filter中的match正则表达式

- 方式一：[http://grokdebug.herokuapp.com/](http://grokdebug.herokuapp.com/)
- 方式二：kibana的开发工具中有Grok Debugger

### 查看grok正则定义函数

- grok模式关键字含义 [grok模式](https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns)

### 示例数据

```log
2020-04-21 18:32:12      paas    192.168.16.15   ds_seata_store           3ms            18      /* mysql-connector-java-5.1.30 ( Revision: alexander.soklakov@oracle.com-20140310090514-8xt1yoht5ksg2e7c ) */SHOW VARIABLES WHERE Variable_name ='language' OR Variable_name = 'net_write_timeout' OR Variable_name = 'interactive_timeout' OR Variable_name = 'wait_timeout' OR Variable_name = 'character_set_client' OR Variable_name = 'character_set_connection' OR Variable_name = 'character_set' OR Variable_name = 'character_set_server' OR Variable_name = 'tx_isolation' OR Variable_name = 'transaction_isolation' OR Variable_name = 'character_set_results' OR Variable_name = 'timezone' OR Variable_name = 'time_zone' OR Variable_name = 'system_time_zone' OR Variable_name = 'lower_case_table_names' OR Variable_name = 'max_allowed_packet' OR Variable_name = 'net_buffer_length' OR Variable_name = 'sql_mode' OR Variable_name = 'query_cache_type' OR Variable_name = 'query_cache_size' OR Variable_name = 'init_connect'
2020-04-21 18:32:12      paas    192.168.16.15   ds_seata_store           0ms             0      SET NAMES utf8mb4
2020-04-21 18:32:12      paas    192.168.16.15   ds_seata_store           0ms             1      /* mysql-connector-java-5.1.30 ( Revision: alexander.soklakov@oracle.com-20140310090514-8xt1yoht5ksg2e7c ) */SELECT @@session.auto_increment_increment

```

### 使用packetbeat插件抓包

```bash
# rpm包安装
sudo yum install libpcap
curl -L -O https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-7.6.2-x86_64.rpm
sudo rpm -vi packetbeat-7.6.2-x86_64.rpm

# 二进制安装
curl -L -O https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-7.6.2-linux-x86_64.tar.gz
tar xzvf packetbeat-7.6.2-linux-x86_64.tar.gz

# packagebeat配置
# console输出
output.console:
  pretty: true
setup.template.overwrite: true

# 手工加载模板
packetbeat setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'

# 删除旧的
curl -XDELETE 'http://localhost:9200/packetbeat-*'

# 安装dashboard
packetbeat  

```



