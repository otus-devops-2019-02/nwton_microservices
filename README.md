# nwton_microservices
nwton microservices repository

# HW14. Технология контейнеризации. Введение в Docker.
## docker-1

Собираю свою собственную лабу через vagrant+virtualbox
с Ubuntu 16.04 и 18.04 для простоты работы под Windows:
``` text
cd my-lab
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt

vagrant up docker-lab-16
vagrant provision docker-lab-16
vagrant ssh docker-lab-16

ssh-add .vagrant/machines/docker-lab-16/virtualbox/private_key
ssh vagrant@10.10.10.16

deactivate
```
Можно использовать ключ из vagrant, а можно добавить свой
ключ в инстанс дюжиной разных способов:
- https://stackoverflow.com/questions/30075461/how-do-i-add-my-own-public-key-to-vagrant-vm


## Установка docker под WSL

Документация на будущее (кратко: последние бесплатные версии
docker под windows используют Hyper-V, а не virtualbox, поддержку
virtualbox удалили, а напрямую из WSL работать не будет по вполне
понятным причинам, при этом можно использовать TCP подключение
к основному демону через указание DOCKER_HOST):
- https://www.freecodecamp.org/news/how-to-set-up-docker-and-windows-subsystem-for-linux-a-love-story-35c856968991/
- https://medium.com/nuances-of-programming/%D0%BA%D0%B0%D0%BA-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B8%D1%82%D1%8C-docker-%D0%B8-windows-subsystem-for-linux-wsl-%D0%B8%D1%81%D1%82%D0%BE%D1%80%D0%B8%D1%8F-%D0%BE-%D0%BB%D1%8E%D0%B1%D0%B2%D0%B8-5259c1e90589
- https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly
- https://nickjanetakis.com/blog/docker-tip-73-connecting-to-a-remote-docker-daemon
- https://github.com/kaisalmen/wsltooling/blob/world/scripts/install/installDocker.sh

Поправка: есть сведения, что в последних версиях WSL включили
правильную поддержку cgroups и там docker-ce версии 17.09.0
даже работает без всяких проблем:
- https://medium.com/faun/docker-running-seamlessly-in-windows-subsystem-linux-6ef8412377aa
- https://github.com/Microsoft/WSL/issues/2291#issuecomment-383698720
- https://github.com/Microsoft/WSL/issues/2291#issuecomment-471235931
- https://github.com/Microsoft/WSL/issues/2291#issuecomment-475941596

На моём компьютере удалось запустить демон только добавлением строчки
`DOCKER_OPTS="--iptables=false --bridge=none"`
в файле /etc/default/docker, после этого сервис стартовал и получилось
запустить тестовые контейнеры:
``` text
docker run --network host hello-world
docker run --network host -it busybox sh
docker run --network host -it alpine sh
```

Как PoC работы docker под WSL подойдёт, но интереснее полноценная
версия работающая в виртуалке.

## Установка docker под Linux

Требуемые версии софта для ДЗ
* Docker – 17.06+
* docker-compose – 1.14+
* docker-machine – 0.12.0+

Работаем в VM под Ubuntu 16.04 Xenial
- https://docs.docker.com/install/linux/docker-ce/ubuntu/
``` text
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Установка именно определённой версии
``` text
apt-cache madison docker-ce
apt-cache madison docker-ce-cli
apt-cache madison containerd.io

apt-cache policy docker-ce
apt-cache policy docker-ce-cli
apt-cache policy containerd.io

sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

## Основное задание

Для того, чтобы можно было запускать docker не только от root, но и от
текущего пользователя, добавляем себя в группу и перелогиниваемся:
``` text
$ docker run hello-world
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.39/containers/create: dial unix /var/run/docker.sock: connect: permission denied.

$ ls -l /var/run/docker.sock
srw-rw---- 1 root docker 0 Jul  4 14:18 /var/run/docker.sock

$ sudo usermod -aG docker $USER

$ cat /etc/group | grep docker
docker:x:999:vagrant
```

Проверка работы и основные команды:
``` text
docker version
docker info

docker run hello-world
docker run -it ubuntu bash

docker ps
docker ps -a
docker images

docker run -it ubuntu bash
docker run -it ubuntu:18.04 /bin/bash
    -i - запуск контейнера в foreground-режиме (docker attach).
    -d - запуск контейнера в background-режиме.
    -t - создание TTY.
    --rm - после остановки контейнера удалить с диска, иначе
        контейнер останется и можно подключиться через attach
    docker run = docker create + docker start

docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.CreatedAt}}\t{{.Names}}"
docker create ...
docker start <container-id>
docker attach <container-id>
docker exec -it <container-id> bash

docker commit <container-id> <image-name>
docker inspect <u_container_id>
docker inspect <u_image_id>

docker ps -q
docker kill $(docker ps -q)
docker stop $(docker ps -q)

docker system df
docker rm $(docker ps -a -q) # удалит все незапущенные контейнеры
docker rmi $(docker images -q) # удалит все образы, кроме запущенных контейнеров
```

## Дополнительное задание
Сравните вывод двух следующих команд:
- docker inspect <u_container_id>
- docker inspect <u_image_id>
На основе вывода команд объясните, чем отличается контейнер от образа.

root@docker-16:~# docker inspect 8b5eb26cce58 > /vagrant/docker_inspect_container.log
root@docker-16:~# docker inspect b2ba41d58bcb > /vagrant/docker_inspect_image_cont.log
root@docker-16:~# docker inspect 4c108a37151f > /vagrant/docker_inspect_image_ubuntu.log

Образ - содержимое диска контейнера (_виртуальной машины_).
При создании нового образа через commit контейнера добавляется еще один слой в Layers
с новым sha256 в дополнении к тем, что были в базовом образе, с текущим состоянием
файловой системы инстанса.
Контейнер - запущенный экземпляр _виртуальной машины_ из образа, имеет только ссылку
на образ и все остальные атрибуты инстанса - лимиты памяти и CPU, сетевые настройки,
текущий статус (запущен или нет) и пр.


# HW15. Docker контейнеры. Docker под капотом
## docker-2

## Предварительная настройка
Создаем проект docker
- https://console.cloud.google.com/compute

Установка GCloud SDK уже была в HW6, меняем проект и проверяем
``` text
gcloud info
gcloud init
gcloud auth application-default login
...
Credentials saved to file:
    [~/.config/gcloud/application_default_credentials.json]

These credentials will be used by any library that requests
Application Default Credentials.

To generate an access token for other uses, run:
  gcloud auth application-default print-access-token
```

## docker-machine

Установка docker-mashine for Linux
- https://docs.docker.com/machine/install-machine/
``` text
$ base=https://github.com/docker/machine/releases/download/v0.16.0 &&
  curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
  sudo install /tmp/docker-machine /usr/local/bin/docker-machine
```

Запуск:
```
gcloud info | grep project
export GOOGLE_PROJECT=docker-245721

docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-zone europe-west1-b \
    docker-host


docker-machine ls

eval $(docker-machine env docker-host)
```

Содержимое передаваемое в eval
``` text
$ docker-machine env docker-host
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://xxxxx:2376"
export DOCKER_CERT_PATH="~/.docker/machine/machines/docker-host"
export DOCKER_MACHINE_NAME="docker-host"
```

Переключение на удалённый docker
(все команды docker будут выполняться на удалённом хосте):
``` bash
eval $(docker-machine env <имя>)
```

Переключение на локальный докер:
``` bash
eval $(docker-machine env --unset)
```

Удаление инстанса:
```bash
docker-machine rm <имя>
```

## Основное задание

Сравните сами вывод:
``` text
docker run --rm -ti tehbilly/htop
docker run --rm --pid host -ti tehbilly/htop
```

В первом случае мы работаем в ограниченном namespace,
во втором варианте делаем "побег из курятника" до
уровня хоста.

Построение образа и запуск контейнера с приложением в GCP:
```
cd docker-monolith
docker build -t reddit:latest .
docker images -a

docker run --name reddit -d --network=host reddit:latest
docker-machine ls

gcloud compute firewall-rules create reddit-app \
    --allow tcp:9292 \
    --target-tags=docker-machine \
    --description="Allow PUMA connections" \
    --direction=INGRESS
```

Загрузка образа на Docker Hub
``` text
docker login
docker images -a
docker tag reddit:latest nwton/otus-reddit:1.0
docker push nwton/otus-reddit:1.0
```

Работа с образом в локальной лаборатории
``` text
$ vagrant ssh docker-lab-16

docker run --name reddit -d -p 9292:9292 nwton/otus-reddit:1.0
docker ps

    http://10.10.10.16:9292/ - ссылка открывается
```

Исследование контейнера в локальной лаборатории
``` text
docker logs reddit -f
docker exec -it reddit bash
    ps aux
    killall5 1
docker ps -a

docker start reddit
docker stop reddit && docker rm reddit
docker run --name reddit --rm -it nwton/otus-reddit:1.0 bash
    ps aux
    exit

docker inspect nwton/otus-reddit:1.0
docker inspect nwton/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'
docker run --name reddit -d -p 9292:9292 nwton/otus-reddit:1.0
docker exec -it reddit bash
    mkdir /test1234
    touch /test1234/testfile
    rmdir /opt
    exit
docker diff reddit
docker stop reddit && docker rm reddit
docker run --name reddit --rm -it nwton/otus-reddit:1.0 bash
    ls /
```

## В процессе сделано:
- Создан новый проект docker в GCP и сконфигурирован gcloud
- Установил docker-machine - встроенный в докер инструмент
  для создания хостов и установки на них docker engine
- Запустил инстанс docker-host в GCP через docker-machine
- Сравнил запуск контейнера в ограниченном namespace и полном
- Создал образ с приложением и запустил контейнер в GCP
- Зарегистрировался в Docker Hub (https://hub.docker.com/)
- Загрузил в Docker Hub собранный образ приложения
- Проверил запуск контейнера в локальном docker
- Проинспектировал внутренности полученного контейнера

## Как запустить проект:
 - Запустить на любом хосте с docker
``` text
docker run --name reddit -d -p 9292:9292 nwton/otus-reddit:1.0
docker ps
```

## Как проверить работоспособность:
 - Перейти по ссылке http://localhost:9292 если docker запущен
   локально или http://_docker_host_ip_:9292/


# HW16. Docker образы. Микросервисы
## docker-3

## dockerfile linter + best practices

Hadolint
- https://github.com/hadolint/hadolint

Best Practices
- https://github.com/hadolint/hadolint/wiki
- https://docs.docker.com/engine/articles/dockerfile_best-practices/
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/

## Основное занятие

Получаем исходники
``` text
wget https://github.com/express42/reddit/archive/microservices.zip
unzip microservices.zip
mv reddit-microservices src
rm microservices.zip
```

Предварительно скачиваем образ mongo и начинаем собирать образы
``` text
docker pull mongo:latest

docker build -t nwton/post:1.0 ./post-py
docker build -t nwton/comment:1.0 ./comment
docker build -t nwton/ui:1.0 ./ui
```

Ошибки при сборке образов:
- post-ui
  - для thriftpy нужен gcc - это пакет build-base для alpine
- comment и ui
  - образ указан совсем древний ruby 2.2, сейчас минимальный 2.4
  - https://hub.docker.com/_/ruby/

Запускаем приложение
``` text
docker network create reddit

docker run -d --network=reddit \
    --network-alias=post_db --network-alias=comment_db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=post \
    nwton/post:1.0
docker run -d --network=reddit \
    --network-alias=comment \
    nwton/comment:1.0
docker run -d --network=reddit \
    -p 9292:9292 nwton/ui:1.0
```

## Дополнительное задание

Останавливаем все контейнеры и перезапускаем с другими алиасами,
для того чтобы всё работало - через опцию -e в ENV передаём новые
имена (аналог доменных имён и файла /etc/hosts)
- https://docs.docker.com/engine/reference/builder/#env
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#env

``` text
docker kill $(docker ps -q)
docker ps -a

docker run -d --network=reddit \
    --network-alias=extra_post_db --network-alias=extra_comment_db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=extra_post \
    -e POST_DATABASE_HOST=extra_post_db \
    nwton/post:1.0
docker run -d --network=reddit \
    --network-alias=extra_comment \
    -e COMMENT_DATABASE_HOST=extra_comment_db \
    nwton/comment:1.0
docker run -d --network=reddit \
    -e POST_SERVICE_HOST=extra_post \
    -e COMMENT_SERVICE_HOST=extra_comment \
    -p 9292:9292 nwton/ui:1.0
```

## Уменьшение размера образов

Сборка новых версий
``` text
$ docker images
REPOSITORY    TAG           IMAGE ID      CREATED             SIZE
nwton/ui      1.0           f8ae204cf9f8  37 minutes ago      921MB
nwton/comment 1.0           09614f0fe7cf  38 minutes ago      919MB
nwton/post    1.0           55b14653cc6b  About an hour ago   109MB
mongo         latest        785c65f61380  24 hours ago        412MB
ruby          2.4           91fee5436512  7 days ago          873MB
ruby          2.2           6c8e6f9667b2  14 months ago       715MB
python        3.6.0-alpine  cb178ebbf0f2  2 years ago         88.6MB

docker rm $(docker ps -a -q)
docker rmi $(docker images -q)

docker build -t nwton/comment:2.0 ./comment
docker build -t nwton/ui:2.0 ./ui
```

Удаляем старые образы и запускаем приложение, для проверки
что запустилось всё как нужно - удаляем все лишние образа
``` text
docker kill $(docker ps -q)

docker run -d --network=reddit \
    --network-alias=post_db --network-alias=comment_db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=post \
    nwton/post:1.0
docker run -d --network=reddit \
    --network-alias=comment \
    nwton/comment:2.0
docker run -d --network=reddit \
    -p 9292:9292 nwton/ui:2.0

docker rm $(docker ps -a -q)
docker rmi $(docker images -q)

$ docker images
REPOSITORY    TAG           IMAGE ID      CREATED             SIZE
nwton/ui      2.0           dc9e49ba32d4  2 minutes ago       403MB
nwton/comment 2.0           c41be17c4bd8  2 minutes ago       400MB
nwton/post    1.0           55b14653cc6b  About an hour ago   109MB
mongo         latest        785c65f61380  25 hours ago        412MB
ubuntu        16.04         13c9f1285025  2 weeks ago         119MB
python        3.6.0-alpine  cb178ebbf0f2  2 years ago         88.6MB
```
Результат - уменьшили больше чем в 2 раза (с 900 Мб до 400 Мб)

## Постоянное хранилище данных

Создаем docker volume и передаём через параметр `-v`
только в контейнер с mongodb
``` text
docker volume create reddit_db
docker kill $(docker ps -q)

docker run -d --network=reddit \
    --network-alias=post_db --network-alias=comment_db \
    -v reddit_db:/data/db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=post \
    nwton/post:1.0
docker run -d --network=reddit \
    --network-alias=comment \
    nwton/comment:2.0
docker run -d --network=reddit \
    -p 9292:9292 nwton/ui:2.0
```

## Дополнительное задание - уменьшение размера образа

Различные варианты
- http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/
- https://medium.com/@gdiener/how-to-build-a-smaller-docker-image-76779e18d48a

Образ на базе Alpine
- https://alpinelinux.org/

Alpine 3.10 = python 3.7
- https://pkgs.alpinelinux.org/packages?name=python3&branch=v3.10
Alpine 3.9 = python 3.6
- https://pkgs.alpinelinux.org/packages?name=python3&branch=v3.9

Сборка python:3.6.0-alpine собирается вот так:
- https://hub.docker.com/_/python/
- https://github.com/docker-library/python/
- https://github.com/docker-library/python/blob/master/3.6/alpine3.9/Dockerfile
- https://github.com/docker-library/python/blob/master/3.6/alpine3.10/Dockerfile

``` text
docker build -t nwton/post:3.0 ./post-py
docker build -t nwton/comment:3.0 ./comment
docker build -t nwton/ui:3.0 ./ui

docker kill $(docker ps -q)

docker run -d --network=reddit \
    --network-alias=post_db --network-alias=comment_db \
    -v reddit_db:/data/db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=post \
    nwton/post:3.0
docker run -d --network=reddit \
    --network-alias=comment \
    nwton/comment:3.0
docker run -d --network=reddit \
    -p 9292:9292 nwton/ui:3.0

docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
```

Итоговый результат:
``` text
$ docker images
REPOSITORY    TAG     IMAGE ID      CREATED         SIZE
nwton/ui      3.0     1d28e9ebfb53  3 minutes ago   71.3MB
nwton/comment 3.0     37dd08860408  3 minutes ago   68.5MB
nwton/post    3.0     3189c96bb718  3 minutes ago   68.5MB
mongo         latest  785c65f61380  27 hours ago    412MB
alpine        3.9     055936d39205  7 weeks ago     5.53MB
```

За счёт использования единого базового образа для всех
трёх компонентов - будет ещё лучше (скорость скачивания
и скорость сборки).

Для компонента post-py:
- базовый образ на базе python:3.6.0-alpine = 109MB
- переход на alpine:3.9 = 71.8MB
- очистка каталогов *cache* = 68.5MB

Для компонентов comments и ui:
- базовый образ на базе ruby:2.4 = 921MB
- переход на ubuntu:16.04 = 400MB
- переход на alpine:3.9 = 233MB
- правильный порядок и RUN в одну строку = 69.8MB
- очистка каталогов *cache* = 68.5MB


## В процессе сделано:
- Загрузил исходники проекта
- Создал Dockerfile для трёх модулей (post, comment, ui)
- Починил и оптимизировал Dockerfile для сборки образов
- Запустил приложение в контейнерах с настройками
  сети по-умолчанию
- Перезапустил приложение в контейнерах с изменёнными алиасами
  (дополнительное задание)
- Пересобрал образ ui на базе образа ubuntu 16.04
- Пересобрал образ comment на базе образа ubuntu 16.04
  (дополнительное задание), размер уменьшился с 900 до 400 Мб
- Создал Docker volume для контейнера с MongoDB и переключил
  на его использование (посты больше не теряются)
- Перевёл образы на alpine:3.9 и уменьшил размер до 70 Мб

## Как запустить проект:
 - Запустить на любом хосте с docker
``` text
docker pull mongo:latest
docker build -t nwton/post:3.0 ./post-py
docker build -t nwton/comment:3.0 ./comment
docker build -t nwton/ui:3.0 ./ui

docker network create reddit
docker volume create reddit_db

docker run -d --network=reddit \
    --network-alias=post_db --network-alias=comment_db \
    -v reddit_db:/data/db \
    mongo:latest
docker run -d --network=reddit \
    --network-alias=post \
    nwton/post:3.0
docker run -d --network=reddit \
    --network-alias=comment \
    nwton/comment:3.0
docker run -d --network=reddit \
    -p 9292:9292 nwton/ui:3.0
```

## Как проверить работоспособность:
 - Перейти по ссылке http://localhost:9292 если docker запущен
   локально или http://_docker_host_ip_:9292/


# HW17. Сетевое взаимодействие Docker контейнеров. Docker Compose. Тестирование образов
## docker-4
