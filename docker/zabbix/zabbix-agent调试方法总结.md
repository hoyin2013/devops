# zabbix-agent调试方法总结

## 1.agent端调试

- 如果为手工安装zabbix，启动和停止监控分别使用`pkill zabbix`,`/path/to/zabbix_agent`
- 修改zabbix_agent.conf
- 如果调用的是官方模板，则先去服务端观察模板项中item值的形式，例如mysql官方监控模板中的item形势为'mysql[]',则agent端传参变量名应当取名为mysql[*]
`UserParameter=mysql[*],/usr/local/mysql/bin/mysqladmin -uzabbix -pzabbix_32 -S /home/mysql/data/mysql/mysql3306/mysql.sock extended-status 2>/dev/null|grep -w "$1"|cut -d"|" -f3`

## 2.格式说明

mysql[*],xxx $1 xxx
    - 参数名：mysql
    - 参数：$1，说明共需要传1个参数
    - 调用方法举例：mysql[Uptime]
    - 具体参数值应该填写什么内容需要依据参数','号后面的脚本运行时需要的参数而定
    - UserParameter 可以设置多个

    - 重要：
        配置完成后将监控服务器地址配置成本机地址方便测试，如：
        Server=127.0.0.1
        ServerActive=127.0.0.1

## 3.重启zabbix_agent后进行测试,举例

zabbix_get -s 127.0.0.1 -k mysql[Uptime]
zabbix_get -s 127.0.0.1 -k "system.cpu.load[all,avg15]"

- zabbix_get在zabbix安装目录下的bin文件夹中
- 确保能获取到所需要的值

## 4.调试完成后，将server改回真实监控服务器地址并重启agent

## 5.server端调试

在zabbix管理界面中检查"配置"->"主机"->"监控项"中的"键值"是否和调试客户端中配置的参数名格式一致
检查zabbix_agent状态是否正常
