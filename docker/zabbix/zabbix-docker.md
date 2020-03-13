# docker方式安装zabbix-server

## 简介

- zabbix-server依赖zabbix源码包，mysql，php环境，nginx四大块组件
- 源码下载地址`https://github.com/zabbix/zabbix.git`,安装时需要经过configure，make等步骤
- 官方提供的docker镜像中有基于好几种操作系统的，可自行选择
- zabbix-server-mysql中提供的zabbix数据库脚本，自身没有安装mysql程序，需要将脚本恢复到数据库服务器上，编码是utf8编码
- zabbix-web-nginx-mysql镜像里面除了没有数据库外，nginx，php和zabbix的web文件准备妥当
- zabbix源码中没有没有直接可以导入到数据库中的脚本文件
- `git clone https://git.zabbix.com/scm/zbx/zabbix.git`下载源码有时候贼慢，也有可能和我家里的网络有关

## 本人安装步骤

- 安装数据库，将数据库安装到mysql容器数据库中
    docker run -tid --name myzabbix \
    --link hoyin_mysql_1:zabbix-server \   # 连接我的mysql容器数据库，并将主机映射成zabbix-server
    -e DB_SERVER_HOST="zabbix-server" \   # --link定义的名称是啥，这里就是啥
    -e MYSQL_DATABASE="zabbix" \       # 默认就是zabbix
    -e MYSQL_USER="zabbix" \             # 数据库用户
    -e MYSQL_PASSWORD="Zabbix@2020" \   # 数据库的密码
    -e ZBX_SERVER_HOST="myzabbix" \   # 我这里和--name保持一致，不加是什么效果，没去测试
    -e PHP_TZ="Asia/Shanghai"  zabbix/zabbix-server-mysql

- 销毁myzabbix容器
    docker rm -f myzabbix

- 启动前端容器,连接数据库容器
    docker run -tid --name myzabbix -p 80:80  \
    --link hoyin_mysql_1:zabbix-server \
    -e DB_SERVER_HOST="zabbix-server" \
    -e MYSQL_DATABASE="zabbix" \
    -e MYSQL_USER="zabbix" \
    -e MYSQL_PASSWORD="Zabbix@2020" \
    -e ZBX_SERVER_HOST="myzabbix" \
    -e PHP_TZ="Asia/Shanghai"  zabbix/zabbix-web-nginx-mysql

## 登录web端

- <http://127.0.0.1/>
- 默认用户名和密码为：Admin/zabbix

## 问题

- 主界面找不到zabbix agent，应该是因为服务器上没有装agent
- 显示Zabbix server is runing No myzabbix:10051 这个估计和代理没装有关系

## docker-compose 一站式解决

- 修改自官方docker-compose，把java网关，snmp和一些多余的东西去掉了
- `zabbix-server` 服务
- `zabbix-web-nginx-mysql` web前端
- `zabbix-agent`  zabbix代理
- `mysql-server`  数据库

### 启动：`docker-compose up -d`

- zbx_env 里面用来存放数据和配置文件
