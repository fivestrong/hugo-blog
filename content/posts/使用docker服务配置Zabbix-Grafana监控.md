---
title: 使用docker服务配置Zabbix+Grafana监控
date: 2017-10-30 12:20:08
draft: true
---
纯命令行启动
```
1. Start empty MySQL server instance

docker run --name zabbix-mysql -t \
      --restart=always \
      -e MYSQL_DATABASE="zabbix" \
      -e MYSQL_USER="zabbix" \
      -e MYSQL_PASSWORD="zabbix_pwd" \
      -e MYSQL_ROOT_PASSWORD="root_pwd" \
      -d mysql:5.7 \
      --character-set-server=utf8 --collation-server=utf8_bin

2. Start Zabbix server instance and link the instance with created MySQL server instance

docker run --name zabbix-server -t \
      --restart=always \
      -e DB_SERVER_HOST="zabbix-mysql" \
      -e MYSQL_DATABASE="zabbix" \
      -e MYSQL_USER="zabbix" \
      -e MYSQL_PASSWORD="zabbix_pwd" \
      -e MYSQL_ROOT_PASSWORD="root_pwd" \
      --link zabbix-mysql:zabbix-mysql \
      -p 10051:10051 \
      -d zabbix/zabbix-server-mysql

3. Start Zabbix web interface and link the instance with created MySQL server and Zabbix server instances

docker run --name zabbix-web -t \
      -e DB_SERVER_HOST="zabbix-mysql" \
      -e MYSQL_DATABASE="zabbix" \
      -e MYSQL_USER="zabbix" \
      -e MYSQL_PASSWORD="zabbix_pwd" \
      -e MYSQL_ROOT_PASSWORD="root_pwd" \
      -e PHP_TZ='Asia/Hong_Kong' \
      -e ZBX_SERVER_NAME='xxx monit Server' \
      -v /etc/localtime:/etc/localtime:ro \
      -v /root/src/zabbix-grafana/zabbix/graphfont.ttf:/usr/share/zabbix/fonts/graphfont.ttf \
      --link zabbix-mysql:zabbix-mysql \
      --link zabbix-server:zabbix-server \
      -p 80:80 \
      -d zabbix/zabbix-web-nginx-mysql
```
docker-compose file
https://github.com/fivestrong/zabbix-grafana-docker