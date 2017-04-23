1. Настройка нового сервера
===========================

- Настроить локаль::

    Для Ubuntu:
    # locale-gen ru_RU.utf8
    # dpkg-reconfigure locales

    Для Debian:
    # dpkg-reconfigure locales
    Поставить галки на en_US.UTF-8 и ru_RU.UTF-8, выбрать по умолчанию ru_RU.UTF-8

- Настроить временую зону::

    # dpkg-reconfigure tzdata

- Если установлен apache, то или перевести его на порт отличный от 80 или удалить::

    # apt-get remove apache2 apache2.2-bin

- Для Debian:

 - Загрузить ключ для apt от nginx и varnish и установить их::

    # wget -qO - http://nginx.org/keys/nginx_signing.key | apt-key add -
    # wget -qO - http://repo.varnish-cache.org/debian/GPG-key.txt  | apt-key add -

 - Добавить источники для установки новых версий nginx и varnish::

    # echo "deb http://nginx.org/packages/debian/ squeeze nginx" >> /etc/apt/sources.list.d/nginx.list
    # echo "deb-src http://nginx.org/packages/debian/ squeeze nginx" >> /etc/apt/sources.list.d/nginx.list
    # echo "deb http://repo.varnish-cache.org/debian/ squeeze varnish-3.0" >> /etc/apt/sources.list.d/varnish.list

- Выполнить следующие команды::

    # apt-get update
    # apt-get upgrade
    # apt-get install nginx varnish build-essential python-dev libxslt1-dev zlib1g-dev libfreetype6-dev
    # apt-get install mysql-server libmysqlclient-dev supervisor mc htop
    Для Ubuntu:
    # apt-get install libjpeg8-dev
    Для Debian:
    # apt-get install libjpeg62-dev

- Cоздать базу и пользователя в MySQL::

    # mysql --user=root -p
    mysql> CREATE SCHEMA `easymon` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
    mysql> CREATE USER `easymon`@`localhost` IDENTIFIED BY 'some_password';
    mysql> GRANT ALL PRIVILEGES ON `easymon`.* TO `easymon`@`localhost` WITH GRANT OPTION;
    mysql> exit;

  Если забыли пароль от root пользователя то можно воспользоваться следующей инструкцией::

    You can reset the root password by running the server with --skip-grant-tables and logging in
    without a password by running the following as root (or with sudo):

    # service mysql stop
    # mysqld_safe --skip-grant-tables &
    $ mysql -u root

    mysql> use mysql;
    mysql> update user set password=PASSWORD("YOUR-NEW-ROOT-PASSWORD") where User='root';
    mysql> flush privileges;
    mysql> quit

    # service mysql stop
    # service mysql start
    $ mysql -u root -p

- Если на сервере нет сайтов, которые используют системный varnish, то надо запретить его запуск при загрузке сервера::

    # services varnish stop

    Для Ubuntu:
    # echo 'manual' > /etc/init/varnish.override

    Для Debian надо в файле /etc/default/varnish поменять START=yes на START=no

- Добавить системного пользователя::

    # adduser easymon

2. Установка сайта
==================

- Залогинится под пользователем и перейти в его домашнюю папку::

    # su - easymon

- Распаковать архив с релизом сайта. Перейти в папку с сайтом::

    $ tar -xf easymon-<name>-<version>.tar.bz2
    $ cd easymon

- Заменить в файле buildout.cfg нужные настройки (параметры подключения к базе данных, настройки SMTP,
  домен сайта и др.)::

    $ mcedit buildout.cfg

- Выполнить следующие команды::

    $ python bootstrap.py
    $ ./bin/buildout

3. Создание и наполнение таблиц в базе данных
=============================================

- Для создания таблиц в базе данных выполните следующую команду::

    $ ./bin/manage.py createdb

Будет задано несколько вопросов. На все вопросы типа Yes/No - отвечать "yes",
кроме одного в котором спрашивается про добавление демонстрационного контента на
сайт (Would you like to install some initial content?).
Будет предложенно ввести данные администратора: логин, email, пароль.
Так же надо будет ввести правильно домен на котором будет распологаться сайт (используется при отправке писем).

- Загрузить в базу начальные данные::

    ./bin/manage.py loaddata flatblocks.json modifications.json games.json colors.json services.json
    ./bin/manage.py loaddata settings.json

3.1 Массовая загрузка списка серверов
-------------------------------------

Для массовой загрузки списка серверов надо выполнить следующую команду::

    ./bin/manage.py load_servers cs-16 admin path/to/servers.txt

где cs-16 - это идентификатор игры (cs-16, css, csgo);
    admin - логин существующего в базе пользователя, который будет владельцем добавленых серверов;
    path/to/servers.txt - путь к файлу со списком серверов (в каждой строчке по одному серверу в формате host:port)

3.2 Экспорт картинок для карт
-----------------------------

Для экспорта картинок для карт выполните следующую команду::

    ./bin/manage.py export_maps cs-16

где cs-16 - это идентификатор игры (cs-16, css, csgo);
В текущей директории будет создана папка с именем вида <game>_maps (например: cs-16_maps).
В этой папку будут находится все картинки карт.

3.3 Импорт картинок для карт
----------------------------

Для импорта картинок для карт выполните следующую команду::

    ./bin/manage.py import_maps cs-16

где cs-16 - это идентификатор игры (cs-16, css, csgo);
Картинки для карт будут браться из папки с именем вида <game>_maps (например: cs-16_maps).
Если в базе уже есть картинки для карт с такимиже именами как импортируемые, то они будут перезаписаны на новые.

4. Настройка автозапуска
========================

- Вернуться под пользователя root (Ctrl + D).

- В папке /etc/supervisor/conf.d создать симлинк на конфигурацию супервизора::

    # cd /etc/supervisor/conf.d
    # ln -fs /home/easymon/easymon/etc/supervisor/easymon.conf easymon.conf

- Для Ubuntu в папке /etc/nginx/sites-enabled создать симлинк на конфигурацию nginx::

    # cd /etc/nginx/sites-enabled
    # ln -fs /home/easymon/easymon/etc/nginx/easymon.conf easymon.conf

- Для Debian в папке /etc/nginx/conf.d создать симлинк на конфигурацию nginx::

    # cd /etc/nginx/conf.d
    # ln -fs /home/easymon/easymon/etc/nginx/easymon.conf easymon.conf

5. Запуск
=========

- Перезапустить nginx и supervisord::

    # service nginx restart
    # service supervisor stop
    # service supervisor start

6. Перенос на другой сервер
===========================

- Создать бекап базы данных.

- Создать бекап папки /home/easymon/easymon/easymonsite/media.

- Настроить новый сервер по пунктам 1, 2 и 4.

- Восстановить на новом сервере из бекапа базу данных.

- Восстановить на новом сервере из бекапа папку /home/easymon/easymon/easymonsite/media.

- Запустить сайт (пункт 5)

7. Обновление
=============

- Не обязательно, но желательно удалить папку /home/easymon/easymon/src

- Скопировать из архива с заменой все файлы кроме buildout.cfg (или вместе с ним, но тогда надо будет заново
  поправить в нём настройки). Если вы меняли какие то файлы (например тему в папке themes), то необходимость их
  копирования из обновления решайте сами.

- Выполнить команды::

    $ python bootstrap.py
    $ ./bin/buildout
    $ ./bin/manage.py migrate

- Под рутом перезапустить nginx и supervisord::

    # service nginx restart
    # service supervisor stop
    # service supervisor start

PS:

- При обновлении до версии 2.8.0+ с более старых версий необходимо перед выполнением миграции выполнить
под root-ом следующие команды::

    # mysql
    mysql> use easymon;
    mysql> ALTER TABLE `core_sitepermission` ADD CONSTRAINT `core_sitepermission_user_id_d964e296aed9970_uniq` UNIQUE (`user_id`);
    mysql> exit;

