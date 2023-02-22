# Logging Linux Shell Commands
[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Fpassword123456%2Flogging_linux_shell_command&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

The following is a configuration for maintaining a command history utilizing a simplistic method that I used, which can be utilized on any Linux system. 
It is not a particularly innovative or cutting-edge approach. If necessary, feel free to modify it to meet your needs. 

If you have a SIEM in place, the parsed command history can easily be received via SYSLOG.
No any agents are required. Just set it up and you can distribute the configuration file by copying it.

# Logging field information

no | field | description
----- | ----- | ----- 
1 | ctime_hms | The current time in the format of HH:MM:SS.
2 | login_user | The username of the user who is currently logged in.
3 | sudo_user | The username of the user who used sudo to run the command. If sudo wasn't used, set this variable to "null".
4 | is_root | Whether the current user is the root user. If the current user is the root user, set this variable to "y". Otherwise, set it to "n".
5 | shell_status | The command prompt symbol. If the current user is the root user, set this variable to "#". Otherwise, set it to "$".
6 | remote_ip | he IP address of the remote host that the user logged in from.
7 | pwd | The current working directory.
8 | command | The command that was executed.
9 | cmd_retn_code | The return code of the command that was executed.
10 | cmd_pid | The process ID of the command that was executed.
11 | sudo_chk | Whether the command that was executed used sudo. If the command used sudo, set this variable to "y". Otherwise, set it to "n".
12 | sudo_with | Whether sudo was used to run the command. If the current user is the root user, set this variable to "y". Otherwise, set it to the value of sudo_chk.
13 | ps | The command prompt string.

# Variables in logging

no | field | description
----- | ----- | ----- 
1 | datetime | The date and time when the command was executed.
2 | tty | The terminal device name.
3 | bash_pid | The process ID of the current bash shell.
4 | type | The type of the log entry. "new_login" or "logged_in".
5 | username | The username of the user who executed the command.
6 | sudo_user | The username of the user who used sudo to run the command. If sudo wasn't used, this variable is set to "null".
7 | root | Whether the current user is the root user. If the current user is the root user, this variable is set to "y". Otherwise, it's set to "n".
8 | ip | The IP address of the remote host that the user logged in from.
9 | pwd | The current working directory.
10 | cmd | The command that was executed

# SET-UP

1. Add the `HISTTIMEFORMAT` in the /etc/profile
```bash
HISTSIZE=2000
HISTTIMEFORMAT="[%Y-%m-%d %H:%M:%S]  "
```

2. Create a file named "e.g) history_log.sh" under the "/etc/profile.d/" directory and add the following code to it:
```bash
logger -p local7.notice -t cmd_h1st "datetime='$(date +"%Y-%m-%d %T")',tty='$(tty | cut -d '/' -f 3-4)',bash_pid='$$',type='new_login',username='$LOGNAME',message='$LOGNAME logged at $(date +"%Y-%m-%d %T") from $(tty | awk -F "/" '{print $3"/"$4}' | xargs -I % bash -c 'w | grep -i %' | awk '{print $3}')'"

function log_command {
    local ctime_hms=$(date +"%T")
    local login_user=$(whoami)
    local sudo_user=$([[ -z "$(echo $SUDO_USER)" ]] && echo "null" || echo "$SUDO_USER")
    local is_root=$([[ "$(id -u)" == "0" ]] && echo "y" || echo "n")
    local shell_status=$([[ "$(id -u)" == "0" ]] && echo "#" || echo "$")
    local remote_ip=$(tty | awk -F "/" '{print $3"/"$4}' | xargs -I % bash -c 'w | grep -i %' | awk '{print $3}')
    local pwd=$(pwd)
    local command=$(echo "$1" | cut -f 4- -d ' ')
    local cmd_retn_code="$2"
    local cmd_pid="$3"
    local sudo_chk=$(echo "$command" | grep -q "sudo" && echo "y" || echo "n")
    local sudo_with=$([[ "$(id -u)" == "0" ]] && echo "y" || echo "$sudo_chk")
    local ps="[$ctime_hms][$login_user@$(hostname)]$pwd~$shell_status $command"

    logger -p local7.notice -t cmd_h1st "datetime='$(date +"%Y-%m-%d %T")',tty='$(tty | cut -d '/' -f 3-4)',bash_pid='$$',type='logged_in',username='$login_user',sudo_user='$sudo_user',root='$is_root',ip='$remote_ip',pwd='$pwd',cmd='$command',cmd_ret_code='$cmd_retn_code',cmd_pid='$cmd_pid',cmd_with_sudo='$sudo_with',ps='$ps'"
}


PROMPT_COMMAND='__ret="$?"; __cmd=$(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//g"); __ppid=$(echo $$); __cpid=$(ps -o ppid= -o pid= | awk "\$1==${__ppid} {print \$2}"); log_command "${__cmd}" "${__ret}" "${__cpid}"'

```

3. Open the "/etc/rsyslog.conf" file and add the configuration to save in local6.info format to the "/var/log/command.log" file.
```
# vim /etc/rsyslog.conf
.....
local7.notice       /var/log/command.log
```

4. Restart the rsyslog service to apply the changes.
```
# systemctl restart rsyslog.service
```

4. If you log out of the shell and log back in, you will see that the /var/log/command.log file has been created.

# result
- example of /var/log/command.log

```bash
# cat /var/log/command.log
...

... cmd_h1st[987170]: datetime='2023-02-22 16:19:21',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='ps -ef',cmd_ret_code='0',cmd_pid='987133',cmd_with_sudo='n',ps='[16:19:21][user1@testwork9]/home/user1~$ ps -ef'
... cmd_h1st[987213]: datetime='2023-02-22 16:19:22',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='ls -al',cmd_ret_code='0',cmd_pid='987176',cmd_with_sudo='n',ps='[16:19:22][user1@testwork9]/home/user1~$ ls -al'
... cmd_h1st[987256]: datetime='2023-02-22 16:19:39',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='ls -al',cmd_ret_code='0',cmd_pid='987219',cmd_with_sudo='n',ps='[16:19:38][user1@testwork9]/home/user1~$ ls -al'
... cmd_h1st[987299]: datetime='2023-02-22 16:19:40',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='w',cmd_ret_code='0',cmd_pid='987262',cmd_with_sudo='n',ps='[16:19:40][user1@testwork9]/home/user1~$ w'
... cmd_h1st[987342]: datetime='2023-02-22 16:19:42',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='ps',cmd_ret_code='0',cmd_pid='987305',cmd_with_sudo='n',ps='[16:19:42][user1@testwork9]/home/user1~$ ps'
... cmd_h1st[987387]: datetime='2023-02-22 16:19:46',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='cat /etc/shadow',cmd_ret_code='1',cmd_pid='987350',cmd_with_sudo='n',ps='[16:19:46][user1@testwork9]/home/user1~$ cat /etc/shadow'
... cmd_h1st[987430]: datetime='2023-02-22 16:19:49',tty='pts/2',bash_pid='983421',type='logged_in',username='user1',sudo_user='null',root='n',ip='192.168.100.1',pwd='/home/user1',cmd='abcdefg',cmd_ret_code='127',cmd_pid='987393',cmd_with_sudo='n',ps='[16:19:49][user1@testwork9]/home/user1~$ abcdefg'

```

# Additional
- If you are using a SIEM, you can receive the logs through syslog and parse them for analysis.
- If necessary make additional fields and apply.
- In case multiple users log in with the same username, we can individually identify who they are.
- By default, when a user change level to root, their  IP address is lost, so making it impossible to recored the user's remote connection ip, but it is fine now. IP is recorded in all cases.
- Final updated at 2023.02.22
