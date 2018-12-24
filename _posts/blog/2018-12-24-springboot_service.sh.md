#!/bin/sh
#AppNameDefault 要求启动脚本和app启动jar要放同一目录

AppNameDefault=`ls *.jar |head -1`
if [ "$AppName" = "" ];
then
  echo "------------------------------------"
  echo -e "\033[0;31m 获取默认$AppNameDefault \033[0m"
  AppName=$AppNameDefault
fi

if [ "$AppName" = "" ];
then
    echo "------------------------------------"
    echo -e "\033[0;31m 未找到AppName,请确认... \033[0m"
    exit 1
fi


touch $AppName.service

echo -e "[Unit]
Description=$AppName.service
After=syslog.target
[Service]
Type=forking
ExecStart=/usr/bin/bash -c '/usr/local/app/spring_boot.sh start /usr/local/app/middlelayer-gateway-0.0.1-SNAPSHOT.jar'
ExecReload=/usr/bin/bash -c '/usr/local/app/spring_boot.sh restart'
ExecStop=/usr/bin/bash -c '/usr/local/app/spring_boot.sh stop'

[Install]
WantedBy=multi-user.target" > $AppName.service


sudo rm -rf /etc/systemd/system/$AppName.service

sudo ln $AppName.service /etc/systemd/system/$AppName.service

sudo chmod  754 /etc/systemd/system/$AppName.service

sudo  systemctl daemon-reload
