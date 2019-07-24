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

## Работа по ДЗ

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


# HW24. Контейнерная оркестрация.
## Только теория.


# KUBERNETES

# HW25. Введение в Kubernetes.
## kubernetes-1


# HW26. Основные модели безопасности и контроллеры в Kubernetes
## kubernetes-2


# HW27. Ingress-контроллеры и сервисы в Kubernetes.
## kubernetes-3


# HW28. Интеграция Kubernetes в GitlabCI.
## kubernetes-4


# HW29. Kubernetes. Мониторинг и логирование
## kubernetes-5


# Курсовой проект
