# Установка сервера 1С Предприятие на Ubuntu сервер

1. Переносим установочный дистрибутив на сервер

- Перенос одного файла
scp "путь к файлу" user@ip-сервера:/tmp/1c_distrub/

- Перенос целой папки (флаг -r)
scp -r "путь к файлу" user@ip-сервера:/tmp/

2. Подключаемся к удаленной машине по ssh

ssh -i "путь к файлу ключа" user@ip-сервера

3. Устанавливаем зависимости и сразу архиватор

sudo apt update
sudo apt install libenchant1c2a ttf-mscorefonts-installer libgsf-1-114 tar

4. Распаковываем архив

tar -xvzf /tmp/1c_distrub/setup-full-8.5.1.1150-x86_64.run.gz

5. Делаем файл исполняемым

sudo chmod +x /tmp/1c_distrub/setup-full-8.5.1.1150-x86_64.run

6. Запускаем установщик в текстовом режиме

sudo /tmp/1c_distrub/setup-full-8.5.1.1150-x86_64.run --mode text

7. Запуск службы

sudo systemctl status srv1cv8-8.5.1.1150

или

sudo systemd status srv1cv8-8.5.1.1150

8. Настройка автозапуска

sudo systemctl enable srv1cv8-8.5.1.1150

# Установка сервера PostgreSQL

1. Переносим установочный дистрибутив на сервер

scp "путь к файлу" user@ip-сервера:/tmp/pg_distrub/

2. Подключаемся к удаленной машине по ssh

ssh -i "путь к файлу ключа" user@ip-сервера

3. Устанавливаем локаль

sudo locale-gen ru_RU.UTF-8
sudo update-locale LANG=ru_RU.UTF-8

4. Устанавливаем Postgres

sudo wget https://repo.postgrespro.ru/1c/1c-17/keys/pgpro-repo-add.sh
sudo sh pgpro-repo-add.sh
sudo apt-get install postgrespro-1c-17

5. Инициализация и запуск

sudo /opt/pgpro/1c-17/bin/pg-setup initdb --tune=1c --locale=ru_RU.UTF-8

sudo systemctl enable postgrespro-1c-17
sudo systemctl start postgrespro-1c-17

6. Установка пароля

sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'ваш_пароль';"

7. Разрешение внешних подключений

- отредактировать файл, установив listen_addresses = '*'.
/var/lib/pgpro/1c-17/data/postgresql.conf

- в файле pg_hba.conf добавить строку для используемой сети:

TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all           postgres        192.168.1.10/32         md5

8. Перезапуск Postgres

sudo systemctl restart postgrespro-1c-17


# Создание информационной базы

1. Запуск сервера администрирования (ras)

/opt/1cv8/x86_64/8.5.1.1150/ras cluster --daemon

2. Узнаем ID кластера

/opt/1cv8/x86_64/8.5.1.1150/rac cluster list

3. Создаем базу

/opt/1cv8/x86_64/8.3.xx.xxxx/rac infobase --cluster=ID-КЛАСТЕРА create --name=MyBase --dbms=PostgreSQL --db-server=адрес_сервера  --db-name=mybase_db --db-user=postgres --db-pwd=пароль_postgres --license-distribution=allow --locale=ru_RU --create-database

# Установка и запуск Apache

1. Установка

sudo apt-get update
sudo apt-get install apache2

2. Подготовка каталога для публикации

sudo mkdir -p /var/www/html/testbase

sudo chown -R www-data:www-data /var/www/html/testbase

3. Публикация

sudo /opt/1cv8/x86_64/8.5.1.1150/webinst -apache24 -wsdir TestBase -dir /var/www/html/testbase -connstr "Srvr=10.130.0.10;Ref=TestBase;" -confPath /etc/apache2/apache2.conf

4. Перезапуск Apache

sudo systemctl restart apache2

5. Вход

http://<IP_сервера_Apache>/TestBase

# Загрузка конфигурации

1. Переносим .cf на сервер

scp "путь к файлу" user@ip-сервера:/tmp/metadata.cf

2. /opt/1cv8/x86_64/8.5.1.1150/1cv8 DESIGNER \
/S "<IP_сервера_Postgres>\TestBase" \
/LoadCfg "/tmp/metadata.cf" \
/N "Администратор"/
/P "ВашПароль"\
/UpdateDBCfg

•	DESIGNER: запуск в режиме конфигуратора.
•	/S: строка подключения ("IP_сервера_1С\Имя_Базы").
•	/LoadCfg: путь к файлу конфигурации.
•	/UpdateDBCfg: обязательный флаг, который применяет загруженную конфигурацию к структуре таблиц в PostgreSQL (аналог нажатия F7 в графическом интерфейсе).

# Активация лицензии

1. Назначение прав пользователю 1С

sudo chown -R usr1cv8:grp1cv8 /var/1C/1cv8/conf/

2. Запуск утилиты активации

cd /opt/1cv8/x86_64/8.5.1.1150/ && ./licmngr --activate


# Установка Docker

1. Установка
   sudo apt update
   sudo apt install docker.io -y

2. Запуск службы

sudo systemctl enable --now docker

3. Настройка Post-install

sudo groupadd docker

sudo usermod -aG docker $USER

4. Чтобы не перезагружать компьютер, можно обновить права в текущей сессии терминала:

newgrp docker

5. Проверка

docker run hello-world

6. Установка Docker Compose

sudo apt-get update && sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update && sudo apt-get install docker-compose-plugin

7. Проверка

docker compose version

# Запуск системы мониторинга

1. Перенос конфигурации на сервер

ssh -i <файл_ключа> пользователь@ip-сервера "mkdir -p ~/my-monitoring"

scp -i <файл_ключа> docker-compose.yml prometheus.yml custom.ini <username>@<IP_сервера>:~/my-monitoring

2. Запуск

cd ~/my-monitoring
docker compose up -d

3. Проверка

docker compose ps
docker logs ivanov-aa-netology-grafana

Grafana: http://<IP_сервера>:8080
Prometheus: http://<IP_сервера>:9090