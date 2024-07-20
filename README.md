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
   
    


