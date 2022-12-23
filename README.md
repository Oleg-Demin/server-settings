# Дипломная работа

## Выбор места размещения сервисов

Для реализации поставленной задачи необходимо решить где будут установленны сервисы. Можно установить сервисы на собственном серверном пространсте (влечет за собой дополнительные издержки по администрированию), если такое имеется в наличии, иначе будет необходимо взять в аренду виртуальный частный сервер (VPS) у компании которая предоставляет такую услугу.

### Анализ возможных решений

В нашем случае будет удобнее взять в аренду VPS, так как не имеется в наличие собственного сервера.

Поставщиком VPS, для данного проекта, является [ItPark](https://dc.itpark.tech/cloud/kvm-servers.php?city=kazan), который был выбран в соответствии с тем что:

1. сервера, на которых будет запущен наш VPS, находятся в Казани
2. служба поддержки отвечает быстро и на Русском языке
3. демократичные цены (ниже большинства конкурентов на рынке)

### Конфигурация виртуального частного сервера

Операционная система: **CentOS 7**

| Название параметра    | Сокращение | Число | СИ |
|-----------------------|------------|-------|----|
| Оперативная память    | RAM        | 7168  | Мб |
| Количество IP-адресов | IP         | 1     | Шт |
| Дисковое пространство | HDD SAS    | 20480 | Мб |
| Количество ядер       | CPU Core   | 2     | Шт |

## Устранение неисправностей сервера (невозможность обновить yum)

При создании машины со стороны

```bash
yum list updates
yum update

ps aux | grep yum
kill -9 8533

shutdown --reboot
yum update
```

## Установка net 6.0 (возможно нужно будет удалить)

Данная настройка в дальнейшем скорее всего будет удалена, так как планируется разворачивать приложения в доккере.

```bash
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm
yum install dotnet-sdk-6.0
```

## Установка Postgresql

```bash
yum install postgres
yum install postgresql-server
postgresql-setup initdb
systemctl enable postgresql
systemctl start postgresql
systemctl status postgresql

sudo -iu postgres
sudo -u postgres psql -l
```

## Первоначальная настройка firewall

```bash
systemctl status -l firewalld
systemctl enable firewalld
systemctl start firewalld
systemctl status -l firewalld

firewall-cmd --zone=public --list-all
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

## Установка Wireguard

### Настройка VPN сервиса на стороне сервера

Скачиваем [wireguard](https://www.wireguard.com/) согласно [инструкции](https://www.wireguard.com/install/). Схемы настройки отличаются в зависимости от платформы на которую будет установлен VPN.

```bash
yum update

yum install yum-utils epel-release
sudo yum-config-manager --setopt=centosplus.includepkgs=kernel-plus --enablerepo=centosplus --save
sudo sed -e 's/^DEFAULTKERNEL=kernel$/DEFAULTKERNEL=kernel-plus/' -i /etc/sysconfig/kernel
sudo yum install kernel-plus wireguard-tools
sudo reboot
```

Генерируем публичные и приватные ключи для VPN сервиса и VPN клиента.

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
wg genkey | tee /etc/wireguard/client_privatekey | wg pubkey | tee /etc/wireguard/client_publickey
chmod 600 /etc/wireguard/privatekey
```

Смотрим какой у нас сетевой интерфейс с помощью команды.

```bash
ip a
```

Искомый интерфейс `eth0`

```text
1: lo: <LOOPBACK,UP,LOWER_UP> ***
        ***
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ***
        ***
```

Просматриваем сгеренированные приватный ключь сервиса VPN и публичный ключь клиента VPN, после чего необходимо создать конфигурационный файл (`wg0.conf`) для нашей частной виртуальной сети.

```bash
cat /etc/wireguard/privatekey
cat /etc/wireguard/client_publickey
nano /etc/wireguard/wg0.conf
```

В файле `wg0.conf` повторяем следующие настройки, заменяя:

1. **\<privatekey\>** на приватный ключь сервиса VPN
2. **\<client_publickey\>** на публичный ключь клиента VPN

```conf
[Interface]
PrivateKey = <privatekey>
Address = 10.0.0.1/24
ListenPort = 51840
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE


[Peer]
PublicKey = <client_publickey>
AllowedIPs = 10.0.0.2/32
```

Открываем порт **51840** по протоколу **udp** для взаимодействия по виртуальной частной сети, а также разрешаем маршрутизацию трафика через нашь VPN сервис.

```bash
firewall-cmd --permanent --zone=public --add-port=51840/udp
firewall-cmd --permanent --zone=public --add-masquerade
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

Откроем файл `sysctl.conf`.

```bash
nano /etc/sysctl.conf
```

Запишем в данный документ следующие строки:

```conf
net.ipv4.ip_forward=1
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
```

Проверяем сохранились ли изменения внесенные файлом `sysctl.conf`.

```bash
sysctl -p
```

Запускаем wireguard сервис.

```bash
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service
systemctl status wg-quick@wg0.service
```

Для подключения к нашей частной виртуальной сети в дьльнейшем нам понадобится знать ***приватный ключь клиента***, ***публичный ключь сервиса*** и ***открытый IP адресс сервера*** на которм расположен VPN. В данных целях необходимо выполнить следующие команды:

```bash
cat /etc/wireguard/client_privatekey
cat /etc/wireguard/publickey
hostname -I
```

### Настройка VPN сервиса на стороне клиента

На локальной машине (например, на ноутбуке) создаём текстовый файл (`client_wireguard.conf`) с конфигурацией клиента заменяя:

1. **\<client-privatekey\>** на приватный ключь клиента VPN
2. **\<publickey\>** на публичный ключь сервиса VPN
3. **\<server-id\>** на открытый IP адресс сервера

```conf
[Interface]
PrivateKey = <client-privatekey>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <publickey>
Endpoint = <server-id>:51840
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```

Запускаем Wireguard на клиентской машине (например, на ноутбуке),
используя [приложение, с официального сайта](https://www.wireguard.com/install/). Версия приложения зависит от платформы на которую будет установлен VPN.

## Установка Minio

Minio это S3 файловое хранилище.  
Скачиваем Minio и выдаем разрешаем на использование сервиса для всех пользователей.

```bash
yum update
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
```

Переносим полученну папку `minio` в репозиторий `/usr/local/bin` .

```bash
mv minio /usr/local/bin
```

Создаем пользователя `minio-user` без возможности входа в систему. И передаем ему права на владение папками ...

```bash
sudo useradd -r minio-user -s /sbin/nologin
chown minio-user:minio-user /usr/local/bin/minio
mkdir /usr/local/share/minio
chown minio-user:minio-user /usr/local/share/minio
mkdir /etc/minio
chown minio-user:minio-user /etc/minio
```

!!!Надо дополнить!!!

```bash
hostname -I
nano /etc/default/minio
```

!!!Надо дополнить!!!  
Запись в файле `/etc/default/minio`

```conf
MINIO_ROOT_USER="<login>"
MINIO_ROOT_PASSWORD="<password>"
MINIO_VOLUMES="/usr/local/share/minio/"
MINIO_OPTS="-C /etc/minio --address <ip-address>:9000 --console-address <ip-address>:9001"
MINIO_SERVER_URL="https://<domen>:9000"
MINIO_BROWSER_REDIRECT_URL="https://<domen>:9001"
```

```bash
curl -O https://raw.githubusercontent.com/minio/minio-service/master/linux-systemd/minio.service
cat minio.service
mv minio.service /etc/systemd/system
```

```bash
firewall-cmd --zone=public --list-all
firewall-cmd --zone=public --add-port=9001/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

```bash
systemctl status -l minio
systemctl daemon-reload
systemctl enable minio
systemctl start minio
systemctl status -l minio
```

### Certbot

```bash
yum update
yum install certbot pythin3-certbot-nginx

certbot certonly --standalone -d wallido.ru -d www.wallido.ru
certbot certificates

cp /etc/letsencrypt/live/wallido.ru/privkey.pem /etc/minio/certs/private.key
cp /etc/letsencrypt/live/wallido.ru/fullchain.pem /etc/minio/certs/public.crt
chown minio-user:minio-user /etc/minio/certs/private.key
chown minio-user:minio-user /etc/minio/certs/public.crt

systemctl restart minio
systemctl status -l minio
```

### Minio mc - терминальный контроллер S3 хранилища

Примеры работы c консольным minio клиентом `mc`: <https://docs.min.io/docs/minio-client-complete-guide.html>

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
mv mc /usr/local/bin
mc alias set minio/ https://<domen>:9000 <login> <password>
mc alias list
mc alias remove minio

mc admin info minio
mc ls minio
mc tree minio
mc tree --files minio

mc mb minio/made-bucket
mc rb minio/remove-bucket
```

## Установка Gitlab

В моем случае [установка ведется для Centos-7](https://about.gitlab.com/install/#centos-7)

### Install and configure the necessary dependencies

On CentOS 7 (and RedHat/Oracle/Scientific Linux 7), the commands below will also open HTTP, HTTPS and SSH access in the system firewall. This is an optional step, and you can skip it if you intend to access GitLab only from your local network.

```bash
sudo yum install -y curl policycoreutils-python openssh-server perl
# Enable OpenSSH server daemon if not enabled: sudo systemctl status sshd
sudo systemctl enable sshd
sudo systemctl start sshd
# Check if opening the firewall is needed with: sudo systemctl status firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld
```

Next, install Postfix to send notification emails. If you want to use another solution to send emails please skip this step and configure an external SMTP server after GitLab has been installed.

```bash
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```

During Postfix installation a configuration screen may appear. Select 'Internet Site' and press enter. Use your server's external DNS for 'mail name' and press enter. If additional screens appear, continue to press enter to accept the defaults.

### Add the GitLab package repository and install the package

Add the GitLab package repository.

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
```

Next, install the GitLab package. Make sure you have correctly set up your DNS, and change [https://gitlab.wallido.ru](https://gitlab.wallido.ru) to the URL at which you want to access your GitLab instance. Installation will automatically configure and start GitLab at that URL.

For https:// URLs, GitLab will automatically request a certificate with Let's Encrypt, which requires inbound HTTP access and a valid hostname. You can also use your own certificate or just use http:// (without s).

If you would like to specify a custom password for the initial administrator user (root), check the documentation. If a password is not specified, a random password will be automatically generated.

```bash
sudo EXTERNAL_URL="https://gitlab.wallido.ru" yum install -y gitlab-ee
```

### Browse to the hostname and login

Unless you provided a custom password during installation, a password will be randomly generated and stored for 24 hours in `/etc/gitlab/initial_root_password`. Use this password with username `root` to login.

See our documentation for detailed instructions on installing and configuration.

### Set up your communication preferences

Visit our **email subscription preference center** to let us know when to communicate with you. We have an explicit email opt-in policy so you have complete control over what and how often we send you emails.

Twice a month, we send out the GitLab news you need to know, including new features, integrations, docs, and behind the scenes stories from our dev teams. For critical security updates related to bugs and system performance, sign up for our dedicated security newsletter.

> **Important note:** If you do not opt-in to the security newsletter, you will not receive security alerts.

## Дальнейние планы

Создать на gitlab 3 прокета:

1. сервис авторизации
2. сервис работы с файлами (ядро)
3. сервис клиент

Обеспечить автоматизацию их развертывание в доккерах при помощи CI/CD.

Обеспечить администрирование проекта (клиент файлового хранилища Minio для администрирования и другое, если на это хватит времени)

## Спасибо за внимание

Решил протестить открытие [файла по ссылке](https://wallido.ru:9000/pictures/BannerSaga%28awatar%29.png), находящегося в публичном баккете Minio.

>Проверить наличие данного файла можно перейдя [по ссылке к клиенту Minio](https://wallido.ru:9001). Войти использую логин `test-user` и пароль `test-user`.

![Спасибо за внимание](https://wallido.ru:9000/pictures/BannerSaga%28awatar%29.png)
