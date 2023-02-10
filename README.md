# Logging Linux Shell Commands
[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Fpassword123456%2Flogging_linux_shell_command&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

The following is a configuration for maintaining a command history utilizing a simplistic method that I used, which can be utilized on any Linux system. 
It is not a particularly innovative or cutting-edge approach. If necessary, feel free to modify it to meet your needs. 

If you have a SIEM in place, the parsed command history can easily be received via SYSLOG.
No any agents are required. Just set it up and you can distribute the configuration file by copying it.

# Logging field information

ID | Field | Channel
----- | ----- | ----- 
1 | time_now | current time
2 | tme_hms | current time - hours/minutes/seconds
3 | tty_number | Type and number of the current terminal session
4 | user | current login username
5 | is_root | current user is root or not
6 | shell_status | check user prompt. if root "#", not root "$"
7 | ip | IP address connecting from a remote
8 | current_path |  the current working directory in the shell
9 | command | the command entered in the shell.
10 | ps | A custom field that is left in the same way as the PS status of the shell.

# SET-UP

1. Create a file named "e.g) history_log.sh" under the "/etc/profile.d/" directory and add the following code to it:
```bash
#!/bin/bash

function log_command {
    local time_now=$(date +"%Y-%m-%d %T")
    local time_hms=$(date +"%T")
    local tty_number=$(tty | cut -d '/' -f 4)
    local user=$(whoami)
    local is_root=$([[ "$(id -u)" == "0" ]] && echo "y" || echo "n")
    local shell_status=$([[ "$(id -u)" == "0" ]] && echo "#" || echo "$")
    local ip=$(tty | awk -F "/" '{print $3"/"$4}' | xargs -I % bash -c 'w | grep -i %' | awk '{print $3}')
    local current_path=$(pwd)
    local command=$@
    local ps="[$time_hms][$user@$(hostname)]$current_path~$shell_status $command"

    logger -p local6.info -t cmd_h1st "datetime='$time_now',tty='$tty_number',user='$user',root='$is_root',ip='$ip',pwd='$current_path',cmd='$command',ps='$ps'"
}

PROMPT_COMMAND='log_command "$(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//")"'

```

2. Open the "/etc/rsyslog.conf" file and add the configuration to save in local6.info format to the "/var/log/command.log" file.
```
# vim /etc/rsyslog.conf
.....
local6.info        /var/log/command.log
```

3. Restart the rsyslog service to apply the changes.
```
# systemctl restart rsyslog.service
```

4. If you log out of the shell and log back in, you will see that the /var/log/command.log file has been created.

# result
- example of /var/log/command.log

```bash
# cat /var/log/command.log
..
...
.....

.... cmd_h1st[720030]: datetime='2023-02-11 00:16:27',tty='2',user='opc',root='n',ip='1*5.2**.*9.1**',pwd='/home/opc',cmd='sudo -i',ps='[00:16:27][opc@buddy]/home/opc~$ sudo -i'
.... cmd_h1st[720061]: datetime='2023-02-11 00:16:29',tty='2',user='opc',root='n',ip='1*5.2**.*9.1**',pwd='/home/opc',cmd='env',ps='[00:16:29][opc@buddy]/home/opc~$ env'
.... cmd_h1st[720093]: datetime='2023-02-11 00:17:12',tty='2',user='opc',root='n',ip='1*5.2**.*9.1**',pwd='/home/opc',cmd='ls',ps='[00:17:12][opc@buddy]/home/opc~$ ls'
.... cmd_h1st[720659]: datetime='2023-02-11 07:16:51',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='exit',ps='[07:16:51][root@buddy]/root~# exit'
.... cmd_h1st[720689]: datetime='2023-02-11 07:16:51',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='exit',ps='[07:16:51][root@buddy]/root~# exit'
.... cmd_h1st[720719]: datetime='2023-02-11 07:16:52',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='exit',ps='[07:16:52][root@buddy]/root~# exit'
.... cmd_h1st[720750]: datetime='2023-02-11 07:16:52',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='ll',ps='[07:16:52][root@buddy]/root~# ll'
.... cmd_h1st[720797]: datetime='2023-02-11 07:18:14',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='vim /etc/rsyslog.conf ',ps='[07:18:14][root@buddy]/root~# vim /etc/rsyslog.conf '
.... cmd_h1st[720827]: datetime='2023-02-11 07:18:16',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='history',ps='[07:18:16][root@buddy]/root~# history'
.... cmd_h1st[720889]: datetime='2023-02-11 07:18:47',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='systemctl restart rsyslog.service ',ps='[07:18:47][root@buddy]/root~# systemctl restart rsyslog.service '
.... cmd_h1st[720919]: datetime='2023-02-11 07:18:47',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='systemctl restart rsyslog.service ',ps='[07:18:47][root@buddy]/root~# systemctl restart rsyslog.service '
.... cmd_h1st[720949]: datetime='2023-02-11 07:18:47',tty='2',user='root',root='y',ip='1*5.2**.*9.1**',pwd='/root',cmd='systemctl restart rsyslog.service ',ps='[07:18:47][root@buddy]/root~# systemctl restart rsyslog.service '
.... cmd_h1st[720983]: datetime='2023-02-11 07:21:38',tty='2',user='nara',root='n',ip='1*5.2**.*9.1**',pwd='/home/nara',cmd='sudo -i',ps='[07:21:38][nara@buddy]/home/nara~$ sudo -i'
.... cmd_h1st[721013]: datetime='2023-02-11 07:21:39',tty='2',user='nara',root='n',ip='1*5.2**.*9.1**',pwd='/home/nara',cmd='sudo -i',ps='[07:21:39][nara@buddy]/home/nara~$ sudo -i'
.... cmd_h1st[721044]: datetime='2023-02-11 07:21:48',tty='2',user='nara',root='n',ip='1*5.2**.*9.1**',pwd='/home/nara',cmd='ls -al /etc',ps='[07:21:48][nara@buddy]/home/nara~$ ls -al /etc'
.... cmd_h1st[721077]: datetime='2023-02-11 07:21:55',tty='2',user='nara',root='n',ip='1*5.2**.*9.1**',pwd='/home/nara',cmd='df -h /var/log',ps='[07:21:55][nara@buddy]/home/nara~$ df -h /var/log'

```

# Additional
- If you are using a SIEM, you can receive the logs through syslog and parse them for analysis.
- If necessary make additional fields and apply.
- In case multiple users log in with the same username, we can individually identify who they are.
- By default, when a user change level to root, their  IP address is lost, so making it impossible to recored the user's remote connection ip, but it is fine now. IP is recorded in all cases.
