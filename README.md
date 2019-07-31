# nwton_microservices
nwton microservices repository

# HW14. Технология контейнеризации. Введение в Docker.
## docker-1

## Локальная лаба для docker
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

### Использование лабы как docker-host
Для использования лабы в качестве собственного docker-host
надо добавить опцию `-H tcp://0.0.0.0` при запуске сервиса.
Это можно сделать через override для systemd unit
``` text
$ less /lib/systemd/system/docker.service

$ systemctl edit docker
$ service docker restart
$ service docker status

$ cat /etc/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0 -H fd:// --containerd=/run/containerd/containerd.sock
```

И после этого используем внутри WSL коннект к определённой лабе:
``` text
$ unset DOCKER_HOST
$ docker ps -a
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

$ export DOCKER_HOST="tcp://10.10.10.16:2375"
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND
8cf36aeac1a4        ubuntu              "bash"

$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ubuntu                  latest              4c108a37151f        2 weeks ago         64.2MB
hello-world             latest              fce289e99eb9        6 months ago        1.84kB
```

### Использование generic для docker-machine

Самая простая инсталляция инстанса docker-lab
(через vagrant сделан бутстрап только для python)
``` text
vagrant up docker-lab
docker-machine create \
    --driver generic \
    --generic-ip-address=10.10.10.42 \
    --generic-ssh-key .vagrant/machines/docker-lab/virtualbox/private_key \
    --generic-ssh-user=vagrant \
    docker-lab

eval $(docker-machine env docker-lab)
docker run hello-world
```

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

Запускаем старый docker-host и подключаемся
``` text
docker-machine ls
docker-machine status docker-host
docker-machine start docker-host
eval $(docker-machine env docker-host)

docker pull joffotron/docker-net-tools:latest
```

### Проверяем none network driver
Запуск
``` text
docker run -ti --rm --network none joffotron/docker-net-tools -c ifconfig
```
Доступен только loopback

### Проверяем host network driver
Запуск
``` text
docker run -ti --rm --network host joffotron/docker-net-tools -c ifconfig

docker run --network host -d nginx
docker run --network host -d nginx
docker ps
docker run --network host -d nginx
docker run --network host -d nginx
docker ps

docker kill $(docker ps -q)
```
Доступны все интерфейсы с хоста.

При одновременной запуске нескольких контейнеров
претендующих на один порт - приложение в повторных
контейнерах крэшится (т.к. не может получить биндинг)
и остаётся работать только самый первый контейнер.


### Проверяем docker network namespaces
Запуск
``` text
docker-machine ssh docker-host
    $ sudo ln -s /var/run/docker/netns /var/run/netns
    $ sudo ip netns

docker run -ti --rm --network none joffotron/docker-net-tools -c sh
    $ sudo ip netns
    $ sudo ip netns exec default ifconfig
    $ sudo ip netns exec 4b575d46533e ifconfig

docker run -ti --rm --network host joffotron/docker-net-tools -c sh
    $ sudo ip netns
```

Отдельный namespace для host network не создается.
Всё работает в default namespace.

### Проверяем bridge network driver
Запуск
``` text
docker network ls
docker network rm reddit
docker network create reddit --driver bridge

docker run -d --network=reddit mongo:latest
docker run -d --network=reddit nwton/post:3.0
docker run -d --network=reddit nwton/comment:3.0
docker run -d --network=reddit -p 9292:9292 nwton/ui:3.0
```

Присваиваем контейнерам DNS имена и алиасы
``` text
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
```

Запускаем в изолированных бриджах
``` text
docker kill $(docker ps -q)

docker network create back_net --subnet=10.0.2.0/24
docker network create front_net --subnet=10.0.1.0/24
docker network rm reddit
docker network ls

docker run -d --network=back_net \
    --network-alias=post_db --network-alias=comment_db \
    -v reddit_db:/data/db \
    --name mongo_db \
    mongo:latest
docker run -d --network=back_net \
    --network-alias=post \
    --name post \
    nwton/post:3.0
docker run -d --network=back_net \
    --network-alias=comment \
    --name comment \
    nwton/comment:3.0
docker run -d --network=front_net \
    --name ui \
    -p 9292:9292 nwton/ui:3.0

docker network connect front_net post
docker network connect front_net comment
```

Проверяем как это всё выглядит на docker-host
``` text
docker network ls
docker-machine ssh docker-host
    $ sudo apt-get update && sudo apt-get install bridge-utils
    $ ifconfig | grep br
    $ brctl show br-13153a014c2d
    $ brctl show br-904be4661a83
    $ sudo iptables -nL -t nat
    $ ps ax | grep docker-proxy
```

## Работа с docker-compose

Установка под WSL в локальный каталог
(или можно сделать `pip install docker-compose`)
- https://docs.docker.com/compose/install/#install-compose
``` text
curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o ~/bin/docker-compose

chmod +x ~/bin/docker-compose

$ docker-compose --version
docker-compose version 1.24.0, build 0aa59064
```

Запуск сборки через docker-compose
``` text
docker kill $(docker ps -q)

export USERNAME=nwton
docker-compose up -d
docker-compose ps

docker images
docker ps
```
Примечание: т.к. ранее уже собирались образы с тэгом
версии 3.0, а старые 1.0 были удалены, то будет пересборка.

Документация по ENV
- https://docs.docker.com/compose/environment-variables/#the-env-file

Пересборка с использованием env файла
``` text
docker kill $(docker ps -q)

unset USERNAME
docker-compose up -d
    # WARNING: Service "post_db" is using volume "/data/db" from
    # the previous container. Host mapping "reddit_db" has no effect.
    # Remove the existing containers (with `docker-compose rm post_db`)
    # to use the host volume

docker-compose stop post_db
docker-compose rm post_db
docker-compose up -d
docker-compose ps

docker volume ls
docker volume rm src_post_db
docker volume rm $(docker volume ls -q)

docker images
docker ps

docker-compose down
docker-compose up -d
```

Примечание: при параметризации нужно учитывать версию compose,
т.к. указание параметров различается в разных версиях, например,
указание name для volumes требует версию 2.1+ или 3.4+, но не
работает в 3.3, как использовано в темплейте из презентации
- https://docs.docker.com/compose/compose-file/#name
- https://docs.docker.com/compose/compose-file/#volumes
- https://docs.docker.com/compose/compose-file/#volume-configuration-reference
- https://docs.docker.com/storage/volumes/
- https://docs.docker.com/compose/compose-file/compose-file-v2/#name
- https://docs.docker.com/compose/compose-file/compose-file-v1/

Для смены префикса всех обьектов можно использовать
переменную COMPOSE_PROJECT_NAME, но при смене на лету
нужно помнить, что composer забудет про уже созданные
объекты с предыдущим префиксов и хвосты придётся удалять
уже вручную, лучше сделать заранее `docker-compose down`.

Смену имени контейнера с добавлением алиасов проводить
следует примерно так
``` text
docker-compose stop post_db
docker-compose rm post_db
...
docker-compose up -d mongo_db
```


## Дополнительное задание
Создайте docker-compose.override.yml для reddit проекта, который позволит
- Изменять код каждого из приложений, не выполняя сборку образа
- Запускать puma для руби приложений в дебаг режиме с двумя воркерами (флаги --debug и -w 2)

Документация:
- https://docs.docker.com/compose/extends/

Создан файл, в котором переопределены:
- для каждого каталога с приложением указанным в Dockerfile
  со значением APP_HOME=/app создан отдельный volume
- через command переопределен CMD указанный в Dockerfile

Проверяем, перезапускаем и снова проверяем
``` text
$ docker ps
$ docker volume ls

$ docker-compose up -d
Creating volume "stage_app_ui" with default driver
Creating volume "stage_app_comment" with default driver
Creating volume "stage_app_post" with default driver
Recreating stage_ui_1       ... done
Recreating stage_mongo_db_1 ... done
Recreating stage_post_1     ... done
Recreating stage_comment_1  ... done

$ docker volume rm $(docker volume ls -q)

$ docker volume ls
DRIVER              VOLUME NAME
local               c15d3ef0539b408753eae541b35c4bd7c41db7ca80668172380bffcfa9746d5f
local               reddit_db
local               stage_app_comment
local               stage_app_post
local               stage_app_ui

$ docker ps
CONTAINER ID        IMAGE               COMMAND
243841b997a7        nwton/comment:1.0   "puma --debug -w 2"
2e7cb0784f32        nwton/post:1.0      "python3 post_app.py"
387682279f99        nwton/ui:1.0        "puma --debug -w 2"
c64c6b741e2b        mongo:3.2           "docker-entrypoint.s…"
```


## В процессе сделано:
- Исследовал none, host и bridge драйвера сети в docker
- Изучил особенности работы с сетями в docker
- Установил docker-compose
- Создал docker-compose.yml для деплоя приложения
- Добавил параметризацию и поднял версию до 3.4:
  * порт приложения;
  * версии сервисов;
  * имя volume для базы данных
- Добавил работу компонентов в раздельных сетях
- Сменил имя и добавил пропущеный алиас для контейнера с
  базой данных (не работали комментарии)
- Добавил переменную COMPOSE_PROJECT_NAME для изменения префикса
- Добавил docker-compose.override.yml, который позволит
  * Изменять код каждого из приложений, не выполняя сборку образа
  * Запускать puma для руби приложений в дебаг режиме с двумя
    воркерами (флаги --debug и -w 2)

## Как запустить проект:
 - Запустить на любом хосте с docker
``` text
cd src
cp .env.example .env
docker-compose up -d
...
docker-compose down
```

## Как проверить работоспособность:
 - Перейти по ссылке http://localhost:9292 если docker запущен
   локально или http://_docker_host_ip_:9292/


# HW18. Технология непрерывной поставки ПО
## Только теория.


# HW19. Устройство Gitlab CI. Построение процесса непрерывной интеграции
## gitlab-ci-1

## Создание инстанса с Gitlab и начальная настройка

Создаём свой инстанс с Gitlab CI (2 vCPU + 7,5 Gb mem)
и открываем доступ, по
- https://docs.gitlab.com/ce/install/requirements.html
``` text
ssh-keygen -y -f ~/.ssh/appuser
cat ~/.ssh/appuser.pub

gcloud info | grep project
export GOOGLE_PROJECT=docker-245721

docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-2 \
    --google-zone europe-west1-b \
    --google-disk-size 100 \
    gitlab-ci

gcloud compute firewall-rules create \
    --allow tcp:80,tcp:443 \
    --target-tags=docker-machine \
    --description="gitlab-ci allow access to HTTP and HTTPS" \
    --direction=INGRESS \
    web-access-gitlab-ci

docker-machine ls

eval $(docker-machine env gitlab-ci)
docker run hello-world

eval $(docker-machine env --unset)
```

Используем omnibus установку Gitlab CI
- https://docs.gitlab.com/omnibus/README.html
- https://docs.gitlab.com/omnibus/docker/README.html
- https://docs.gitlab.com/omnibus/docker/README.html#install-gitlab-using-docker-compose

Предварительно создавать каталоги в /srv/gitlab
*не требуется, они будут созданы автоматически*
при деплое и/или старте контейнера.
``` bash
cd gitlab-ci
docker-compose up -d
```

Заходим на сервер и проверяем
``` bash
docker-machine ssh gitlab-ci
```

Переходим в админку http://35.187.101.147/ и задаем пароль.
Затем отключаем регистрацию новых пользователей:
- Admin Area
  - Settings
    - Sign-up restrictions
      - Sign-up enabled

Создаем группу: homework
Создаем проект: example

## Работа по заданию

Добавляем remotes в локальный проект
``` text
git checkout -b gitlab-ci-1
git remote add gitlab http://35.187.101.147/homework/example.git
git push gitlab gitlab-ci-1
```

Создаем файл .gitlab-ci.yml и отправляем
``` text
git add .gitlab-ci.yml
git commit -m 'add pipeline definition'
git push gitlab gitlab-ci-1
```

Получаем токен для runner
- http://35.187.101.147/homework/example/-/settings/ci_cd
- https://docs.gitlab.com/runner/install/

``` text
Set up a specific Runner manually
Install GitLab Runner
Specify the following URL during the Runner setup: http://35.187.101.147/
Use the following registration token during setup: CcV-r_GcnERyzN1q_uZD
Start the Runner!
```

Устанавливаем контейнер с runner и регистрируем:
``` text
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

docker exec -it gitlab-runner \
  gitlab-runner register --run-untagged --locked=false
```

Пример регистрации:
``` text
$ docker exec -it gitlab-runner \
>   gitlab-runner register --run-untagged --locked=false
Runtime platform
arch=amd64 os=linux pid=13 revision=0e5417a3 version=12.0.1
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://35.187.101.147/
Please enter the gitlab-ci token for this runner:
CcV-r_GcnERyzN1q_uZD
Please enter the gitlab-ci description for this runner:
[45eab39a6476]: my-runner
Please enter the gitlab-ci tags for this runner (comma separated):
linux,xenial,ubuntu,docker
Registering runner... succeeded                     runner=CcV-r_Gc
Please enter the executor: parallels, ssh, docker+machine, kubernetes, docker-ssh, shell, virtualbox, docker-ssh+machine, docker:
docker
Please enter the default Docker image (e.g. ruby:2.6):
alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

Добавим исходный код reddit в репозиторий
``` text
git clone https://github.com/express42/reddit.git && rm -rf ./reddit/.git
git add reddit/
git commit -m "Add reddit app"
git push gitlab gitlab-ci-1
```

После этого правим код .gitlab-ci.yml и пушим в наш Gitlab.


## В процессе сделано:
- Создал инстанс в GCP с docker через docker-machine
- Разрешил веб-доступ к инстансу
- Установил Gitlab через omnibus
- Создал тестовый проект в gitlab и подключил как remotes
- Создал и подключил Gitlab Runner к проекту
- Поэтапно улучшал качество .gitlab-ci.yml
- Кратко ознакомился с динамическим окружением

## Как запустить проект:
- через docker-machine или ENV переменную подключиться
  к удаленному хосту с docker и запустить
``` bash
cd gitlab-ci
docker-compose up -d
```

## Как проверить работоспособность:
- Перейти по ссылке http://_IP_gitlab_ci_host_/


# HW20. Введение в мониторинг. Модели и принципы работы систем мониторинга
## monitoring-1

## Начальная настройка окружения

Создадим правило фаервола для Prometheus и Puma.
``` text
gcloud compute firewall-rules create prometheus-default --allow tcp:9090
gcloud compute firewall-rules create puma-default --allow tcp:9292
```

Создадим Docker хост в GCE и настроим локальное окружение на работу с ним.
``` text
gcloud info | grep project
export GOOGLE_PROJECT=docker-245721

# create docker host
docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-2 \
    --google-zone europe-west1-b \
    docker-host

# Настроить докер клиент на удалённый докер
eval $(docker-machine env docker-host)

# Переключение на локальный докер
eval $(docker-machine env --unset)

# Просмотр IP удалённого докера
$ docker-machine ip docker-host

# Удаление докер хоста
$ docker-machine rm docker-host
```

## Работа по ДЗ

Систему мониторинга Prometheus будем запускать внутри Docker контейнера.
Для начального знакомства воспользуемся готовым образом с DockerHub.
``` text
docker run --rm -p 9090:9090 -d --name prometheus  prom/ prometheus:v2.1.0

docker run --rm -p 9090:9090 -d --name prometheus  prom/prometheus

docker-machine ip docker-host

docker stop prometheus
``` text

Пример метрик:
``` text
prometheus_build_info{branch="HEAD",goversion="go1.12.7",instance="localhost:9090",job="prometheus",revision="e5b22494857deca4b806f74f6e3a6ee30c251763",version="2.11.1"}

up{instance="localhost:9090",job="prometheus"}
```

Собираем собственный docker образ для prometheus
``` text
export USER_NAME=username
cd monitoring/prometheus/
docker build -t $USER_NAME/prometheus .
cd -
```
или в одну строчку без смены рабочего каталога:
``` text
docker build -t $USER_NAME/prometheus ./monitoring/prometheus
```

Собираем собственные docker образы сервисов
``` text
for i in ui post-py comment
do
  cd src/$i; bash docker_build.sh; cd -
done
```

Запуск и остановка наших сервисов
``` text
cd docker
docker-compose up -d

docker-compose stop post
docker-compose start post

docker build -t $USER_NAME/prometheus ../monitoring/prometheus

docker-compose down
docker-compose up -d

cd -
```

Отправка собранных образов на Docker Hub
``` text
docker login

docker push $USER_NAME/ui
docker push $USER_NAME/comment
docker push $USER_NAME/post
docker push $USER_NAME/prometheus
```

Удаляем временный хост для docker
``` text
docker-machine rm docker-host
```

## Дополнительные задания

Экспортер для MongoDB
- Проект [dcu/mongodb_exporter](https://github.com/dcu/mongodb_exporter)
  не самый лучший вариант, т.к. у него есть проблемы с поддержкой (не обновляется).

Blackbox prober exporter https://prometheus.io
- https://github.com/prometheus/blackbox_exporter

Документация по makefile
- https://www.ibm.com/developerworks/ru/library/l-debugmake/index.html
- https://habr.com/ru/post/132524/
- http://rus-linux.net/nlib.php?name=/MyLDP/algol/gnu_make/gnu_make_3-79_russian_manual.html

## В процессе сделано:
- Добавил правила фаервола GCP для доступа к prometheus и reddit-puma
- Создал инстанс в GCP с docker через docker-machine
- Создал собственные docker образы:
  - для prometheus
  - для microservices из src
- изменён docker-compose файл
  - удалены build (сборка через docker_buld.sh)
  - используемые версии images бампнуты до latest
- собранные docker образы выложены на [Docker Hub](https://hub.docker.com/u/nwton/)
  - [nwton/ui](https://hub.docker.com/r/nwton/ui)
  - [nwton/comment](https://hub.docker.com/r/nwton/comment)
  - [nwton/post](https://hub.docker.com/r/nwton/post)
  - [nwton/prometheus](https://hub.docker.com/r/nwton/prometheus)

## Как запустить проект:
- через docker-machine или ENV переменную подключиться
  к удаленному хосту с docker и запустить
``` bash
cd docker
docker-compose up -d
```

## Как проверить работоспособность:
- Перейти по ссылке http://_IP_docker_host_:9090/


# HW21. Мониторинг приложения и инфраструктуры
## monitoring-2

## Начальная настройка окружения

Создадим Docker хост в GCE и настроим локальное окружение на работу с ним.
Идентично HW20.

## Работа по ДЗ
Оставим описание приложений в docker-compose.yml, а мониторинг выделим в отдельный файл docker-composemonitoring.yml.

``` bash
# Для запуска приложений:
docker-compose up -d
# Для запуска мониторинга:
docker-compose -f docker-compose-monitoring.yml up -d
```

### cAdvisor
Добавляем [cAdvisor](https://github.com/google/cadvisor)
``` bash
gcloud compute firewall-rules create cadvisor-default --allow tcp:8080

docker build -t $USER_NAME/prometheus ../monitoring/prometheus

docker-compose -f docker-compose-monitoring.yml down
docker-compose -f docker-compose-monitoring.yml up -d
```

### Grafana
Добавляем [Grafana](https://grafana.com/)
``` bash
gcloud compute firewall-rules create grafana-default --allow tcp:3000

docker-compose -f docker-compose-monitoring.yml up -d grafana
docker-compose -f docker-compose-monitoring.yml stop grafana
docker-compose -f docker-compose-monitoring.yml up -d grafana
```

Настройка Grafana:
- Делаем Add datasource
  - Name: Prometheus Server
  - Type: Prometheus
  - URL: http://prometheus:9090
  - Access: Proxy
- Добавляем Dashboard
  - [Поиск](https://grafana.com/grafana/dashboards)
  - [Поиск prometheus+grafana](https://grafana.com/grafana/dashboards?dataSource=prometheus&search=docker)
  - [Docker and system monitoring by Thibaut Mottet](https://grafana.com/grafana/dashboards/893)
  - [Download JSON](https://grafana.com/api/dashboards/893/revisions/5/download)

Загружаем Dashboard
``` bash
mkdir -p ../monitoring/grafana/dashboards
curl \
    -L https://grafana.com/api/dashboards/893/revisions/5/download \
    -o ../monitoring/grafana/dashboards/DockerMonitoring.json
```

После этого Import (http://IP:3000/dashboard/import)
- либо ручное копирование JSON в интерфейс
- либо через указание ID = 893

Добавляем собственные метрики:
- [сервис UI](https://github.com/express42/reddit/commit/e443f6ab4dcf25f343f2a50c01916d750fc2d096)
- [сервис Post](https://github.com/express42/reddit/commit/d8a0316c36723abcfde367527bad182a8e5d9cf2)

``` bash
docker build -t $USER_NAME/prometheus ../monitoring/prometheus

docker-compose -f docker-compose-monitoring.yml stop prometheus
docker-compose -f docker-compose-monitoring.yml up -d prometheus
```

### Alertmanager

Для отправки нотификаций в слак канал потребуется создать СВОЙ
[Incoming Webhook](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks?page=1)

После тестов - там же меняем ссылку и/или отключаем.

``` bash
gcloud compute firewall-rules create alertmanager-default --allow tcp:9093

docker build -t $USER_NAME/alertmanager ../monitoring/alertmanager
docker-compose -f docker-compose-monitoring.yml up -d alertmanager

docker build -t $USER_NAME/prometheus ../monitoring/prometheus

docker-compose -f docker-compose-monitoring.yml stop prometheus
docker-compose -f docker-compose-monitoring.yml up -d prometheus

docker-compose stop post
docker-compose up -d post

docker-compose -f docker-compose-monitoring.yml down
docker-compose -f docker-compose-monitoring.yml up -d
```

### Завершение
``` bash
docker login

docker push $USER_NAME/ui
docker push $USER_NAME/comment
docker push $USER_NAME/post
docker push $USER_NAME/prometheus
docker push $USER_NAME/alertmanager

docker-machine rm docker-host
```

## В процессе сделано:
- Создал инстанс в GCP с docker через docker-machine
- Из файла docker-compose мониторинг выделен в отдельный файл
- Добавил в мониторинг cAdvisory
- Добавил в мониторинг Grafana
  - Добавил официальный дашбоард **Docker and system monitoring**
  - Создал дашборд **UI_Service_Monitoring.json**
  - Создал дашборд **Business_Logic_Monitoring.json**
- Добавил alertmanager с отправкой в slack
- собранные docker образы выложены на [Docker Hub](https://hub.docker.com/u/nwton/)
  - [nwton/ui](https://hub.docker.com/r/nwton/ui)
  - [nwton/comment](https://hub.docker.com/r/nwton/comment)
  - [nwton/post](https://hub.docker.com/r/nwton/post)
  - [nwton/prometheus](https://hub.docker.com/r/nwton/prometheus)
  - [nwton/alertmanager](https://hub.docker.com/r/nwton/alertmanager)

## Как запустить проект:
- через docker-machine или ENV переменную подключиться
  к удаленному хосту с docker и запустить
``` bash
cd docker
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d
```

## Как проверить работоспособность:
- Перейти по ссылке http://_IP_docker_host_:9090/


# HW22. Применение инструментов для обработки лог данных
## Только теория.


# HW23. Применение системы логирования в инфраструктуре на основе Docker.
## logging-1

## Начальная настройка окружения

Создадим Docker хост в GCE и настроим локальное окружение на работу с ним.
Идентично HW20.

Дополнение: можно открывать необходимые порты для конкретного докер
хоста сразу при его создании, с использованием google-open-port:
``` bash
$ docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-oscloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-open-port 5601/tcp \
    --google-open-port 9292/tcp \
    --google-open-port 9411/tcp \
    logging

$ eval $(docker-machine env logging)
```

## Работа по ДЗ

### Обновляем исходники
Обновляем исходный код [на версию](https://github.com/express42/reddit/tree/logging)
``` bash
rm -rf src/
wget https://github.com/express42/reddit/archive/logging.zip
unzip logging.zip
mv reddit-logging src
rm logging.zip
git add src
```

Восстанавливаем разрушенное:
``` bash
git checkout HEAD src/comment/Dockerfile
git checkout HEAD src/post-py/Dockerfile
git checkout HEAD src/ui/Dockerfile
```

Собираем собственные docker образы сервисов
с новыми тэгами logging
``` text
for i in ui post-py comment
do
  cd src/$i; bash docker_build.sh; cd -
done
```

Запускаемся
``` bash
cd docker
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d
```

### Дебажим запуск контейнера

``` text
$ docker ps -a
CREATED              STATUS                      PORTS                    NAMES
About a minute ago   Up About a minute                                    stage_post_1
About a minute ago   Up About a minute           0.0.0.0:9292->9292/tcp   stage_ui_1
About a minute ago   Exited (1) 12 seconds ago                            stage_comment_1
comment:logging        "puma"                   About a minute ago   Exited (1) 12 seconds ago                            stage_comment_1

$ docker run -it nwton/comment:logging sh
/app # puma
Puma starting in single mode...
* Version 3.10.0 (ruby 2.5.5-p157), codename: Russell's Teapot
* Min threads: 0, max threads: 16
* Environment: development
! Unable to load application: TZInfo::DataSourceNotFound: No source of timezone data could be found.
Please refer to http://tzinfo.github.io/datasourcenotfound for help resolving this error.
Traceback (most recent call last):
        39: from /usr/bin/puma:23:in `<main>'
        38: from /usr/bin/puma:23:in `load'
        37: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/bin/puma:10:in `<top (required)>'
        36: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/cli.rb:77:in `run'
        35: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/launcher.rb:183:in `run'
        34: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/single.rb:87:in `run'
        33: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/runner.rb:138:in `load_and_bind'
        32: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/configuration.rb:243:in `app'
        31: from /usr/lib/ruby/gems/2.5.0/gems/puma-3.10.0/lib/puma/configuration.rb:314:in `load_rackup'
        30: from /usr/lib/ruby/gems/2.5.0/gems/rack-2.0.5/lib/rack/builder.rb:40:in `parse_file'
        29: from /usr/lib/ruby/gems/2.5.0/gems/rack-2.0.5/lib/rack/builder.rb:49:in `new_from_string'
        28: from /usr/lib/ruby/gems/2.5.0/gems/rack-2.0.5/lib/rack/builder.rb:49:in `eval'
        27: from config.ru:in `<main>'
        26: from config.ru:in `new'
        25: from /usr/lib/ruby/gems/2.5.0/gems/rack-2.0.5/lib/rack/builder.rb:55:in `initialize'
        24: from /usr/lib/ruby/gems/2.5.0/gems/rack-2.0.5/lib/rack/builder.rb:55:in `instance_eval'
        23: from config.ru:1:in `block in <main>'
        22: from /usr/lib/ruby/2.5.0/rubygems/core_ext/kernel_require.rb:59:in `require'
        21: from /usr/lib/ruby/2.5.0/rubygems/core_ext/kernel_require.rb:59:in `require'
        20: from /app/comment_app.rb:49:in `<top (required)>'
        19: from /app/comment_app.rb:49:in `new'
        18: from /usr/lib/ruby/gems/2.5.0/gems/rufus-scheduler-3.4.2/lib/rufus/scheduler.rb:89:in `initialize'
        17: from /usr/lib/ruby/gems/2.5.0/gems/rufus-scheduler-3.4.2/lib/rufus/scheduler.rb:543:in `start'
        16: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:265:in `now'
        15: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:19:in `now'
        14: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:19:in `new'
        13: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:303:in `initialize'
        12: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:275:in `get_tzone'
        11: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:147:in `get_tzone'
        10: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:171:in `local_tzone'
         9: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:626:in `determine_local_tzone'
         8: from /usr/lib/ruby/gems/2.5.0/gems/et-orbi-1.0.8/lib/et-orbi.rb:687:in `determine_local_tzones'
         7: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/timezone.rb:124:in `all'
         6: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/timezone.rb:130:in `all_identifiers'
         5: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/timezone.rb:661:in `data_source'
         4: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/data_source.rb:39:in `get'
         3: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/data_source.rb:39:in `synchronize'
         2: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/data_source.rb:40:in `block in get'
         1: from /usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/data_source.rb:179:in `create_default_data_source'
/usr/lib/ruby/gems/2.5.0/gems/tzinfo-1.2.3/lib/tzinfo/data_source.rb:182:in `rescue in create_default_data_source': No source of timezone data could be found. (TZInfo::DataSourceNotFound)
Please refer to http://tzinfo.github.io/datasourcenotfound for help resolving this error.
/app #
```

### Обновляем исходники до ветки microservices
Обновляем исходный код [на версию microservices](https://github.com/express42/reddit/tree/microservices)
``` bash
rm -rf src/
wget https://github.com/express42/reddit/archive/microservices.zip
unzip microservices.zip
mv reddit-microservices src
rm microservices.zip
git add src
```

Восстанавливаем разрушенное и файлы сборки с правильными тэгами:
``` bash
git checkout HEAD src/comment/Dockerfile
git checkout HEAD src/post-py/Dockerfile
git checkout HEAD src/ui/Dockerfile

git checkout HEAD src/comment/docker_build.sh
git checkout HEAD src/post-py/docker_build.sh
git checkout HEAD src/ui/docker_build.sh
```

Собираем собственные docker образы сервисов
с новыми тэгами logging
``` text
for i in ui post-py comment
do
  cd src/$i; bash docker_build.sh; cd -
done
```

Запускаемся (старое будет прибито)
``` bash
cd docker
docker ps -a
docker-compose up -d --remove-orphans
docker-compose -f docker-compose-monitoring.yml up -d
```

### Добавляем EFK

В этом ДЗ мы рассмотрим пример системы централизованного логирования
на примере Elastic стека (ранее известного как ELK), который включает
в себя 3 осовных компонента:
- ElasticSearch (TSDB и поисковый движок для хранения данных)
- Logstash (для агрегации и трансформации данных)
- Kibana (для визуализации)

Однако для агрегации логов вместо Logstash мы будем использовать Fluentd, таким образом получая еще одно популярное сочетание этих инструментов, получившее название EFK.

Открываем порты и запускаем
``` bash
gcloud compute firewall-rules create kibana-default --allow tcp:5601

docker build -t $USER_NAME/fluentd ../logging/fluentd

docker-compose -f docker-compose-logging.yml up -d

docker-compose -f docker-compose-logging.yml up -d fluentd
```

### Тестирование логов

На время тестирования логирования можно мониторинг остановить,
иначе каждые 5 секунд идёт обращение к /metrics и /healthcheck
и дальше смотрил логи онлайн
``` text
docker-compose -f docker-compose-monitoring.yml stop

docker-compose logs -f post
```

Переключаемся на сбор логов [для docker через драйвер ﬂuentd](https://docs.docker.com/config/containers/logging/fluentd/)
``` text
docker-compose -f docker-compose-logging.yml up -d

docker-compose down
docker-compose up -d
```

И настраиваем http://KIBANA:5601/ на fluentd-*

Пересборка fluentd:
``` bash
docker build -t $USER_NAME/fluentd ../logging/fluentd
docker-compose -f docker-compose-logging.yml up -d fluentd
```

Или полный перезапуск логгирования
``` bash
docker-compose -f docker-compose-logging.yml down
docker-compose -f docker-compose-logging.yml up -d
```

Открываем порты для zipkin и делаем полный перезапуск
``` bash
gcloud compute firewall-rules create zipkin-default --allow tcp:9411

docker-compose -f docker-compose-logging.yml -f docker-compose.yml down
docker-compose -f docker-compose-logging.yml -f docker-compose.yml up -d
```

### Завершение
``` bash
docker login

docker push $USER_NAME/ui
docker push $USER_NAME/comment
docker push $USER_NAME/post

docker push $USER_NAME/prometheus
docker push $USER_NAME/alertmanager

docker push $USER_NAME/fluentd

docker-machine rm docker-host
```

## В процессе сделано:
- Создал инстанс в GCP с docker через docker-machine
- Добавлен compose файл для развертывания EFK
- Для сервисов ui и post настроено логгирование во fluentd
- Собрал собственный fluentd контейнер
- Изучил kibana
- Добавил поддержку zipkin
- собранные docker образы выложены на [Docker Hub](https://hub.docker.com/u/nwton/)
  - [nwton/ui](https://hub.docker.com/r/nwton/ui)
  - [nwton/comment](https://hub.docker.com/r/nwton/comment)
  - [nwton/post](https://hub.docker.com/r/nwton/post)
  - [nwton/prometheus](https://hub.docker.com/r/nwton/prometheus)
  - [nwton/alertmanager](https://hub.docker.com/r/nwton/alertmanager)
  - [nwton/fluentd](https://hub.docker.com/r/nwton/fluentd)

## Как запустить проект:
- через docker-machine или ENV переменную подключиться
  к удаленному хосту с docker и запустить
``` bash
cd docker
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d
docker-compose -f docker-compose-logging.yml up -d
```

## Как проверить работоспособность:
- Перейти по ссылке http://_IP_docker_host_:9090/

# HW24. Контейнерная оркестрация.
## Только теория.


# KUBERNETES

# HW25. Введение в Kubernetes.
## kubernetes-1

## Kubernetes The Hard Way

В качестве домашнего задания предлагается пройти курс
[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
разработанный инженером Google Kelsey Hightower.

Туториал представляет собой:
- Пошаговое руководство по ручной инсталляции основных
  компонентов Kubernetes кластера;
- Краткое описание необходимых действий и объектов

## Работа по ДЗ

[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/01-prerequisites.md>

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-client-tools.md>

Linux tools from <https://pkg.cfssl.org/>
```bash
cd ~/bin/
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl cfssljson

wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl
chmod +x kubectl

kubectl version --client

cd -
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md>

```bash
###
gcloud compute networks create \
  kubernetes-the-hard-way --subnet-mode custom

Created [https://www.googleapis.com/compute/v1/projects/docker-245721/global/networks/kubernetes-the-hard-way].
NAME                     SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
kubernetes-the-hard-way  CUSTOM       REGIONAL

Instances on this network will not be reachable until firewall rules
are created. As an example, you can allow all internal traffic between
instances as well as SSH, RDP, and ICMP by running:
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network kubernetes-the-hard-way --allow tcp,udp,icmp --source-ranges <IP_RANGE>
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network kubernetes-the-hard-way --allow tcp:22,tcp:3389,icmp

###
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24

gcloud compute firewall-rules create \
  kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16

gcloud compute firewall-rules create \
  kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0

gcloud compute firewall-rules list \
  --filter="network:kubernetes-the-hard-way"

gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)

gcloud compute addresses list \
  --filter="name=('kubernetes-the-hard-way')"
```

Kubernetes Controllers
```bash
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

Kubernetes Workers
```bash
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

Verification and SSH
```bash
gcloud compute instances list

gcloud compute ssh controller-0

Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
```

- https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

Certificate Authority
```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Client and Server Certificates
```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

The Kubelet Client Certificates
```bash
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

The Controller Manager Client Certificate
```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

The Kube Proxy Client Certificate
```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

The Scheduler Client Certificate
```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

The Kubernetes API Server Certificate
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

The Service Account Key Pair
```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

Distribute the Client and Server Certificates
```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done

for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md>

The kubelet Kubernetes Configuration File
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${instance}.kubeconfig
done
```

The kube-proxy Kubernetes Configuration File
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

The kube-controller-manager Kubernetes Configuration File
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

The kube-scheduler Kubernetes Configuration File
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

The admin Kubernetes Configuration File
```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

Distribute the Kubernetes Configuration Files
```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done

for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/06-data-encryption-keys.md>

Generating the Data Encryption Config and Key
```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md>

Running Commands in Parallel with tmux

**tmux** can be used to run commands on multiple compute instances at the
same time. Labs in this tutorial may require running the same commands
across multiple compute instances, in those cases consider using tmux and
splitting a window into multiple panes with **synchronize-panes** enabled
to speed up the provisioning process.

Enable **synchronize-panes**: **ctrl+b** then **shift :**.
Then type set **synchronize-panes on** at the prompt.
To disable synchronization: **set synchronize-panes off**.

Bootstrapping an etcd Cluster Member
```bash
# The commands in this lab must be run on each controller instance:
# controller-0, controller-1, and controller-2. Login to each controller
# instance using the gcloud command. Example:
# gcloud compute ssh controller-0

wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"

{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}

{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}

INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)

ETCD_NAME=$(hostname -s)

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

Verification
```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md>

Bootstrapping the Kubernetes Control Plane
```bash
# The commands in this lab must be run on each controller instance:
# controller-0, controller-1, and controller-2. Login to each controller
# instance using the gcloud command. Example:
# gcloud compute ssh controller-0

sudo mkdir -p /etc/kubernetes/config

wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl"

{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}

{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}

INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

Enable HTTP Health Checks
```bash
sudo apt-get install -y nginx

cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF

{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}

sudo systemctl restart nginx
sudo systemctl enable nginx
```

Verification
```bash
kubectl get componentstatuses --kubeconfig admin.kubeconfig

curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

RBAC for Kubelet Authorization
(достаточно выполнить только на одной ноде)
```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

The Kubernetes Frontend Load Balancer
(запускаем от себя)
```bash
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-health-check \
    --network kubernetes-the-hard-way \
    --source-ranges 209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 \
    --allow tcp

  gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

  gcloud compute target-pools add-instances kubernetes-target-pool \
   --instances controller-0,controller-1,controller-2

  gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${KUBERNETES_PUBLIC_ADDRESS} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool
}
```

Verify
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
{
  "major": "1",
  "minor": "12",
  "gitVersion": "v1.12.0",
  "gitCommit": "0ed33881dc4355495f623c6f22e7dd0b7632b7c0",
  "gitTreeState": "clean",
  "buildDate": "2018-09-27T16:55:41Z",
  "goVersion": "go1.10.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md>

Bootstrapping the Kubernetes Worker Nodes
```bash
# The commands in this lab must be run on each worker instance:
# worker-0, worker-1, and worker-2. Login to each worker instance
# using the gcloud command. Example:
# gcloud compute ssh worker-0

{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}

wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.12.0/crictl-v1.12.0-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-the-hard-way/runsc-50c283b9f56bb7200938d9e207355f05f79f0d17 \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc5/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.0-rc.0/containerd-1.2.0-rc.0.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubelet

sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

{
  sudo mv runsc-50c283b9f56bb7200938d9e207355f05f79f0d17 runsc
  sudo mv runc.amd64 runc
  chmod +x kubectl kube-proxy kubelet runc runsc
  sudo mv kubectl kube-proxy kubelet runc runsc /usr/local/bin/
  sudo tar -xvf crictl-v1.12.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
  sudo tar -xvf containerd-1.2.0-rc.0.linux-amd64.tar.gz -C /
}

POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)

cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF

sudo mkdir -p /etc/containerd/

cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
    [plugins.cri.containerd.gvisor]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF

cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF

{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}

cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF

cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

Verification
```bash
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"

NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   42s   v1.12.0
worker-1   Ready    <none>   42s   v1.12.0
worker-2   Ready    <none>   42s   v1.12.0
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md>

Configuring kubectl for Remote Access
```bash
# Run the commands in this lab from the same directory used to
# generate the admin client certificates.

{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}

kubectl get componentstatuses
# NAME                 STATUS    MESSAGE             ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-2               Healthy   {"health":"true"}
# etcd-0               Healthy   {"health":"true"}
# etcd-1               Healthy   {"health":"true"}

kubectl get nodes
# NAME       STATUS   ROLES    AGE     VERSION
# worker-0   Ready    <none>   4m56s   v1.12.0
# worker-1   Ready    <none>   4m56s   v1.12.0
# worker-2   Ready    <none>   4m56s   v1.12.0
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md>

Provisioning Pod Network Routes
```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done

for i in 0 1 2; do
  gcloud compute routes create kubernetes-route-10-200-${i}-0-24 \
    --network kubernetes-the-hard-way \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done

gcloud compute routes list --filter "network: kubernetes-the-hard-way"
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md>

Deploying the DNS Cluster Add-on
```bash
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml

kubectl get pods -l k8s-app=kube-dns -n kube-system
```

Verification
```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
# kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be
# removed in a future version. Use kubectl create instead.
# deployment.apps/busybox created

kubectl get pods -l run=busybox

POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")

kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

- <https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md>

Smoke Test
In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

```bash
## Data Encryption
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"

gcloud compute ssh controller-0 \
  --command "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"

## Deployments
kubectl run nginx --image=nginx
kubectl get pods -l run=nginx

## Port Forwarding
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:80
# different terminal
# curl --head http://127.0.0.1:8080

## Logs
kubectl logs $POD_NAME

## Exec
kubectl exec -ti $POD_NAME -- nginx -v

## Services
kubectl expose deployment nginx --port 80 --type NodePort

NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')

gcloud compute firewall-rules create kubernetes-the-hard-way-allow-nginx-service \
  --allow=tcp:${NODE_PORT} \
  --network kubernetes-the-hard-way

EXTERNAL_IP=$(gcloud compute instances describe worker-0 \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

Untrusted Workloads
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: untrusted
  annotations:
    io.kubernetes.cri.untrusted-workload: "true"
spec:
  containers:
    - name: webserver
      image: gcr.io/hightowerlabs/helloworld:2.0.0
EOF

kubectl get pods -o wide

INSTANCE_NAME=$(kubectl get pod untrusted --output=jsonpath='{.spec.nodeName}')

gcloud compute ssh ${INSTANCE_NAME}
# run in terminal worker-1
# sudo runsc --root  /run/containerd/runsc/k8s.io list

# POD_ID=$(sudo crictl -r unix:///var/run/containerd/containerd.sock \
#   pods --name untrusted -q)
# CONTAINER_ID=$(sudo crictl -r unix:///var/run/containerd/containerd.sock \
#   ps -p ${POD_ID} -q)
# sudo runsc --root /run/containerd/runsc/k8s.io ps ${CONTAINER_ID}
```

## Запуск своих подов
```bash
$ cd ../reddit/

$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
busybox-bd8fb7cbd-zr6k8   1/1     Running   0          22m
nginx-dbddb74b8-6drcn     1/1     Running   0          18m
untrusted                 1/1     Running   0          5m30s

kubectl apply -f comment-deployment.yml
kubectl apply -f mongo-deployment.yml
kubectl apply -f post-deployment.yml
kubectl apply -f ui-deployment.yml

$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
busybox-bd8fb7cbd-zr6k8              1/1     Running   0          26m
comment-deployment-868496875-qcp65   1/1     Running   0          2m26s
mongo-deployment-67f58fb89-5689s     1/1     Running   0          2m19s
nginx-dbddb74b8-6drcn                1/1     Running   0          22m
post-deployment-5585b48cf9-ds4nk     1/1     Running   0          2m11s
ui-deployment-5bc65bd775-gkhzx       1/1     Running   0          2m6s
untrusted                            1/1     Running   0          9m12s
```

## Очистка

[Cleaning Up](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/14-cleanup.md)
```bash
gcloud -q compute instances delete \
  controller-0 controller-1 controller-2 \
  worker-0 worker-1 worker-2

{
  gcloud -q compute forwarding-rules delete kubernetes-forwarding-rule \
    --region $(gcloud config get-value compute/region)

  gcloud -q compute target-pools delete kubernetes-target-pool

  gcloud -q compute http-health-checks delete kubernetes

  gcloud -q compute addresses delete kubernetes-the-hard-way
}

gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external \
  kubernetes-the-hard-way-allow-health-check

{
  gcloud -q compute routes delete \
    kubernetes-route-10-200-0-0-24 \
    kubernetes-route-10-200-1-0-24 \
    kubernetes-route-10-200-2-0-24

  gcloud -q compute networks subnets delete kubernetes

  gcloud -q compute networks delete kubernetes-the-hard-way
}
```

## В процессе сделано:
- Развернул кластер Kubernetes по курсу
  [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- Результаты сохранил в папке **the_hard_way**
- Удалил кластер

## Как запустить проект:
- Поднять кластер, затем развернуть на нем приложения

```bash
cd kubernetes/reddit/
kubectl apply -f comment-deployment.yml
kubectl apply -f mongo-deployment.yml
kubectl apply -f post-deployment.yml
kubectl apply -f ui-deployment.yml
```

## Как проверить работоспособность:
- Проверить что кластер создается и поды запускаются

```bash
$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
comment-deployment-868496875-qcp65   1/1     Running   0          2m26s
mongo-deployment-67f58fb89-5689s     1/1     Running   0          2m19s
post-deployment-5585b48cf9-ds4nk     1/1     Running   0          2m11s
ui-deployment-5bc65bd775-gkhzx       1/1     Running   0          2m6s
```


# HW26. Основные модели безопасности и контроллеры в Kubernetes
## kubernetes-2

## Локальное окружение

### kubectl
kubectl для WSL
(по методу из kubernetes-the-hard-way)
- https://kubernetes.io/docs/tasks/tools/install-kubectl/
```bash
cd ~/bin/
wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl
chmod +x kubectl
kubectl version --client
cd -
```

Более свежие версии
```bash
cd ~/bin/
curl -L https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl -o kubectl-v1.12.0

curl -L https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl -o kubectl-v1.15.0

curl -L https://storage.googleapis.com/kubernetes-release/release/v1.15.1/bin/linux/amd64/kubectl -o kubectl-v1.15.1

rm kubectl
ln -s kubectl-v1.15.1 kubectl
chmod +x kubectl

kubectl version --client
cd -
```

### minicube
minicube для Windows
- https://kubernetes.io/docs/tasks/tools/install-minikube/
``` text
choco install minikube kubernetes-cli

kubernetes-cli v1.15.1 [Approved]
Minikube v1.2.0 [Approved]
kubernetes-cli v1.15.1 already installed.
```

Cleanup local state
``` text
# If you have previously installed minikube, and run:
minikube start
# And this command returns an error:
# machine does not exist
# You need to clear minikube’s local state:
minikube delete
```

Whats next
- https://kubernetes.io/docs/setup/learning-environment/minikube/

Запуск с использованием Hyper-V (вместо Virtualbox)
``` text
# указание драйвера и параметров при запуске
minikube start --vm-driver=hyperv
minikube start --memory=4096

# или настройка параметров
minikube config set vm-driver hyperv

# прочие параметры
minikube config set WantReportErrorPrompt false
minikube config set memory 2048
minikube config set memory 4096
minikube config set memory 6144
minikube config set disk-size 32GB

# откатываемся на virtualbox и стартуем
minikube config set vm-driver virtualbox
minikube start
* minikube v1.2.0 on windows (amd64)
* Creating virtualbox VM (CPUs=2, Memory=2048MB, Disk=20000MB) ...
* Configuring environment for Kubernetes v1.15.0 on Docker 18.09.6
* Downloading kubeadm v1.15.0
* Downloading kubelet v1.15.0
* Pulling images ...
* Launching Kubernetes ...
* Verifying: apiserver proxy etcd scheduler controller dns
* Done! kubectl is now configured to use "minikube"
```

**Важно**: minikube не работает под virtualbox 5.2.30, необходимо
обновляться до 6.0.10, тут работает немного получше. Дополнительно
ещё можно снизить количество одновременно запускаемых реплик.
После запуска первых подов перестают загружаться docker images,
при этом не зависит от порядка деплоя.
От объема выделенной памяти виртуалке ситуация зависит слабо.
Проверено на Windows и Linux рабочих станциях.
Чтобы заработало пришлось несколько раз перезапускать minikube.

### kind
Как альтернативу minikube можно kind попробовать,
он, вроде как, постабильнее будет.
- <https://kind.sigs.k8s.io/docs/user/quick-start/>

## Работа по ДЗ

### Запуск и проверка
Информацию о контекстах kubectl сохраняет в файле **~/.kube/config**
``` text
PS C:\> cd ~

PS C:\Users\admin> kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   16m   v1.15.0

PS C:\Users\admin> cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: C:\Users\admin\.minikube\ca.crt
    server: https://192.168.99.100:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: C:\Users\admin\.minikube\client.crt
    client-key: C:\Users\admin\.minikube\client.key
```

Обычно порядок конфигурирования kubectl следующий:
``` text
1) Создать cluster:
$ kubectl config set-cluster … cluster_name
2) Создать данные пользователя (credentials)
$ kubectl config set-credentials … user_name
3) Создать контекст
$ kubectl config set-context context_name \
--cluster=cluster_name \
--user=user_name
4) Использовать контекст
$ kubectl config use-context context_name

Текущий контекст можно увидеть так:
$ kubectl config current-context
minikube

Список всех контекстов можно увидеть так:
$ kubectl config get-contexts
```

Проверяем:
``` text
PS C:\Users\admin> kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   16m   v1.15.0

PS C:\Users\admin> kubectl config current-context
minikube

PS C:\Users\admin> kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube
```

### Deployment

``` bash
kubectl apply -f ui-deployment.yml
kubectl get deployment

# kubectl apply -f ./kubernetes/reddit

kubectl get pods --selector component=ui
# NAME                  READY   STATUS    RESTARTS   AGE
# ui-5868d4bf65-5pqhs   1/1     Running   0          3m
# ui-5868d4bf65-llct9   1/1     Running   0          3m
# ui-5868d4bf65-ltcxk   1/1     Running   0          3m

kubectl port-forward <pod-name> 8080:9292
# http://localhost:8080/

kubectl apply -f comment-deployment.yml
kubectl apply -f post-deployment.yml

kubectl apply -f mongo-deployment.yml

## cleanup if we need
kubectl delete --all pods --namespace=default
kubectl delete --all deployments --namespace=default

kubectl get deployment
kubectl get pods
```

### Services

``` bash
kubectl get deployment
kubectl get pods

kubectl describe service comment | grep Endpoints
kubectl exec -ti <pod-name> nslookup comment


kubectl apply -f ui-service.yml
kubectl apply -f comment-service.yml
kubectl apply -f post-service.yml
kubectl apply -f mongo-service.yml

kubectl get pods
kubectl port-forward <pod-name> 9292:9292
# http://localhost:9292/

kubectl logs <post-pod>

kubectl apply -f ...

kubectl logs <post-pod>

$ kubectl delete -f mongodb-service.yml
Или
$ kubectl delete service mongodb

## NodePort
minikube service ui

minikube services list

minikube addons list
kubectl get pods
```

### Namespaces

``` bash
kubectl get all -n kube-system --selector k8s-app=kubernetes-dashboard

minikube service kubernetes-dashboard -n kube-system

kubectl apply -f dev-namespace.yml

kubectl apply -n dev -f ...

minikube service ui -n dev

kubectl apply -f ui-deployment.yml -n dev
```

## Google Kubernetes Engine

Создаём через панель кластер, затем подключаемся:
``` bash
gcloud container clusters get-credentials standard-cluster-1 \
  --zone europe-west1-c --project docker-245721

kubectl config current-context
# gke_docker-245721_europe-west1-c_standard-cluster-1
```

Запустим наше приложение в GKE
``` bash
# Создадим dev namespace
kubectl apply -f ./kubernetes/reddit/dev-namespace.yml
# Задеплоим все компоненты приложения в namespace dev:
kubectl apply -f ./kubernetes/reddit/ -n dev
```

Откроем диапазон портов kubernetes для публикации сервисов
``` bash
Настройте:
• Название - произвольно, но понятно
• Целевые экземпляры - все экземпляры в сети
• Диапазоны IP-адресов источников  - 0.0.0.0/0
Протоколы и порты - Указанные протоколы и порты
tcp:30000-32767

gcloud compute --project=docker-245721 \
  firewall-rules create temp-kubernetes-default \
  --direction=INGRESS --priority=1000 \
  --network=default --action=ALLOW \
  --rules=tcp:30000-32767 --source-ranges=0.0.0.0/0
```

Проверка
``` bash
kubectl get pods -n dev

# Найдите внешний IP-адрес любой ноды из кластера
# либо в веб-консоли, либо External IP в выводе:
kubectl get nodes -o wide

# Найдите порт публикации сервиса ui
kubectl describe service ui -n dev | grep NodePort
Type:                     NodePort
NodePort:                 <unset>  32092/TCP

# Идем по адресу http://<node-ip>:<NodePort>
```

### Dashboard in GKE

``` bash
$ kubectl proxy
# Заходим по адресу http://localhost:8001/ui
```

Правильный адрес:
- [dashboard](http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default)

Security RBAC
``` bash
kubectl create clusterrolebinding kubernetes-dashboard \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:kubernetes-dashboard
```

## В процессе сделано:
План:
- Развернуть локальное окружение для работы с Kubernetes
- Развернуть Kubernetes в GKE
- Запустить reddit в Kubernetes
Выполнено:
- Установил minikube + kubectl последних версий локально
- Изменил файлы XX-deployment.yml
- Проверил проброс портов через localhost - всё ОК
- Создал файлы XX-service.yml
- Создал файл описания namespaces dev-namespace.yml
- Развернул k8s в GKE
- Запустил reddit в k8s GKE
- Сделал скриншот интерфейса приложения
- Запустил dashboard в GKE и настроил RBAC

## Как запустить проект:
Подключаемся к кластеру k8s и деплоим приложение
в тестовый неймспейс dev:
``` bash
# gcloud container clusters get-credentials ...
kubectl apply -f ./kubernetes/reddit/dev-namespace.yml
kubectl apply -f ./kubernetes/reddit/ -n dev
```

## Как проверить работоспособность:
- Идем по адресу http://<node-ip>:32092


# HW27. Ingress-контроллеры и сервисы в Kubernetes.
## kubernetes-3

## Работа по ДЗ

### ClusterIP
kube-dns
``` bash
kubectl get services -n dev
# NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
# comment      ClusterIP   10.83.5.217    <none>        9292/TCP         64m
# comment-db   ClusterIP   10.83.3.83     <none>        27017/TCP        64m
# mongodb      ClusterIP   10.83.6.77     <none>        27017/TCP        64m
# post         ClusterIP   10.83.10.199   <none>        5000/TCP         64m
# post-db      ClusterIP   10.83.1.81     <none>        27017/TCP        64m
# ui           NodePort    10.83.13.229   <none>        9292:32092/TCP   64m

# Проскейлим в 0 сервис, который следит, чтобы dns-kube
# подов всегда хватало
kubectl scale deployment --replicas 0 -n kube-system \
    kube-dns-autoscaler

# Проскейлим в 0 сам kube-dns
kubectl scale deployment --replicas 0 -n kube-system \
    kube-dns

# Проверяем
kubectl exec -ti -n dev <имя любого pod-а> ping comment
# ping: bad address 'comment'
# command terminated with exit code 1

# Вернем kube-dns-autoscale в исходную
kubectl scale deployment --replicas 1 -n kube-system \
    kube-dns-autoscaler
```

### NodePort

``` text
spec:
  type: NodePort
  ports:
  - port: 9292
    nodePort: 32092
    protocol: TCP
    targetPort: 9292
  selector:
    app: reddit
    component: ui
```

### LoadBalancer

``` text
spec:
  type: LoadBalancer
  ports:
  - port: 80
    nodePort: 32092
    protocol: TCP
    targetPort: 9292
  selector:
    app: reddit
    component: ui
```

Запускаем
``` bash
kubectl apply -f ui-service.yml -n dev
kubectl get services -n dev
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# ui           LoadBalancer   10.83.13.229   <pending>     80:32092/TCP   82m
kubectl get service  -n dev --selector component=ui
# NAME   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
# ui     LoadBalancer   10.83.13.229   35.205.186.119   80:32092/TCP   84m
```

### Ingress

Перейдите в настройки кластера в веб-консоли gcloud и
добавьте в дополнениях **Балансировка нагрузки HTTP**
(проверяем, возможно уже был включен)
- <https://console.cloud.google.com/kubernetes>

ui-ingress.yml
``` text
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ui
spec:
  backend:
    serviceName: ui
    servicePort: 80
```

Проверяем
- https://console.cloud.google.com/net-services/loadbalancing/loadBalancers/list

"Т.е. для работы с Ingress в GCP нам нужен минимум
Service с типом NodePort" - по факту им может являться
Service с типом LoadBalancer.

Запускаем
``` bash
kubectl apply -f ui-ingress.yml -n dev
kubectl get services -n dev
kubectl get ingress -n dev

# if we need restart
kubectl delete -f ui-service.yml -n dev
kubectl delete -f ui-ingress.yml -n dev

kubectl apply -f ui-service.yml -n dev
kubectl apply -f ui-ingress.yml -n dev
```

### Secret - TLS Termination

Теперь давайте защитим наш сервис с помощью TLS.
``` bash
# Для начала вспомним Ingress IP
kubectl get ingress -n dev
NAME   HOSTS   ADDRESS          PORTS   AGE
ui     *       35.244.212.169   80      103s

# Далее подготовим сертификат используя IP как CN
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=35.244.212.169"

# И загрузить сертификат в кластер kubernetes
kubectl create secret tls ui-ingress \
  --key tls.key --cert tls.crt -n dev

# Проверить можно командой
kubectl describe secret ui-ingress -n dev
```

Теперь настроим Ingress на прием только HTTPS траффика
изменяем ui-ingress.yml и применяем:
``` bash
kubectl apply -f ui-ingress.yml -n dev

kubectl get ingress -n dev
# NAME   HOSTS   ADDRESS          PORTS     AGE
# ui     *       35.244.212.169   80, 443   8m51s
```

Проверяем:
- https://console.cloud.google.com/net-services/loadbalancing/loadBalancers/list
При этом IP адрес для HTTPS в панели указан уже другой.

Иногда протокол HTTP может не удалиться у существующего
Ingress правила, тогда нужно его вручную удалить и пересоздать
``` bash
kubectl delete ingress ui -n dev
kubectl apply -f ui-ingress.yml -n dev
```

### Network Policy

Найдите имя кластера (и зону тоже !!)
``` text
$ gcloud beta container clusters list
NAME                LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION   NUM_NODES  STATUS
standard-cluster-1  europe-west1-c  1.12.8-gke.10   35.195.59.161  g1-small      1.12.8-gke.10  2          RUNNING
```

Включим network-policy для GKE
``` bash
## example
gcloud beta container clusters update <cluster-name> \
  --zone=us-central1-a --update-addons=NetworkPolicy=ENABLED
gcloud beta container clusters update <cluster-name> \
  --zone=us-central1-a  --enable-network-policy

## work
gcloud beta container clusters update standard-cluster-1 \
  --zone=europe-west1-c --update-addons=NetworkPolicy=ENABLED
gcloud beta container clusters update standard-cluster-1 \
  --zone=europe-west1-c --enable-network-policy
```

Применяем политику
``` bash
kubectl apply -f mongo-network-policy.yml -n dev
```

Примечание: для особо слабых инстансов ограничения не работают.
- <https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy>

``` text
Network policy is not supported for clusters whose nodes are f1-micro or g1-small
instances, as the resource requirements are too high for instances of that size.

Note: If you enable or disable network policy for an existing cluster, GKE is
required to re-create all of your cluster's node pools to ensure that the nodes
are configured to run the network policy process.
```

### Хранилище для базы

### Volume

``` text
    spec:
      containers:
      - image: mongo:3.2
        name: mongo
        volumeMounts:
        - name: mongo-persistent-storage
          mountPath: /data/db
      volumes:
      - name: mongo-persistent-storage
        emptyDir: {}
```

Сейчас используется тип Volume emptyDir. При создании пода с
таким типом просто создается пустой docker volume.
При остановке POD’a содержимое emtpyDir удалится навсегда. Хотя
в общем случае падение POD’a не вызывает удаления Volume’a.
Задание:
1) создайте пост в приложении
2) удалите deployment для mongo
3) Создайте его заново

``` bash
kubectl delete -f mongo-deployment.yml -n dev
kubectl delete -f mongodb-service.yml -n dev
kubectl delete service mongodb -n dev
###
kubectl apply -f mongo-deployment.yml -n dev
kubectl apply -f mongodb-service.yml -n dev
```

Вместо того, чтобы хранить данные локально на ноде, имеет смысл
подключить удаленное хранилище. В нашем случае можем
использовать Volume gcePersistentDisk, который будет складывать
данные в хранилище GCE.

``` bash
gcloud compute disks create --size=25GB \
  --zone=europe-west1-c reddit-mongo-disk

# WARNING: You have selected a disk size of under [200GB].
# This may result in poor I/O performance. For more information, see:
#   https://developers.google.com/compute/docs/disks#performance.
# Created [https://www.googleapis.com/compute/v1/projects/docker-245721/zones/europe-west1-c/disks/reddit-mongo-disk].
# NAME               ZONE            SIZE_GB  TYPE         STATUS
# reddit-mongo-disk  europe-west1-c  25       pd-standard  READY
#
# New disks are unformatted. You must format and mount a disk before
# it can be used. You can find instructions on how to do this at:
# https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting
```

Добавим новый Volume POD-у базы. Меняем Volume на другой тип.
``` text
    spec:
      containers:
      - image: mongo:3.2
        name: mongo
        volumeMounts:
        - name: mongo-gce-pd-storage
          mountPath: /data/db
      volumes:
      - name: mongo-persistent-storage
        emptyDir: {}
        volumes:
      - name: mongo-gce-pd-storage
        gcePersistentDisk:
          pdName: reddit-mongo-disk
          fsType: ext4
```

Проверяем что хранилище постоянно:
``` bash
# Монтируем выделенный диск к POD’у mongo
kubectl apply -f mongo-deployment.yml -n dev

# Дождитесь, пересоздания Pod'а (занимает до 10 минут).
# Зайдем в приложение и добавим пост

# Удалим deployment
kubectl delete deploy mongo -n dev

# Снова создадим деплой mongo.
kubectl apply -f mongo-deployment.yml -n dev
```

Список созданных дисков и использование ВМ
- <https://console.cloud.google.com/compute/disks>

### PersistentVolume

Используемый механизм Volume-ов можно сделать удобнее.
Мы можем использовать не целый выделенный диск для
каждого пода, а целый ресурс хранилища, общий для всего
кластера.
Тогда при запуске Stateful-задач в кластере, мы сможем
запросить хранилище в виде такого же ресурса, как CPU или
оперативная память.
Для этого будем использовать механизм PersistentVolume.

Создадим описание PersistentVolume - mongo-volume.yml
``` text
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: reddit-mongo-disk
spec:
  capacity:
    storage: 25Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    fsType: "ext4"
    pdName: "reddit-mongo-disk"
```

Добавим PersistentVolume в кластер
``` bash
kubectl apply -f mongo-volume.yml -n dev
```

### PersistentVolumeClaim

Мы создали ресурс дискового хранилища, распространенный
на весь кластер, в виде PersistentVolume.
Чтобы выделить приложению часть такого ресурса - нужно
создать запрос на выдачу - PersistentVolumeClaim.
Claim - это именно запрос, а не само хранилище.
С помощью запроса можно выделить место как из
конкретного PersistentVolume (тогда параметры accessModes
и StorageClass должны соответствовать, а места должно
хватать), так и просто создать отдельный PersistentVolume под
конкретный запрос.

Создадим описание PersistentVolumeClaim (PVC) - mongo-claim.yml
``` text
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
```

Добавим PersistentVolumeClaim в кластер
``` bash
kubectl apply -f mongo-claim.yml -n dev
```

Мы выделили место в PV по запросу для нашей базы.
Одновременно использовать один PV можно только по
одному Claim’у

Если Claim не найдет по заданным параметрам PV внутри
кластера, либо тот будет занят другим Claim’ом
то он сам создаст нужный ему PV воспользовавшись
стандартным StorageClass.

``` text
$ kubectl describe storageclass standard -n dev
Name:                  standard
IsDefaultClass:        Yes
Annotations:           storageclass.beta.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/gce-pd
Parameters:            type=pd-standard
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

Подключим PVC к нашим Pod'ам - меняем mongo-deployment.yml
``` text
    spec:
      containers:
      - image: mongo:3.2
        name: mongo
        volumeMounts:
        - name: mongo-gce-pd-storage
          mountPath: /data/db
      volumes:
      - name: mongo-gce-pd-storage
        persistentVolumeClaim:
          claimName: mongo-pvc
```

Обновим описание нашего Deployment’а
``` bash
kubectl apply -f mongo-deployment.yml -n dev
```

### Динамическое выделение Volume'ов

Создав PersistentVolume мы отделили объект "хранилища" от
наших Service'ов и Pod'ов. Теперь мы можем его при
необходимости переиспользовать.
Но нам гораздо интереснее создавать хранилища при
необходимости и в автоматическом режиме. В этом нам
помогут StorageClass’ы. Они описывают где (какой
провайдер) и какие хранилища создаются.
В нашем случае создадим StorageClass Fast так, чтобы
монтировались SSD-диски для работы нашего хранилища.

Создадим описание StorageClass’а - storage-fast.yml
``` text
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

Добавим StorageClass в кластер
``` bash
kubectl apply -f storage-fast.yml -n dev
```

Создадим описание PersistentVolumeClaim - mongo-claim-dynamic.yml
``` text
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 10Gi
```

Добавим StorageClass в кластер
``` bash
kubectl apply -f mongo-claim-dynamic.yml -n dev
```

Подключение динамического PVC
Подключим PVC к нашим Pod'ам mongo-deployment.yml
``` text
    spec:
      containers:
      - image: mongo:3.2
        name: mongo
        volumeMounts:
        - name: mongo-gce-pd-storage
          mountPath: /data/db
      volumes:
      - name: mongo-gce-pd-storage
        persistentVolumeClaim:
          claimName: mongo-pvc-dynamic
```

Обновим описание нашего Deployment'а
``` bash
kubectl apply -f mongo-deployment.yml -n dev
```

Давайте посмотрит какие в итоге у нас получились
PersistentVolume'ы
``` text
$ kubectl get persistentvolume -n dev
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                   STORAGECLASS   REASON   AGE
pvc-372571cc-b26e-11e9-8c9b-42010a8400fe   15Gi       RWO            Delete           Bound       dev/mongo-pvc           standard                19m
pvc-53083dfe-b270-11e9-8c9b-42010a8400fe   10Gi       RWO            Delete           Bound       dev/mongo-pvc-dynamic   fast                    4m15s
reddit-mongo-disk                          25Gi       RWO            Retain           Available                                                   24m
```

На созданные Kubernetes'ом диски можно посмотреть
в [web console](https://console.cloud.google.com/compute/disks)


## В процессе сделано:
- Протестировал сетевые типы и объекты
  - ClusterIP
  - NodePort
  - LoadBalancer
  - Ingress/Ingress Controller (надо долго ждать :)
  - Secret/TLS Termination (надо очень долго ждать :)
  - Network Policies
- Протестировал различные типы volumes и использование
  хранилищ PersistentVolumes и запросов PersistentVolumeClaims

## Как запустить проект:
Подключаемся к кластеру k8s и деплоим приложение
в тестовый неймспейс dev и загружаем ключи:
``` bash
# gcloud container clusters get-credentials ...
kubectl apply -f ./kubernetes/reddit/dev-namespace.yml
kubectl apply -f ./kubernetes/reddit/ -n dev
kubectl create secret tls ui-ingress \
  --key tls.key --cert tls.crt -n dev
```

## Как проверить работоспособность:
Перейти по ссылке (сертификат самоподписанный, HTTP отключен)
- https://IP


# HW28. Интеграция Kubernetes в GitlabCI.
## kubernetes-4

## Подготовка рабочего окружения

### Helm
Документация
- Quickstart Guide
  - https://docs.helm.sh/using_helm/#quickstart-guide
- install guide
  - https://docs.helm.sh/using_helm/#installing-helm

Версии
- https://github.com/helm/helm/releases
- Последняя стабильная версия [v2.14.2](https://github.com/helm/helm/releases/tag/v2.14.2)
- Версия в методичке [v2.9.1](https://github.com/helm/helm/releases/tag/v2.9.1)

``` bash
mkdir -p ~/otus/helm
cd ~/otus/helm
wget https://get.helm.sh/helm-v2.14.2-linux-amd64.tar.gz
tar -xzvf helm-v2.14.2-linux-amd64.tar.gz
mv linux-amd64/helm helm-v2.14.2
ln -s /home/nwton/work/otus/helm/helm-v2.14.2 ~/bin/helm
chmod +x ~/bin/helm
cd -
```

Helm читает конфигурацию kubectl (~/.kube/config) и сам определяет текущий
контекст (кластер, пользователь, неймспейс)
Если хотите сменить кластер, то либо меняйте контекст с помощью
$ kubectl config set-context
либо подгружайте helm’у собственный config-файл флагом --kube-context

### Tiler

Установим серверную часть Helm’а - Tiller
Tiller - это аддон Kubernetes, т.е. Pod, который общается с API Kubernetes.
Для этого понадобится ему выдать ServiceAccount и
назначить роли RBAC, необходимые для работы.

Создайте файл tiller.yml и поместите в него манифест
``` text
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

``` bash
# применяем
kubectl apply -f tiller.yml

# Теперь запустим tiller-сервер
helm init --service-account tiller
# Creating /home/nwton/.helm
# Creating /home/nwton/.helm/repository
# Creating /home/nwton/.helm/repository/cache
# Creating /home/nwton/.helm/repository/local
# Creating /home/nwton/.helm/plugins
# Creating /home/nwton/.helm/starters
# Creating /home/nwton/.helm/cache/archive
# Creating /home/nwton/.helm/repository/repositories.yaml
# Adding stable repo with URL:
#   https://kubernetes-charts.storage.googleapis.com
# Adding local repo with URL: http://127.0.0.1:8879/charts
# $HELM_HOME has been configured at /home/nwton/.helm.

# Tiller (the Helm server-side component) has been installed
# into your Kubernetes Cluster.

# Please note: by default, Tiller is deployed with an insecure
#   'allow unauthenticated users' policy.
# To prevent this, run `helm init` with the --tiller-tls-verify flag.
# For more information on securing your installation see:
# https://docs.helm.sh/using_helm/#securing-your-helm-installation
```

Проверим
``` bash
kubectl get pods -n kube-system --selector app=helm
# NAME                             READY   STATUS    RESTARTS   AGE
# tiller-deploy-66478cb847-mplmc   1/1     Running   0          99s
```

### Charts

Chart - это пакет в Helm.

Создайте директорию Charts в папке kubernetes со следующей
структурой директорий:
``` bash
mkdir -p Charts
mkdir -p Charts/comment
mkdir -p Charts/post
mkdir -p Charts/reddit
mkdir -p Charts/ui
```

Правки
``` bash
touch Charts/ui/Chart.yaml
git add Charts/ui/Chart.yaml

mkdir -p Charts/ui/templates
git mv reddit/ui-deployment.yml Charts/ui/templates/deployment.yaml
git mv reddit/ui-ingress.yml Charts/ui/templates/ingress.yaml
git mv reddit/ui-service.yml Charts/ui/templates/service.yaml
```

Вычищаем старые хвосты, которые остались
``` bash
## cleanup if we need
kubectl get deployment
kubectl get pods
kubectl get ingress
kubectl get services

kubectl delete --all pods --namespace=default
kubectl delete --all deployments --namespace=default
kubectl delete --all ingress --namespace=default
kubectl delete --all services --namespace=default

kubectl get deployment -n dev
kubectl get pods -n dev
kubectl get ingress -n dev
kubectl get services -n dev

kubectl delete --all pods --namespace=dev
kubectl delete --all deployments --namespace=dev
kubectl delete --all ingress --namespace=dev
kubectl delete --all services --namespace=dev
```

Установим Chart
```bash
helm install --name test-ui-1 ui/

# Посмотрим, что получилось
helm ls
```

### Templates

Шаблонизируем ui/templates/service.yaml
```text
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: reddit
    component: ui
    release: {{ .Release.Name }}
spec:
  type: NodePort
  ports:
  - port: {{ .Values.service.externalPort }}
    protocol: TCP
    targetPort: 9292
  selector:
    app: reddit
    component: ui
    release: {{ .Release.Name }}
```

Шаблонизируем ui/templates/deployment.yaml
```text
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: reddit
    component: ui
    release: {{ .Release.Name }}
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: reddit
      component: ui
      release: {{ .Release.Name }}
  template:
    metadata:
      name: ui
      labels:
        app: reddit
        component: ui
        release: {{ .Release.Name }}
    spec:
      containers:
      - image: chromko/ui
        name: ui
        ports:
        - containerPort: 9292
          name: ui
          protocol: TCP
        env:
        - name: ENV
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

Шаблонизируем ui/templates/ingress.yaml
```text
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  annotations:
    kubernetes.io/ingress.class: "gce"
spec:
  rules:
  - http:
      paths:
      - path: /*
        backend:
          serviceName: {{ .Release.Name }}-{{ .Chart.Name }}
          servicePort: 9292
```

Определим значения собственных переменных ui/values.yaml
```text
---
service:
  internalPort: 9292
  externalPort: 9292

image:
  repository: chromko/ui
  tag: latest
```

Установим несколько релизов ui
```bash
helm upgrade ui-1 ui/
helm upgrade ui-2 ui/
helm upgrade ui-3 ui/
# UPGRADE FAILED
# Error: "ui-1" has no deployed releases
# Error: UPGRADE FAILED: "ui-1" has no deployed releases

helm install ui --name ui-1
helm install ui --name ui-2
helm install ui --name ui-3

# Где ui-(1/2/3) - имена релизов

# Должны появиться 3 ingress'а
kubectl get ingress
# NAME      HOSTS   ADDRESS   PORTS     AGE
# ui        *                 80, 443   16m
# ui-1-ui   *                 80        33s
# ui-2-ui   *                 80        31s
# ui-3-ui   *                 80        30s
```

По IP-адресам можно попасть на разные релизы ui приложений.
P.S. подождите пару минут, пока ingress’ы станут доступными

Мы уже сделали возможность запуска нескольких версий
приложений из одного пакета манифестов, используя лишь
встроенные переменные. Кастомизируем установку своими
переменными (образ и порт).

Остальные подробности в PDF и gist, там простая копипаста.

Переносим остальные
``` bash
## post
touch Charts/post/Chart.yaml
git add Charts/post/Chart.yaml

mkdir -p Charts/post/templates
git mv reddit/post-deployment.yml Charts/post/templates/deployment.yaml
git mv reddit/post-service.yml Charts/post/templates/service.yaml

## comment
touch Charts/comment/Chart.yaml
git add Charts/comment/Chart.yaml

mkdir -p Charts/comment/templates
git mv reddit/comment-deployment.yml Charts/comment/templates/deployment.yaml
git mv reddit/comment-service.yml Charts/comment/templates/service.yaml

touch Charts/ui/Chart.yaml
git add Charts/ui/Chart.yaml

mkdir -p Charts/ui/templates
git mv reddit/ui-deployment.yml Charts/ui/templates/deployment.yaml
git mv reddit/ui-ingress.yml Charts/ui/templates/ingress.yaml
git mv reddit/ui-service.yml Charts/ui/templates/service.yaml
```

### Templates прочее выжимка

Поскольку адрес БД может меняться в зависимости от
условий запуска:
- бд отдельно от кластера
- бд запущено в отдельном релизе
- ... , то создадим удобный шаблон для задания адреса БД

```text
env:
  - name: POST_DATABASE_HOST
    value: {{ .Values.databaseHost }}
```

Будем задавать бд через переменную databaseHost. Иногда
лучше использовать подобный формат переменных вместо
структур database.host, так как тогда прийдется определять
структуру database, иначе helm выдаст ошибку.
Используем функцию default. Если databaseHost не будет
определена или ее значение будет пустым, то используется
вывод функции printf (которая просто формирует строку <имярелиза>-mongodb)
```text
value: {{ .Values.databaseHost | default (printf "%s-mongodb" .Release.Name) }}
```

В итоге должно получиться (ссылка на gist **ОТСУТСТВУЕТ**)
```text
env:
  - name: POST_DATABASE_HOST
    value: {{ .Values.databaseHost | default (printf "%s-mongodb"
.Release.Name) }}
```

Теперь, если databaseHost не задано, то будет использован
адрес базы, поднятой внутри релиза.

Более подробная по шаблонизации и функциям:
- <https://docs.helm.sh/chart_template_guide/#the-chart-template-developer-s-guide>


Также стоит отметить функционал helm по использованию
helper’ов и функции templates. Helper - это написанная нами
функция. В функция описывается, как правило, сложная логика.
Шаблоны этих функция распологаются в файле _helpers.tpl

Пример функции comment.fullname:
charts/comment/templates/_helpers.tpl
```text
{{- define "comment.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name }}
{{- end -}}
```

которая в результате выдаст то же, что и:
```text
{{ .Release.Name }}-{{ .Chart.Name }}
```

И заменим в соответствующие строчки в файле, чтобы
использовать helper charts/comment/templates/service.yaml
```text
---
apiVersion: v1
kind: Service
metadata:
  name: {{ template "comment.fullname" . }}
  labels:
    app: reddit
    component: comment
    release: {{ .Release.Name }}
spec:
  type: ClusterIP
  ports:
  - port: {{ .Values.service.externalPort }}
    protocol: TCP
    targetPort: {{ .Values.service.internalPort }}
  selector:
    app: reddit
    component: comment
    release: {{ .Release.Name }}
```

Helper'ы хранятся в Charts/<component>/templates/_helpers.tpl

### Управление зависимостями

Мы создали Chart’ы для каждой компоненты нашего
приложения. Каждый из них можно запустить по-отдельности
командой
``` text
$ helm install <chart-path> <release-name>
```
Но они будут запускаться в разных релизах, и не будут видеть
друг друга.
С помощью механизма управления зависимостями создадим
единый Chart reddit, который объединит наши компоненты

Подготовка пакета (на основе requirements.yaml):
```bash
$ helm dep update ./reddit
```
Появится файл requirements.lock с фиксацией
зависимостей Будет создана директория charts с зависимостями
в виде архивов

Chart для базы данных не будем создавать вручную. Возьмем
готовый.
``` text
1. Найдем Chart в общедоступном репозитории
$ helm search mongo
2. добавим в reddit/requirements.yml
3. Выгрузим зависимости
$ helm dep update
```

Выкатывание релиза (как для отдельного компонента, так и для всего приложения):
```bash
$ helm install --name <release-name> <path>
$ helm install --name test-ui-1 ui/
```

Обновление релиза после внесения изменений:
```bash
$ helm upgrade <release-name> <path>
```

Полное удаление релиза:
```bash
$ helm del --purge <release-name>
```

Раскатываем приложение:
```bash
cd kubernetes/Charts
helm install reddit --name reddit-test

# Найдите внешний IP-адрес любой ноды из кластера
# либо в веб-консоли, либо External IP в выводе:
kubectl get nodes -o wide
kubectl get ingress
kubectl get services
```

Обновляемся после внесения изменений:
```bash
helm dep update ./reddit
# helm upgrade <release-name> ./reddit
helm upgrade reddit-test ./reddit
```

## GitLab + Kubernetes

Добавляем новый пул узлов:
- назовите его bigpool
- 1 узел типа n2-standard-2 (7,5 Гб, 2 виртуальных ЦП)
- Размер диска 20-40 Гб

Для отключения RBAC - в настройках кластера необходимо
установить "Устаревшие права доступа" = включено.

Gitlab будем ставить также с помощью Helm Chart’а из пакета Omnibus.

```text
1. Добавим репозиторий Gitlab
$ helm repo add gitlab https://charts.gitlab.io

2. Мы будем менять конфигурацию Gitlab, поэтому скачаем Chart
$ helm fetch gitlab/gitlab-omnibus --version 0.1.37 --untar
$ cd gitlab-omnibus
```

Настройка
```bash
1. Поправьте gitlab-omnibus/values.yaml ( )
https://raw.githubusercontent.com/express42/otus-snippets/master/kubernetes-4/gitlab/omnibus-values.yml

# baseDomain: example.com
# legoEmail: you@example.com

2. Добавьте в gitlab-omnibus/templates/gitlab/gitlabsvc.yaml ( )
https://raw.githubusercontent.com/express42/otus-snippets/master/kubernetes-4/gitlab/svc.yml

apiVersion: v1
kind: Service
metadata:
name: {{ template "fullname" . }}
…
selector:
name: {{ template "fullname" . }}
ports:
…
- name: prometheus
port: 9090
targetPort: prometheus
- name: web
port: 80
targetPort: workhorse

3. Поправить в gitlab-omnibus/templates/gitlabconfig.yaml (ссылка на gist)
https://raw.githubusercontent.com/express42/otus-snippets/master/kubernetes-4/gitlab/config.yml

apiVersion: v1
kind: ConfigMap
…
heritage: "{{ .Release.Service }}"
data:
external_scheme: http
external_hostname: {{ template "fullname" . }}

4. Поправить в gitlab-omnibus/templates/ingress/gitlab-ingress.yaml
https://raw.githubusercontent.com/express42/otus-snippets/master/kubernetes-4/gitlab/ingress.yml

apiVersion: extensions/v1beta1
kind: Ingress
…
spec:
tls:
…
rules:
- host: {{ template "fullname" . }}
```

Установим GitLab
```bash
$ helm install --name gitlab . -f values.yaml

==> v1beta1/Ingress
NAME           HOSTS                                                                  ADDRESS  PORTS  AGE
gitlab-gitlab  gitlab-gitlab,registry.example.com,mattermost.example.com + 1 more...  80, 443  2s


NOTES:

  It may take several minutes for GitLab to reconfigure.
    You can watch the status by running
  # kubectl get deployment -w gitlab-gitlab --namespace default
  You did not specify a baseIP so one will be assigned for you.
  It may take a few minutes for the LoadBalancer IP to be available.
  Watch the status with:
  # kubectl get svc -w --namespace nginx-ingress nginx
  then:
  export SERVICE_IP=$(kubectl get svc --namespace nginx-ingress nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

  Then make sure to configure DNS with something like:
    *.example.com       300 IN A $SERVICE_IP
```

Должно пройти несколько минут. Найдите выданный IP-адрес
ingress-контроллера nginx.
```text
$ kubectl get service -n nginx-ingress nginx
NAME    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                                   AGE
nginx   LoadBalancer   10.83.0.226   104.199.45.238   80:30039/TCP,443:31765/TCP,22:30161/TCP   2m52s
```

Поместите запись в локальный файл /etc/hosts
(поставьте свой IP-адрес)
```text
# для windows %WinDir%\System32\Drivers\Etc
echo "104.199.45.238 gitlab-gitlab staging production" >> /etc/hosts
```

Ждем пока gitlab поднимется
```bash
$ kubectl get pods
NAME                                        READY   STATUS             RESTARTS   AGE
gitlab-gitlab-58d785b57b-78mrm              0/1     Running            0          5m24s
gitlab-gitlab-postgresql-784bcc4487-k7cfm   0/1     Pending            0          5m24s
gitlab-gitlab-redis-55b589c99c-zdgnx        1/1     Running            0          5m24s
gitlab-gitlab-runner-5f8575bb9c-mjnv6       0/1     CrashLoopBackOff   4          5m24s
```

Проблема с запуском (Pending состояние):
```bash
kubectl get pods
kubectl describe pods ***
# pod has unbound immediate PersistentVolumeClaims
kubectl describe pvc

# Нужно удалить лишнее из предыдущего ДЗ
# Список созданных дисков и использование ВМ
# https://console.cloud.google.com/compute/disks

helm del --purge gitlab
helm install --name gitlab . -f values.yaml

# если что-то зависло на середине (например отключение RBAC)
helm upgrade gitlab .

# обновляем IP в hosts после перезапуска
kubectl get service -n nginx-ingress nginx
# NAME    TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                   AGE
# nginx   LoadBalancer   10.83.12.150   34.76.78.159   80:32164/TCP,443:30358/TCP,22:31021/TCP   6m42s

echo "34.76.78.159 gitlab-gitlab staging production" >> /etc/hosts

kubectl get pods
# NAME                                        READY   STATUS    RESTARTS   AGE
# gitlab-gitlab-58d785b57b-86zwp              1/1     Running   0          8m1s
# gitlab-gitlab-postgresql-784bcc4487-tw5nx   1/1     Running   0          8m1s
# gitlab-gitlab-redis-55b589c99c-gtvzv        1/1     Running   0          8m1s
# gitlab-gitlab-runner-5f8575bb9c-s8bqj       1/1     Running   3          8m1s
```

И теперь Идем по адресу http://gitlab-gitlab
Устанавливаем новый пароль и логинимся.

### Работа в Gitlab под k8s

И теперь Идем по адресу http://gitlab-gitlab
Устанавливаем новый пароль и логинимся.

Создаем группу с именем = Docker ID

В настройках группы выберите пункт CI/CD

Добавьте 2 переменные
- *CI_REGISTRY_USER* - логин в dockerhub
- *CI_REGISTRY_PASSWORD* - пароль от Docker Hub

Переменную с паролем от docker hub не делайте protected, иначе пайплайн всё
время будет вываливаться с ошибкой на этапе аутентификации в docker hub.

Из всех пайплайнов от otus в function install_tiller() в строке
"helm init --upgrade" убирайте ключ "--upgrade", иначе пайплайн
будет падать с ошибкой:
"Error: error when upgrading: current Tiller version is newer,
use --force-upgrade to downgrade".

Создаем проекты:
- reddit-deploy
- post
- ui
- comment

Создаем каталог и копируем туда содержимое src и подключаем
каждый проект как отдельный репозиторий.
Папку Charts копируем в reddit-deploy

Для авторизации по ключу вместо логина-пароля при пуше в репу:
необходимо добавить свой ключ в настройки.

Пример:
```bash
cd ui
git init
git remote add origin git@gitlab-gitlab:nwton/ui.git
git add .
git commit -m "init"
git push origin master
git push --set-upstream origin master
cd ..
```

## В процессе сделано:
План:
- Работа с Helm
- Развертывание Gitlab в Kubernetes
- Запуск CI/CD конвейера в Kubernetes
Выполнено

## Как запустить проект:
```bash
cd kubernetes/Charts
helm install reddit --name reddit-test
kubectl get ingress
```

## Как проверить работоспособность:
Перейти по ссылке http://IP

# HW29. Kubernetes. Мониторинг и логирование
## kubernetes-5

## Подготовка

У вас должен быть развернуть кластер k8s:
- минимум 2 ноды g1-small (1,5 ГБ)
- минимум 1 нода n1-standard-2 (7,5 ГБ)

В настройках:
- Stackdriver Logging - Отключен
- Stackdriver Monitoring - Отключен
- Устаревшие права доступа - Включено

Дефолтные настройки кластера в GCE:
- Stackdriver Kubernetes Engine Monitoring	= Отключено
- Прежняя версия Stackdriver Logging	= Включен
- Прежняя версия Stackdriver Monitoring	= Включен

Из Helm-чарта установим ingress-контроллер nginx
```bash
helm install stable/nginx-ingress --name nginx

kubectl --namespace default get services \
  -o wide -w nginx-nginx-ingress-controller

# NOTES:
# The nginx-ingress controller has been installed.
# It may take a few minutes for the LoadBalancer IP to be available.
# You can watch the status by running
# kubectl --namespace default get services \
#   -o wide -w nginx-nginx-ingress-controller'

# An example Ingress that makes use of the controller:

#   apiVersion: extensions/v1beta1
#   kind: Ingress
#   metadata:
#     annotations:
#       kubernetes.io/ingress.class: nginx
#     name: example
#     namespace: foo
#   spec:
#     rules:
#       - host: www.example.com
#         http:
#           paths:
#             - backend:
#                 serviceName: exampleService
#                 servicePort: 80
#               path: /
#     # This section is only required if TLS is to be enabled for the Ingress
#     tls:
#         - hosts:
#             - www.example.com
#           secretName: example-tls

# If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

#   apiVersion: v1
#   kind: Secret
#   metadata:
#     name: example-tls
#     namespace: foo
#   data:
#     tls.crt: <base64 encoded cert>
#     tls.key: <base64 encoded key>
#   type: kubernetes.io/tls

```

Найдите IP-адрес, выданный nginx’у
```bash
$ kubectl get svc
NAME                                  TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
kubernetes                            ClusterIP      10.83.0.1      <none>         443/TCP                      19h
nginx-nginx-ingress-controller        LoadBalancer   10.83.7.21     34.76.78.159   80:30786/TCP,443:31917/TCP   90m
nginx-nginx-ingress-default-backend   ClusterIP      10.83.12.162   <none>         80/TCP                       90m
```

Поместите запись в локальный файл /etc/hosts
```bash
# для windows %WinDir%\System32\Drivers\Etc
echo "34.76.78.159 reddit reddit-prometheus reddit-grafana reddit-non-prod production reddit-kibana staging prod" >> /etc/hosts
```

## Мониторинг

### Стек
В задании будем использовать уже знакомые нам инструменты:
- prometheus - сервер сбора и
- grafana - сервер визуализации метрик
- alertmanager - компонент prometheus для алертинга
- различные экспортеры для метрик prometheus
Prometheus отлично подходит для работы с контейнерами и
динамичным размещением сервисов

### Установим Prometheus
Prometheus будем ставить с помощью Helm чарта
Загрузим prometheus локально в Charts каталог
```bash
$ cd kubernetes/charts && helm fetch --untar stable/prometheus
```

**Важно**: на слайдах неправильный регистр, у нас папка Charts !

Создайте внутри директории чарта файл custom_values.yml
Поместите в него содержимое
- https://gist.githubusercontent.com/chromko/2bd290f7becdf707cde836ba1ea6ec5c/raw/c17372866867607cf4a0445eb519f9c2c377a0ba/gistfile1.txt

Основные отличия от values.yml:
- отключена часть устанавливаемых сервисов
  (pushgateway, alertmanager, kube-state-metrics)
- включено создание Ingress’а для подключения через nginx
- поправлен endpoint для сбора метрик cadvisor
- уменьшен интервал сбора метрик (с 1 минуты до 30 секунд)

Запустите Prometheus в k8s из charsts/prometheus
```bash
helm upgrade prom . -f custom_values.yml --install

# NOTES:
# The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
# prom-prometheus-server.default.svc.cluster.local

# From outside the cluster, the server URL(s) are:
# http://reddit-prometheus
# For more information on running Prometheus, visit:
# https://prometheus.io/
```

Зайдем по ссылке
http://reddit-prometheus/

### Targets

У нас уже присутствует ряд endpoint’ов для сбора метрик
- Метрики API-сервера
- метрики нод с cadvisor’ов
- сам prometheus

Отметим, что можно собирать метрики cadvisor’а
(который уже является частью kubelet) через
проксирующий запрос в kube-api-server

Если зайти по ssh на любую из машин кластера и запросить
$ curl http://localhost:4194/metrics то получим те же метрики у
kubelet напрямую

Но вариант с kube-api предпочтительней, т.к. этот трафик
шифруется TLS и требует аутентификации.

Таргеты для сбора метрик найдены с помощью service discovery
(SD), настроенного в конфиге prometheus (лежит в custom-values.yml)

Использование SD в kubernetes позволяет нам динамично
менять кластер (как сами хосты, так и сервисы и приложения)
Цели для мониторинга находим c помощью запросов к k8s API:

Role объект, который нужно найти:
• node
• endpoints
• pod
• service
• ingress

Т.к. сбор метрик prometheus осуществляется поверх
стандартного HTTP-протокола, то могут понадобится доп.
настройки для безопасного доступа к метрикам.
Ниже приведены настройки для сбора метрик из k8s API
```text
scheme: https
tls_config:
 ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
 insecure_skip_verify: true
bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```
1. Схема подключения - http (default) или https
2. Конфиг TLS - коревой сертификат сервера для проверки
достоверности сервера
3. Токен для аутентификации на сервере

Подробнее о том, как работает relabel_config
- <https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config>

### Metrics
Все найденные на эндпоинтах метрики сразу же
отобразятся в списке (вкладка Graph). Метрики
Cadvisor начинаются с container_.

Cadvisor собирает лишь информацию о потреблении ресурсов
и производительности отдельных docker-контейнеров.
При этом он ничего не знает о сущностях k8s (деплойменты,
репликасеты, ...).

Для сбора этой информации будем использовать сервис
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)

Он входит в чарт Prometheus. Включим его.
prometheus/custom_values.yml
```text
kubeStateMetrics:
  ## If false, kube-state-metrics will not be installed
  ##
  enabled: true
```

Обновим релиз prometheus
```bash
helm upgrade prom . -f custom_values.yml --install
```

По аналогии с kube_state_metrics включите (enabled: true)
поды node-exporter в custom_values.yml.
Проверьте, что метрики начали собираться с них.

### Метрики приложений
Запустите приложение из helm чарта reddit

```bash
helm upgrade reddit-test ./reddit --install
helm upgrade production --namespace production ./reddit --install
helm upgrade staging --namespace staging ./reddit --install
```

Раньше мы *хардкодили* адреса/dns-имена наших
приложений для сбора метрик с них.

Теперь мы можем использовать механизм ServiceDiscovery
для обнаружения приложений, запущенных в k8s.

Приложения будем искать так же, как и служебные сервисы k8s.

Модернизируем конфиг prometheus custom_values.yml
```text
 - job_name: 'reddit-endpoints'
 kubernetes_sd_configs:
 - role: endpoints
 relabel_configs:
 - source_labels: [__meta_kubernetes_service_label_app]
 action: keep
 regex: reddit
```
Используем действие keep, чтобы оставить
только эндпоинты сервисов с метками **app=reddit**

Мы получили эндпоинты, но что это за поды мы не знаем.
Добавим метки k8s
Все лейблы и аннотации k8s изначально отображаются в
prometheus в формате:
__meta_kubernetes_service_label_labelname
__meta_kubernetes_service_annotation_annotationname

Модернизируем конфиг prometheus custom_values.yml
```text
custom_values.yml
 relabel_configs:
 - action: labelmap
 regex: __meta_kubernetes_service_label_(.+)
```
Отобразить все совпадения групп из regex в label’ы Prometheus

Теперь мы видим лейблы k8s, присвоенные POD’ам

Добавим еще label’ы для prometheus и обновим helm-релиз
Т.к. метки вида __meta_* не публикуются, то нужно создать
свои, перенеся в них информацию

Модернизируем конфиг prometheus custom_values.yml
```text
 - source_labels: [__meta_kubernetes_namespace]
 target_label: kubernetes_namespace
 - source_labels: [__meta_kubernetes_service_name]
 target_label: kubernetes_name
```
Обновите релиз prometheus и ...

Сейчас мы собираем метрики со всех сервисов reddit’а в 1 группе
target-ов.
Мы можем отделить target-ы компонент друг от друга (по
окружениям, по самим компонентам), а также выключать и
включать опцию мониторинга для них с помощью все тех же labelов.
Например, добавим в конфиг еще 1 job (ссылка на gist):
- https://gist.githubusercontent.com/chromko/0d0bf3961b84bc2f0b56b0cd073bc217/raw/c916cbe30a34eceb9389a8b253969dc47a3ab857/gistfile1.txt

Для разных лейблов разные регекспы
Обновим релиз prometheus и посмотрим:
Метрики будут отображаться для всех инстансов приложений

Задание 32
Разбейте конфигурацию job’а `reddit-endpoints`
(слайд 24) так, чтобы было 3 job’а для каждой из компонент
приложений (post-endpoints, commentendpoints, ui-endpoints),
а reddit-endpoints уберите.

Модернизируем конфиг prometheus custom_values.yml
(Речь идёт про эту часть со слайда 25)
```text
 - job_name: 'reddit-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
```

Обновиляем релиз prometheus после каждого изменения
```bash
helm upgrade prom . -f custom_values.yml --install
```

### Визуализация

Поставим также grafana с помощью helm
**ВАЖНО** Опять ошибки на слайде:
Вот так, как написано в слайдах - не работает:
```bash
helm upgrade --install grafana stable/grafana \
--set "server.adminPassword=admin" \
--set "server.service.type=NodePort" \
--set "server.ingress.enabled=true" \
--set "server.ingress.hosts={reddit-grafana}"

# NOTES:
# 1. Get your 'admin' user password by running:
#
#    kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
#
# 2. The Grafana server can be accessed via port 80 on the following
# DNS name from within your cluster:
#
#    grafana.default.svc.cluster.local
#
#    Get the Grafana URL to visit by running these commands in the
# same shell:
#
#      export POD_NAME=$(kubectl get pods --namespace default -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
#      kubectl --namespace default port-forward $POD_NAME 3000
#
# 3. Login with the password from step 1 and the username: admin
#
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

**ВАЖНО** Опять ошибки на слайде:
Вот так - работает:
```bash
helm upgrade --install grafana stable/grafana \
--set "adminPassword=admin" \
--set "service.type=NodePort" \
--set "ingress.enabled=true" \
--set "ingress.hosts={reddit-grafana}"

# NOTES:
# 1. Get your 'admin' user password by running:
#
#    kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
#
# 2. The Grafana server can be accessed via port 80 on the following
# DNS name from within your cluster:
#
#    grafana.default.svc.cluster.local
#
#    From outside the cluster, the server URL(s) are:
#      http://reddit-grafana
#
# 3. Login with the password from step 1 and the username: admin
```

http://reddit-grafana/
- admin
- admin

Добавьте prometheus data-source,
это будет http://prom-prometheus-server

Адрес найдите из имени сервиса prometheus сервера
```text
$ kubectl get svc
NAME                                  TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
grafana                               ClusterIP      10.83.13.202   <none>         80/TCP                       7m45s
kubernetes                            ClusterIP      10.83.0.1      <none>         443/TCP                      20h
nginx-nginx-ingress-controller        LoadBalancer   10.83.7.21     34.76.78.159   80:30786/TCP,443:31917/TCP   141m
nginx-nginx-ingress-default-backend   ClusterIP      10.83.12.162   <none>         80/TCP                       141m
prom-prometheus-kube-state-metrics    ClusterIP      None           <none>         80/TCP                       24m
prom-prometheus-node-exporter         ClusterIP      None           <none>         9100/TCP                     22m
prom-prometheus-server                LoadBalancer   10.83.3.156    34.76.51.7     80:31326/TCP                 33m
reddit-test-comment                   ClusterIP      10.83.13.1     <none>         9292/TCP                     20m
reddit-test-mongodb                   ClusterIP      10.83.13.117   <none>         27017/TCP                    20m
reddit-test-post                      ClusterIP      10.83.3.8      <none>         5000/TCP                     20m
reddit-test-ui                        NodePort       10.83.12.148   <none>         9292:30771/TCP               20m
```

Добавим самый распространенный dashboard для
отслеживания состояния ресурсов k8s
- https://grafana.com/dashboards/315

Добавьте собственные дашборды, созданные ранее (в ДЗ по
мониторингу). Они должны также успешно отобразить данные.
Должно быть что-то вроде такого:

### Templating

В коде json соответствующий блок templating

Примечание: в новой версии почему-то постоянно пропадает блок
и все datasource заменяются на хардкод значения, что может
вызвать проблемы при дальнейшем импорте (если имена отличаются).
```text
  "__inputs": [
    {
      "name": "DS_PROMETHEUS_SERVER",
      "label": "Prometheus Server",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],

...
      "datasource": "${DS_PROMETHEUS_SERVER}",
...
```

### Смешанные графики
Импортируйте следующий график (**дашборд !!!**) :
https://grafana.com/dashboards/741
На этом графике одновременно используются метрики и
шаблоны из cAdvisor, и из kube-state-metrics для
отображения сводной информации по деплойментам

## Логирование
1. Данную часть ДЗ рекомендуется выполянять уже после
выполения ДЗ № 25 (по логированию), наработки из
которого используюстя в данной работе
3. Выполнение данной части ДЗ не является обязательным
4. Добавьте label самой мощной ноде в кластере

Снова на слайдах проблема - нужно сначала найти имя
самой большой ноды, а затем уже вешать лейбл.
```bash
kubectl get node
kubectl label node gke-standard-cluster-1-bigpool-5655fd96-n6s4 elastichost=true
```

### Стек
Логирование в k8s будем выстраивать с помощью уже
известного стека EFK:
- ElasticSearch - база данных + поисковый движок
- Fluentd - шипер (отправитель) и агрегатор логов
- Kibana - веб-интерфейс для запросов в хранилище и
  отображения их результатов

Создайте файлы в новой папке kubernetes/ef/
fluentd-ds.yaml (ссылка на gist)
fluentd-configmap.yaml (ссылка на gist)
es-service.yaml (ссылка на gist)
es-statefulSet.yaml (ссылка на gist)
es-pvc.yaml (ссылка на gist)

Запустите стек в вашем k8s
```bash
kubectl apply -f ./efk
# persistentvolumeclaim/elasticsearch-logging-claim created
# service/elasticsearch-logging created
# statefulset.apps/elasticsearch-logging created
# configmap/fluentd-es-config-v0.1.1 created
# daemonset.apps/fluentd-es-v2.0.2 created
```

Kibana поставим из helm чарта
```text
helm upgrade --install kibana stable/kibana \
--set "ingress.enabled=true" \
--set "ingress.hosts={reddit-kibana}" \
--set "env.ELASTICSEARCH_URL=http://elasticsearch-logging:9200" \
--version 0.1.1
```

http://reddit-kibana/

Создаем индекс на fluentd-*

Откройте вкладку Discover в Kibana и введите в строку поиска выражение
```text
kubernetes.labels.component:post OR kubernetes.labels.component:comment OR
kubernetes.labels.component:ui
```
Откройте любой из рез-тов поиска - в нем видно множество инфы о k8s

1. Особенность работы fluentd в k8s состоит в том,
  что его задача помимо сбора самих логов
  приложений, сервисов и хостов, также
  распознать дополнительные метаданные (как
  правило это дополнительные поля с лейблами)
2. Откуда и какие логи собирает fluentd - видно в
  его fluentd-configmap.yaml и в fluentd-ds.yaml
3. Общая схема работы fluentd представлена на след. слайде

## Работа по ДЗ
All done

## В процессе сделано:
План:
- Развертывание Prometheus в k8s
- Настройка Prometheus и Grafana для сбора метрик
- Настройка EFK для сбора логов
Выполнено, часть найденных ошибок записал в README

## Как запустить проект:
Запустить helm, аналогично выполнению ДЗ

## Как проверить работоспособность:
Перейти по ссылкам:
http://reddit-prometheus - прометей
http://reddit-grafana - графана
http://reddit-kibana - кибана

Ссылки если задеплоить приложение:
http://reddit
http://reddit-non-prod
http://production
http://staging
http://prod


# Курсовой проект
