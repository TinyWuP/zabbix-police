# zabbix police 报警收敛聚合脚本
相关配置说明见简书"[zabbix告警收敛](https://opsolo.com/zabbix/zabbix-alarm-convergence-compression/)"

由于报警短信、邮件太多导致运维人员精神高度紧张、时间长了容易忽略重要告警，引起不必要的麻烦。为了解决这个问题针对Zabbix告警做了二次开发，目前该代码已应用于我们公司环境中三年以上。

下面为大家分享下整体的流程以及代码。

一、架构图
①产生的所有告警均由zabbix的actions调用脚本推入缓存redis当中；
②脚本将每分钟(crontab)去redis中拉取数据，根据定义好的一系列规则进行分析、合并；
③根据预先定义好的规则将报警通过定义好的方式发送给相关人员；
![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
zabbix收敛-流程图

二、设置Zabbix
1. 配置Media types
仅传递subject
我这里定义了3个Mediatype 分别用于发送邮件、短信、企业微信(具体可自行调整) （3个除了Name不一样之外其他配置(Script name/Script parameters)保持一致）
脚本 “zabbix-police/police.py” 主要功能是将Subject(Eventid)写入Redis，后面会写到
![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-Mediatype%E9%85%8D%E7%BD%AE.png)
Media type 配置

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E5%A4%9A%E4%B8%AAMediatypes.png)
zabbix收敛-多个Mediatypes
2. 配置Actions
我这里以每个Trigger severity一个Actions举例。（可以根据不同的HostGroup或者其他条件自行配置多个actions）
Default subject 之所以用 “{EVENT.ID}_1、{EVENT.ID}_0“为的是保持唯一性，1代表故障、0代表恢复
Default sbject
Code
{EVENT.ID}_1
Default message
Code
triggervalue|{TRIGGER.VALUE}
hostname|{HOSTNAME1}
ipaddress|{IPADDRESS}
hostgroup|{TRIGGER.HOSTGROUP.NAME}
triggerstatus|{TRIGGER.STATUS}
triggerseverity|{TRIGGER.SEVERITY}
triggername|{TRIGGER.NAME}
triggerkey|{TRIGGER.KEY1}
triggeritems|{ITEM.NAME}
itemvalue|{ITEM.VALUE}
eventid|{EVENT.ID}
action|{ACTION.NAME}
eventage|{EVENT.AGE}
eventtime|{EVENT.DATE} {EVENT.TIME}
Recovery subject
Code
{EVENT.ID}_0
Recovery message
Code
triggervalue|{TRIGGER.VALUE}
hostname|{HOSTNAME1}
ipaddress|{IPADDRESS}
hostgroup|{TRIGGER.HOSTGROUP.NAME}
triggerstatus|{TRIGGER.STATUS}
triggerseverity|{TRIGGER.SEVERITY}
triggername|{TRIGGER.NAME}
triggerkey|{TRIGGER.KEY1}
triggeritems|{ITEM.NAME}
itemvalue|{ITEM.VALUE}
eventid|{EVENT.ID}
action|{ACTION.NAME}
eventage|{EVENT.AGE}
eventtime|{EVENT.DATE} {EVENT.TIME}
![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E5%A4%9A%E4%B8%AAActions.png)
zabbix收敛-多个Actions

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-Actions-%E6%9D%A1%E4%BB%B6%E9%85%8D%E7%BD%AE.png)
zabbix收敛-Actions-条件配置

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-Actions-Operations%E9%85%8D%E7%BD%AE.png)
zabbix收敛-Actions-Operations配置

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-Actions-Recovery%E9%85%8D%E7%BD%AE.png)
zabbix收敛-Actions-Recovery配置

三、配置 Zabbix 服务器
1. 安装环境
Bash
#下载代码
/etc/zabbix/alertscripts
git clone https://github.com/sungaomeng/zabbix-police.git
#安装依赖
yum install gcc python-devel
pip install -r zabbix-police/requirements.txt
2. 脚本
Bash
#文件分布
[root@zabbix-server01 alertscripts]$ tree zabbix-police 
zabbix-police
├── police.py    #Action调用此函数, 用于将EventID写入Redis
├── allpolice.py #综合函数, 总入口, 用于整合其他脚本, 定时被Crontab调用
├── dbread.py    #数据库查询函数, 用于查询Redis、Mysql, 获取EventID、获取告警具体信息、Mediatype脚本对应关系、查询告警接收人等信息
├── police.conf  #定义配置文件, 包括mysql、redis、wechat、email、sms、logfile等配置
├── modconf.py   #加载配置函数, 用于加载配置文件
├── operation.py #操作函数, 用于1. 接收dbread.py返回的告警、恢复信息, 进行合并、压缩处理, 并返回处理结果 2. 定义各告警发送调用函数
├── send_wechat.py #告警发送-微信函数
├── send_sms.py    #告警发送-短信函数
├── send_email.py  #告警发送-邮件函数
├── requirements.txt #依赖
└── README.md
3. Crontab
Bash
[root@zabbix-server01 zabbix-police]$ crontab -l 
* * * * * /usr/bin/python /etc/zabbix/alertscripts/zabbix-police/allpolice.py
四、告警效果
![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E9%82%AE%E4%BB%B6%E5%91%8A%E8%AD%A6.png)
zabbix收敛-邮件告警

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E5%91%8A%E8%AD%A6.png)
zabbix收敛-企业微信告警

![logo](https://github.com/TinyWuP/zabbix-police/blob/master/img/zabbix%E6%94%B6%E6%95%9B-%E7%9F%AD%E4%BF%A1%E5%91%8A%E8%AD%A6.png)
zabbix收敛-短信告警
