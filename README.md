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


# HW16. Docker образы. Микросервисы
## docker-3


# HW17. Сетевое взаимодействие Docker контейнеров. Docker Compose. Тестирование образов
## docker-4
