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
   Port knocking - это скрытый метод для внешнего открытия портов, которые по умолчанию файервол держит закрытыми. Для работы данного метода требуется ряд попыток подключения к заранее определённым закрытым портам. Обращение к этим портам должно осуществляться в заданной последовательности. При соблюдении вышеуказанных условий, файервол открывает для подключения необходимый порт.<br/>
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
   В последнем, из приведённых выше правил, происходит открытие 22-го порта на 30 секунд, при условии, что подключающийся ip находится в списке SSH2. Если в соответствии с данным правилом трафик не принимается (например, не было попыток подключения в течение 30 секунд), но подключающийся IP-адрес находится в списке SSH2, он удаляется из этого списка для осуществления проверки с самого начала:
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
   ```
   После того, как конец последовательности был обработан первым, следующие правила выполняют проверку следования портов. Если проверка пройдена, то происходит переход к цепочке, в которой ip-адрес добавляется в список для следующего запроса. Если перехода на цепочки SSH-INPUT или SSH-INPUTTWO не произошло, то это может свидетельствовать о неверных портах в последовательности или о наличии неустановленного трафика. После чего, по аналогии с предыдущим правилом для списка SSH2, осуществляется удаление ip-адреса из списка SSH1 и отбрасывание трафика:
   ```shell
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
   iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
   ```
   Подобные действия осуществляются для проверки следующего порта. Порядок следования в цепочке TRAFFIC может быть любым при условии, что правила, соответствующие одному и тому же списку, сохраняются вместе и в правильном порядке:
   ```shell
   iptables -A TRAFFIC -i eth1 -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
iptables -A TRAFFIC -i eth1 -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
   ``` 
   
   
   
    


