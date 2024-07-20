# Сценарии iptables
1. Реализовать knocking port:
   - centralRouter может попасть на ssh inetRouter через knock скрипт;
2. Добавить inetRouter2, который виден (маршрутизируется (host-only тип сети для виртуалки))<br/>
   с хоста или форвардится порт через локал хост;
3. Запутсить NGINX на centralServer;
4. Проборосить 80-й порт на inetRouter2 c портом 8080;
5. Дефолт в интернет оставить через inetRouter;
   Дополнительное задание: реализовать проход на 80-й порт без маскарадинга.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0 и образа ubuntu/jammy64 версии 20240301.0.0. <br/>
### Ход решения ###
&ensp;&ensp;Для выполнения данного задания в качестве основы использовалась схема, реализованная в ДЗ "Архитектура сетей":

![изображение](https://github.com/user-attachments/assets/8841be0a-1533-4fb0-b7a7-2a5626706452)

1. Реализация port knocking.
   Port knocking - это скрытый метод для внешнего открытия портов, которые по умолчанию фаервол держит закрытыми. Для работы данного метода требуется ряд попыток подключения к заранее определённым закрытым портам. Обращение к этим портам должно осуществляться в заданной последовательности. При соблюдении вышеуказанных условий, фаервол открывает для подключения необходимый порт.<br/>
   Port knocking метод возможно реализовать с помощью демона knockd либо посредством iptables.<br/>
   Ниже приводится реализация метода port knocking посредством настройки iptables.
   - Создание дополнительных цепочек TRAFFIC, SSH-INPUT, SSH-INPUTTWO:
   ```shell
   iptables -N TRAFFIC
   iptables -N SSH-INPUT
   iptables -N SSH-INPUTTWO
   ```
   - Добавление правил для основной цепочки TRAFFIC:
   ```shell
   iptables -A INPUT -i enp0s8 -j TRAFFIC
   iptables -A TRAFFIC -i enp0s8 -m state --state ESTABLISHED,RELATED -j ACCEPT
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
   ```
   В последнем, из приведённых выше правил, происходит открытие 22-го порта на 30 секунд, при условии, что подключающийся ip находится в списке SSH2. Если в соответствии с данным правилом трафик не принимается (например, не было попыток подключения в течение 30 секунд), но подключающийся IP-адрес находится в списке SSH2, он удаляется из этого списка для осуществления проверки с самого начала.
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
   ```
   После того, как конец последовательности был обработан первым, следующие правила выполняют проверку следования портов. Если проверка пройдена, то происходит переход к цепочке, в которой ip-адрес добавляется в список для следующего запроса. Если перехода на цепочки SSH-INPUT или SSH-INPUTTWO не произошло, то это может свидетельствовать о неверных портах в последовательности или о наличии неустановленного трафика. После чего, по аналогии с предыдущим правилом для списка SSH2, осуществляется удаление ip-адреса из списка SSH1 и отбрасывание трафика.
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
   ```
   Аналогичные действия осуществляются для проверки следующего порта. Порядок следования в цепочке TRAFFIC может быть любым при условии, что правила, соответствующие одному и тому же списку, сохраняются вместе и в правильном порядке.
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
   ```
   В последнем блоке правил для завершения этапа проверки knocking-последовательности, осуществляется попытка установления соединения для ip, соответствующего списку разрешённых ip-адресов. Первый порт в последовательности проверяется как часть основной цепочки TRAFFIC, поскольку каждая новая попытка подключения может являться началом процедуры port knocking. В случае успеха проверки (правильный порт), порт записывается в первый список SSH0. Далее в правиле для проверки второго порта (7777), из списка SSH0 требуется подтверждение проверки первого порта, только после этого производится запись о втором порте в список SSH1. В свою очередь, список SSH1 требуется для проверки третьего порта (9991). В последних правилах блока осуществляется сброс трафика для портов из списков, несмотря на то, что они корректны. Эти действия необходимы для сокрытия успешности подключения к каждому из портов.
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
   iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP
   iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
   iptables -A TRAFFIC -i enp0s8 -j DROP
   ```    
   - Настройка port knocking со стороны клиента. Для того, чтобы со стороны клиента постучаться в порты, используется утилита nmap. Данное действие реализуется с помощью bash-скрипта:
   ```shell
   #!/bin/bash
   HOST=$1
   shift
   for ARG in "$@"
   do
      nmap -Pn --max-retries 0 -p $ARG $HOST
   done
   ```
   - Проверка метода port knocking:
   ```shell
   vagrant@centralRouter:~$ ssh vagrant@192.168.255.1
   ^C
   
   vagrant@centralRouter:~$ ./knock 192.168.255.1 8881 7777 9991
   Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
   Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
   Nmap scan report for 192.168.255.1
   Host is up.

   PORT     STATE    SERVICE
   8881/tcp filtered galaxy4d

   Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
   Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
   Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
   Nmap scan report for 192.168.255.1
   Host is up.

   PORT     STATE    SERVICE
   7777/tcp filtered cbt

   Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
   Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
   Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
   Nmap scan report for 192.168.255.1
   Host is up.

   PORT     STATE    SERVICE
   9991/tcp filtered issa

   Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
   
   vagrant@centralRouter:~$ ssh vagrant@192.168.255.1
   The authenticity of host '192.168.255.1 (192.168.255.1)' can't be established.
   ED25519 key fingerprint is SHA256:dEof/6Fm9Iy3IvzJwcYS1k0EAtuEJAVNkVA7dZbaPBU.
   This key is not known by any other names
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   Warning: Permanently added '192.168.255.1' (ED25519) to the list of known hosts.
   vagrant@192.168.255.1's password: 
   Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)
   ...
   Last login: Sat Jul 20 12:28:39 2024 from 192.168.56.1
   vagrant@inetRouter:~$
   ```
2. Проброс порта.
   - После установки NGINX на centralServer (192.168.0.2), необходимо пробросить порт 8080 inetRouter2 (192.168.255.14) на порт 80 centralServer. Данное действие осуществляется с помощью добавления правила в iptables:
```shell
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.0.2:80
```
   - В рамках выполнения дополнительного задания по реализации прохода на 80-й порт без маскарадинга, осуществлялась настройка статического NAT для трафика 80-го порта centralServer с адресом 192.168.0.2:
```shell
iptables -t nat -A POSTROUTING -o enp0s8 -p tcp -d 192.168.0.2 --dport 80 -j SNAT --to-source 192.168.255.14:8080
```
   - Проверка работы настроенной системы:

![изображение](https://github.com/user-attachments/assets/094b93a1-6382-43d8-ac04-92e080aacd56)

   
   
   
    


