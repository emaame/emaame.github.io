---
layout: post
title: Splatnet2statink.py runs as Daemon
---

# AT YOUR OWN RISK
Nobody never use who don't understand how [splatnet2statink.py](https://github.com/frozenpandaman/splatnet2statink) works.

# What I Want
* `splatnet2statink.py` runs 24/7 in a VPS automatically.
* Alert by e-mail when it dies

`-M` make it as monitor mode, but it is necessary to restart in hand when a server reboot or it dies.

# `Splatnet2statink.py` runs as Daemon with Systemd
Systemd is a popular manager of Daemon in recent Linux dists.
It requires a service configuration files and some script.

## Define Unit as Service
### Edit a service configuration file.

```
sudo vim /etc/systemd/system/splatnet2statink.service
```

Write down such as (change path for your env)

```
[Unit]
Description = splatnet2statink.py
After = syslog.target network.target

[Service]
Type = forking
EnvironmentFile = /etc/default/splatnet2statink
WorkingDirectory = /path/to/script
ExecStart = /path/to/script/start.sh
ExecStop = /path/to/script/stop.sh
User = your_username
Group = your_groupname
KillMode = none
Restart = no

[Install]
WantedBy = multi-user.target
```

### Write an EnvironmentFile
Edit /etc/default/splatnet2statink (this place is defined above).
And make dirs they are defined.

```
RUN_DIR=/path/to/splatnet2statink
LOCK_FILE=/var/lib/splatnet2statink/splatnet2statink.lock

LOG=/var/log/splatnet2statink/statink.log
ERROR_LOG=/var/log/splatnet2statink/splatnet2statink.error_log
PID_FILE=/var/lib/splatnet2statink/run/splatnet2statink/splatnet2statink.pid
```

They are used in Start/Stop scripts as ENV vars.
If you need more ENV vars, you can define more ENV vars.

*NOTE: I use 2 accounts, so I define 2 values for logs and pids.*

### Write Start/Stop scripts
Edit /path/to/script/{start,stop}.sh (this place is also defined above),
And run `chown` and `chmod +x` appropriately.

sudo chown user:group {start,stop}.sh
sudo chmod +x {start,stop}.sh

start.sh runs a background process and records its pid in PID_FILE.
Arrange COMMAND var to set `splatnet2statink.py` parameters you want.
**Keep -M parameter enough LARGE. 50 matches spend about 50 mins at least.** 
This script expect `chmod +x` to `splatnet2statink.py`.

```sh:start.sh
#!/bin/bash
COMMAND="./splatnet2statink.py -M 1800 -r"

cd ${RUN_DIR}
${COMMAND} > ${LOG} 2>${ERROR_LOG} &
echo $! > ${PID_FILE}
```

*NOTE: Since LOG fills by countdown print, I use /dev/null after it works correctly*

stop.sh kills a background process which refers PID_FILE,
and remove PID_FILE and LOCK_FILE.

```sh:stop.sh
#!/bin/bash
[ -f ${PID_FILE} ] && kill -9 $(cat ${PID_FILE})
[ -f ${PID_FILE} ] && rm ${PID_FILE}

[ -f ${LOCK_FILE} ] && rm ${LOCK_FILE}
```

### Check an installation of Unit
systemctl list up all services, and find your unit in the list.
```
sudo systemctl list-unit-files --type=service | grep splatnet2statink
```

You'll get a result such below.

```
splatnet2statink.service                      enabled
```

### Manage the service
Enable it,
now you can get an automatic start of a daemon when a server rebooted.

```
sudo systemctl enable splatnet2statink
```

Start

```
sudo systemctl start splatnet2statink
```

Check Status

```
sudo systemctl status splatnet2statink
```

You'll get such as

```
● splatnet2statink.service - run splatnet2statink
   Loaded: loaded (/etc/systemd/system/splatnet2statink.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2017-12-10 15:51:57 JST; 2h 31min ago
  Process: 764 ExecStop=/path/to/script/stop.sh (code=exited, status=0/SUCCESS)
  Process: 769 ExecStart=/path/to/script/start.sh (code=exited, status=0/SUCCESS)
 Main PID: 3xxxx (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/statink.service
           ├─771 python ./splatnet2statink.py -M 1800 -r
```

You can Stop and Restart

```
sudo systemctl stop splatnet2statink
```

```
sudo systemctl restart splatnet2statink
```

# Make alive monitor of `splatnet2statink.py`
I wanna know when `splatnet2statink.py` dies.
I run another daemon to watch `splatnet2statink.py` alive.

*NOTE: Systemd may have Restart option in a servece configuration file.
I wanna alert*

It check a process name and If it could find out then it sleeps.
If the process dies, it tries to restart.
The restart failed it alerts by email!


*NOTE: Update / Maintenance could be cause of failure.
After a maintenance it restart automatically.*

This script use mail command.
You need to make it works.

```sh:alive_monitor.sh
#!/bin/sh

processName="splatnet2statink.py"
interval=3600

error_reported="reported.lock"

MAILFILE=/var/log/splatnet2statink/splatnet2statink_mail.txt
MAILTO=you@mailserver

while true
do
        isAlive=`ps -ef | grep "$processName" | grep -v grep | grep -v alive_monitor | wc -l`
        if [ $isAlive = 2 ]; then
            echo "Server is running."
            if [ -f ${error_reported} ]; then
                rm $error_reported
            fi
        else
            if [ -f ${error_reported} ]; then
                echo "already mailed failure."
            else
                echo "Server is dead, restarting..."
                systemctl stop splatnet2statink
                systemctl start splatnet2statink
                alive=`systemctl is-active splatnet2statink`
                if [ $alive = 0 ]; then
                        # 再起動成功
                        echo "Server is restarted"
                else
                        echo "Server restart was failed"
                        #メールの本文を生成
                          hostname > $MAILFILE
                          date "+%Y/%m/%d %H:%M:%S" >> $MAILFILE
                          echo "${processName} Down" >> $MAILFILE
                          echo "command"  >> $MAILFILE
                          echo "ps -ef | grep $processName | grep -v grep | wc -l" >> $MAILFILE
                          systemctl status splatnet2statink >> $MAILFILE

                        #メールの件名を生成
                          MAIL_SUB="Process $processName Down"

                        #メールを送信する
                          mail -s $MAIL_SUB $MAILTO < $MAILFILE
                fi
            fi
        fi
        sleep $interval
done
```

/etc/systemd/system/splatnet2statink_alivemonirtor.service

```
[Unit]
Description=Alive monitor for splatnet2statink

[Service]
Type=forking
EnvironmentFile=/etc/default/alive_monitor_splatnet2statink
WorkingDirectory=/path/to/alive_monitor/
ExecStart=/path/to/alive_monitor/start.sh
ExecStop=/path/to/alive_monitor/stop.sh
User=root
Group=root
KillMode=none
Restart = no

[Install]
WantedBy=multi-user.target
```

start.sh / stop.sh and others are the same as splatnet2statink.service.
Enable it and start it.


# References
* [splatnet2statink.py](https://github.com/frozenpandaman/splatnet2statink)
* [splatnet2statink.py でスプラトゥーン2の試合結果を stat.ink へ自動送信しようの巻](https://archive.fo/td52p)
* [Systemdを使ってさくっと自作コマンドをサービス化してみる](https://qiita.com/DQNEO/items/0b5d0bc5d3cf407cb7ff)
* [Systemd入門(4) - serviceタイプUnitの設定ファイル](http://enakai00.hatenablog.com/entry/20130917/1379374797)
* [systemd を利用してプロセスをデーモン化する](http://cameong.hatenablog.com/entry/2016/10/18/121400)
* [簡易プロセス死活監視スクリプト](https://qiita.com/tattsun58/items/67b0f16c86fbe49fe5d0)
* [プロセスの生存を監視するスクリプト](https://ex1.m-yabe.com/archives/1892)
