# IPSecOverL2TP
Server - Ubuntu 18.04  

Client - Ubuntu 18.04

[Шаг 1. Установка StrongSwan и L2TP](#шаг-1-установка-strongswan-и-l2tp)

[Шаг 2. Настройка /etc/ipsec.conf](#шаг-2-настройка-/etc/ipsec.conf)

[Шаг 3. Настройка Pre-Shared Key](#шаг-3-настройка-pre-shared-key)

[Шаг 4. Настройка L2TP](#шаг-4-настройка-l2tp)

[Шаг 5. Добавление пользователей](#шаг-5-добавление-пользователей)

[Шаг 6. Перезагрузка сервисов](#шаг-6-перезагрузка-сервисов)

[Шаг 7. Настройка форвардинга](#шаг-7-настройка-форвардинга)

[Шаг 8. Настройка клиентов](#шаг-8-настройка-клиентов)

## Шаг 1. Установка StrongSwan и L2TP

Во-первых, мы установим StrongSwan, демон IPSec с открытым исходным кодом, который мы настроим как наш VPN-сервер. Мы также установим компонент инфраструктуры открытого ключа, чтобы мы могли создать центр сертификации.  

Обновим пакеты и установим нужные компоненты:  

```
sudo apt update  
sudo apt-get install strongswan xl2tpd
```
## Шаг 2. Настройка /etc/ipsec.conf 

Создадим раздел конфигурации для нашего VPN. Для этого добавим следующие строки в файл /etc/ipsec.conf:  

```
conn L2TP-IPSEC
    authby=secret
    rekey=no
    keyingtries=3
    type=transport
    esp=aes128-sha1
    ike=aes128-sha-modp1024
    ikelifetime=8h
    keylife=1h
    left=XXX.XXX.XXX.XXX # XXX.XXX.XXX.XXX – IP-адрес сервера
    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any
    rightsubnet=0.0.0.0/0
    auto=add
    dpddelay=30
    dpdtimeout=120
    dpdaction=clear
    #force all to be nat'ed. because of iOS
    forceencaps=yes
```
## Шаг 3. Настройка Pre-Shared Key

Изменим файл /etc/ipsec.secrets:

```
# This file holds shared secrets or RSA private keys for authentication.
# RSA private key for this host, authenticating it to any other host
# which knows the public part.
XXX.XXX.XXX.XXX %any : PSK "TypeYourPassPhraseHere" # XXX.XXX.XXX.XXX – IP-адрес сервера
```
## Шаг 4. Настройка L2TP

Изменим файл /etc/ppp/options.xl2tpd:

```
sudo nano /etc/ppp/options.xl2tpd
```
Для этого добавим следующие строки в файл:

```
require-mschap-v2
refuse-mschap
ms-dns 8.8.8.8
ms-dns 8.8.4.4
asyncmap 0
auth
crtscts
idle 1800
mtu 1410
mru 1410
connect-delay 5000
lock
hide-password
local
#debug
modem
name l2tpd
proxyarp
lcp-echo-interval 30
lcp-echo-failure 4
```
Также изменим файл /etc/xl2tpd/xl2tpd.conf:

```
[global]
ipsec saref = no
debug tunnel = no
debug avp = no
debug network = no
debug state = no
access control = no
rand source = dev
port = 1701
auth file = /etc/ppp/chap-secrets

[lns default]
ip range = 192.168.1.10-192.168.122.20
local ip = 192.168.1.1
require authentication = yes
name = l2tp
pass peer = yes
ppp debug = no
length bit = yes
refuse pap = yes
refuse chap = yes
pppoptfile = /etc/ppp/options.xl2tpd

```

## Шаг 5. Добавление пользователей

Необходимо настроить авторизацию для L2TP over IPSec путем добавления пользователей и их паролей. Для этого добавьте их в файл /etc/ppp/chap-secrets:

```
test    l2tpd     TestTest      "*"
```
## Шаг 6. Перезагрузка сервисов

Чтобы убедиться в загрузке конфигурации, перезапустим сервис L2TP и StrongSwan:

```
sudo service xl2tp start/restart
sudo service strongswan start/restart
```
## Шаг 7. Настройка форвардинга

Необходить включить форвардинг IP на L2TPoverIPSec-сервере. Это позволит пересылать пакеты между публичным IP и приватными IP. Изменим /etc/sysctl.conf:

```
net.ipv4.ip_forward = 1
```
Для применения изменений выполните команду:

```
sysctl -p
```
## Шаг 8. Настройка клиентов

> Примечание. По умолчание графическое окно линукс интерфейса маленького размера. Далее при добавлении VPN мы столкнемся с тем, что не все настройки будут помещаться на экране. Для увеличения разрешения остановим ноду. В ее конфигурации(правой кнопкой -> Edit) в пункте QEMU custom options меняем -vga std на -vga qxl. Нажимаем Save.

Для начала обновим и установим следующие пакеты:

```
sudo apt-get update
sudo apt-get install network-manager-l2tp
sudo apt-get install network-manager-l2tp-gnome
```
Для подключения к VPN воспользуемся следующей инструкцией. Заходим в Settings -> Network.

![Настройка](https://sun1-86.userapi.com/t5VBHad3DOLRkw3K8OAfB6hzdb6gIfb2aMzs6w/dqIqedhAibE.jpg )

В разделе VPN нажимаем на "+".

![Добавление VPN](https://sun1-30.userapi.com/95EqMIPesHVw5dMFv_VGvVbIIaCgI0IRJoRw4A/QAxCCH3JF9A.jpg )

В появившемся окне выбираем **Layer 2 Tunneling Protocol (L2TP).**

![Выбор L2TP](https://sun1-98.userapi.com/2pkLLfjPaC1FBXuClphXLB63u3QMvnxe006pIw/cTSGJGxTkOY.jpg )

В поле **Name** даем любое название VPN.

В поле **Gateway** указываем IP-адрес VPN сервера.

В поле **User name** и **Password** вводим логин и пароль для пользователя, указанные в шаге 5. Для нашего примера username - test, password - TestTest. (Для ввода пароля нужно нажать на знак вопроса в поле Password и выбрать "Store the password for all users". В противном случае система будет запрашивать пароль при каждом VPN соединении).

![Ввод данных](https://sun1-24.userapi.com/2lIYrR9sHniUhwBl92ApK5Wwj6Tte029dJernA/o1FnQQ-gZ9c.jpg )

Далее выбираем пункт **PPP Settings..**

Ставим галочку на пункте **Use Point-to-point encryption(MPPE).** И в поле **Security** выбираем **128-bit most secure.**

В разделе **Misc** устанавливаем значения **MTU** и **MRU** в 1410.

![PPP settings](https://sun1-22.userapi.com/ZfQxQMuUPweRYkpVB9WAngno9g03Re613OaFKA/ZhDcFjF7Kzc.jpg )

Нажимаем **OK** и переходим в **IPSec Settings**. В **Pre-shared key** указываем фразу из шага 3. Для нашего примера данной фразой будет TypeYourPassPhraseHere

![IPSec settings](https://sun1-26.userapi.com/eGowqrxIuikJXwaPoFYdA97C3WjxkOw_SooKMg/rIhOWsk6CQI.jpg )

Выбираем **Okey** и нажимаем **Add**. Настройка клиента завершена. 

Кликаем на ползунок и при успешном подключении в верхней строке состояния должен появиться значок с замком.

