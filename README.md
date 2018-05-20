# frodox_microservices

## Homework #14 (docker-1)

ДЗ посвящено знакомству с Docker. Что сделано:

* установлен `docker{,-compose,machine}`
* проверка работы `docker run hello-world`
* Осознанна разница между образом и контейнером

Основные команды работы с docker

* `docker ps -a` --- список всех контейнеров
* `docker images` --- список локальных образов
* `docker run --rm -it ubuntu:latest /bin/bash` --- запуск нового контейнера из указанного **образа**
* `docker start <container_id>` --- старт ранее созданного **контейнера**
* `docker attach <container_id>` --- присоединение к терминалу запущенного контейнера (`^p,q` --- отсоединиться)
* `docker exec -it <container_id> <bash>` --- запуск команды внутри запущенного контейнера
* `docker stop <container_id>` --- SIGTERM и через 10сек KILL контейнера (предпочтительно при необходимости корректного завершения)
* `docker rm <container_id> [-f]` --- удалить [работающий] контейнер
* `docker rmi <image_id> [-f]` --- удалить [используемый] образ


## Homework #15 (docker-2)

* Сконфигурировали GCloud для работы с новым проектом
* Установлен `docker-machine-0.13`
* Собран базовый образ `reddit:latest`
* Зарегистрирован аккаунт на dockerHub (вход по `docker login`)
* Образ из ДЗ загружен на DockerHub. [Ссылка](https://hub.docker.com/r/frodox/otus-reddit/)


Пример создания хост-машины с docker-ом (на ней будем тренироваться):

```
docker-machine create --driver google \
    --google-project $(cat gcp.id) \
    --google-zone europe-west1-b \
    --google-machine-type g1-small \
    --google-machine-image \
    $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
    docker-host

eval $(docker-machine env docker-host)
```

Сборка образа и запуск контейнера:

```
docker build -t reddit:latest .
docker run --name reddit -d --network=host reddit:latest
```

Добавляем правило firewall

```
gcloud compute firewall-rules create reddit-app \
    --allow tcp:9292 --priority=65534 \
    --target-tags=docker-machine \
    --description="Allow TCP connections" \
    --direction=INGRESS
```

Наше приложение доступно напрямую из мира (см. `docker-machine ls`).

Загружаем наш образ в DockerHub.

```
docker login
...
docker tag reddit:latest frodox/otus-reddit:1.0
docker push frodox/otus-reddit:1.0
```


### Задание со `*`:
Сравнить вывод

```
docker run --rm -it tehbilly/htop
docker run --rm --pid host -it tehbilly/htop
```

вторая команда создаёт контейнер в PID-namespace хоста, а не новом
и изолированном. Таким образом контейнер потенциально может иметь доступ ко всем процессам хоста, что не безопасно.


## Homework 16 (docker-3)

* Проба создания Dockerfile для микросервисного приложения
* Пробы по оптимизации размера/создания Docker образов
* Запуск микросервисного приложения вручную
* Использование volume для перманентного сохранения данных

Для начала активируем работу с `docker-host`, по аналогии с 15-м уроком.

Сборка образов

```
cd reddit-microservices
docker build -t frodox/post:1.0 ./post-py/
docker build -t frodox/ui:1.0 ./ui/
docker build -t frodox/comment:1.0 ./comment/
```

Сборка `ui` началась не с первого шага, потому что использовался кеш сборки от `comment`.

Запуск контейнеров

```
docker network create reddit
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post frodox/post:1.0
docker run -d --network=reddit --network-alias=comment frodox/comment:1.0
docker run -d --network=reddit -p 9292:9292 frodox/ui:1.0
```

Создали bridge сеть для контейнеров и запустили их в ней. Тест доступен по
`http://<docker-host-external-ip>:9292/`

Создание volume-а `docker build -t frodox/ui:1.0 ./ui/`

Запуск контейнера с mongoDB с подключённым volume:

```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
```

После перезагрузки контейнеров изменения (нвоые посты) остаются. Чудо!


## Homework 17 (docker-4)

* Пробуем на вкус драйвера сети: none, host, bridge
* Установлен docker-compose
* Написан docker-compose.yml для сборки и запуска микросервисного приложения

Выводы команд

```
docker exec -ti net_test ifconfig
docker-machine ssh docker-host ifconfig
```

совпадают, по понятным причинам - единый сетевой namespace:)

Одновременный запуск нескольких nginx контейнеров выглядит как успешный, однако
все последующие контейнеры тут же завершаются с ошибкой что Adress already in use,
т.к. они пытаются присоединиться к одному и тому же порту в одном и том же сетевом пространстве имён.


```
# старт сервисов
docker-compose up -d

# стоп сервисов
docker-compose down
```

## Homework 19 (GitLab CI)

Основные цели занятия:

* подготовка/запуск инсталяции GitLab omnibus
* настройка репозитория с приложением на запуск тестов
* настройка `.gitlab-ci.yml` для начала процесса непрерывной интеграции

---

Создание машины в GCloud:

```
~/opt/docker-machine-0.13 create --driver google \
    --google-project $(cat gcp.id) \
    --google-zone europe-west1-b \
    --google-machine-type n1-standard-1 --google-disk-size 100 \
    --google-machine-image \
    $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
    --google-tags 'default-allow-ssh,http-server,https-server' \
    gitlab-ci

eval $(~/opt/docker-machine-0.13 env gitlab-ci)
```

Конфигурирование и запуск GitLab
```
cd ansible
ansible-playbook gitlab-prepare.yml
```
Ждём пару минут. Как GitLab становится доступен по external IP нашей VM,
логинимся, создаём проект, получаем токен CI/CD, запускаем и регистрируем раннера

```
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest

docker exec -it gitlab-runner gitlab-runner register

```

## Homework 20 (GitLab CI)

В ходе домашнего задания:

* расширен существующий pipeline
* Добавлены динамические окружения, stage, prod

## Homework 21 (Prometheus)

* Подготовка окружения

```
# firewall
gcloud compute firewall-rules create prometheus-default --allow tcp:9090
gcloud compute firewall-rules create puma-default --allow tcp:9292

# create docker VM
export GOOGLE_PROJECT=$(cat gcp.id)
~/opt/docker-machine-0.13 create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    vm1

# configure local env
eval $(~/opt/docker-machine-0.13 env vm1)

docker run --rm -p 9090:9090 -d --name prometheus  prom/prometheus
```

* Сборка докера с Prometheus с нашим конфигом

```
cd monitoring/prometheus
export USER_NAME=frodox
docker build -t $USER_NAME/prometheus .
```

* Сборка всех сервисов

```
for i in ui post-py comment;
do
    pushd src/$i
    (bash ./docker_build.sh || echo "ERROR" >&2) &
    popd
done
wait
```

* Запуск всех сервисов

```
cd docker/
cp reddit.env .env
docker-compose up -d
```

* Пушим собранные образы в docker.io

```
docker login
...
for i in ui comment post prometheus;
do
    echo -e "\nPushing $USER_NAME/$i\n"
    docker push $USER_NAME/$i || echo "ERRORRRRR" >&2
done
```

`docker-machine rm vm1`

* Ссылки на образы:
  * https://hub.docker.com/r/frodox/prometheus/
  * https://hub.docker.com/r/frodox/post/
  * https://hub.docker.com/r/frodox/ui/
  * https://hub.docker.com/r/frodox/comment/


# Homework 23 (Monitoring-2)


* Подготовка окружения

```
# firewall
gcloud compute firewall-rules create cadvisor-default --allow tcp:8080
gcloud compute firewall-rules create grafana-default --allow tcp:3000
gcloud compute firewall-rules create alertmanager-default --allow tcp:9093

# create docker VM
export GOOGLE_PROJECT=$(cat gcp.id)
~/opt/docker-machine-0.13 create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    vm1

# configure local env
eval $(~/opt/docker-machine-0.13 env vm1)
```

* Билдим образы всех приложений

```
for i in ui post-py comment;
do
    pushd src/$i
    (bash ./docker_build.sh || echo "ERROR" >&2) &
    popd
done
wait
```

* Стартуем

```
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d
```

* Работа над графаной и алертмэнеджером

* Отправка всех образов в докерхаб

```
for i in ui comment post prometheus alertmanager;
do
    echo -e "\nPushing $USER_NAME/$i\n"
    docker push $USER_NAME/$i || echo "ERRORRRRR" >&2
done
```

Ссылка: https://hub.docker.com/u/frodox/


## Homework #25 (Logging & Tracing)

В ходе домашней работы:
* Настроен сбор логов
* Работа со структурированными логами
* Работа с неструктурированными логами
* Распределённая трасировка приложений

Ход работ:

* Создание новой тачки и открытие доп.портов
```
export GOOGLE_PROJECT=$(cat gcp.id)
~/opt/docker-machine-0.13 create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-open-port 5000/tcp \
    --google-open-port 5601/tcp \
    --google-open-port 9292/tcp \
    --google-open-port 9411/tcp \
    logging

eval $(~/opt/docker-machine-0.13 env vm2)

gcloud compute firewall-rules create zipkin-default --allow tcp:9411
gcloud compute firewall-rules create alertmanager-default --allow tcp:9411
```

* Обновлён исходный код в каталоге `/src`
* Сборка новых версий всех образов:
```
for i in ui post-py comment;
do
    pushd src/$i
    (bash ./docker_build.sh || echo "ERROR building $i" >&2) &
    popd
done
wait
```

## Homework #27 (Swarm-1)

* Создаём машины для работы (мастер и воркеры)

```
GCP_ID=$(cat gcp.id)
for m in master-1 worker-1 worker-2;
do
    docker-machine create --driver google \
    --google-project $GCP_ID \
    --google-zone europe-west1-b \
    --google-machine-type g1-small \
    --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
    $m
done
eval $(docker-machine env master-1)
```

* Инициалиазция swarm-кластера

```
docker-machine ssh master-1
sudo docker swarm init
```

Получаем команду для подключения воркеров:
```
docker swarm join --token SWMTKN-1-3k3gaf7p52vqld8zqvnfm53pual9q81cec1kg00kll3kw97inz-0um7ij57wpv1jbshlzg3qnzdz 10.132.0.2:2377
```
Подключаем обоих.

При необходимости, для подключения менеджера достаточно выполнить: `docker swarm join-token manager`

* Создаём docker-compose-swarm.yml для Stack
* Запускаем Stack
```
cd docker
docker stack deploy --compose-file=<(docker-compose -f docker-compose-swarm.yml config 2>/dev/null) DEV
```

* Смотрим его состояние
```
docker stack services DEV
```

Инфа по лейблам (для ограничения размещения)
* `node.*` --- встроенные label для ноды
* `node.labels*` --- заданные вручную label для ноды
* `engine.labels.*` --- label-ы engine

* Добавляем label к мастер-ноде
```
docker node update --label-add reliability=high master-1
```

* Посмотреть label всех нод (пока только так)
```
docker node ls -q | xargs docker node inspect  -f '{{ .ID }} [{{ .Description.Hostname }}]: {{ .Spec.Labels }}'
```

* Поднимаем мониторинг:
```
docker stack deploy --compose-file=<(docker-compose -f docker-compose-monitoring.yml config 2>/dev/null) DEV
```

* Добавляем ещё одного воркера:
```
docker-machine create --driver google \
    --google-project $GCP_ID \
    --google-zone europe-west1-b \
    --google-machine-type g1-small \
    --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
    worker-3

docker-machine ssh worker-3
```

Смотрим контейнеры до добавления воркера:
```
$ docker stack ps DEV

lhekvyex0g0h  DEV_node-exporter.wtg17ff84v6k38h63l8q2rwsh  prom/node-exporter:v0.15.2  worker-2  Running
3trjbxcl1pxw  DEV_node-exporter.pdk3l24byngvyqdshvyobu35x  prom/node-exporter:v0.15.2  worker-1  Running
oakbqxk4p8go  DEV_node-exporter.kdndoqmjq4ovgyen7bjgqkh5n  prom/node-exporter:v0.15.2  master-1  Running
53yduslwbhya  DEV_cadvisor.1                               google/cadvisor:v0.29.0     worker-1  Running
4705deb9uzp9  DEV_alertmanager.1                           frodox/alertmanager         master-1  Running
sgo6gi2ldpq4  DEV_prometheus.1                             frodox/prometheus           master-1  Running
gl2qvv0fkrs6  DEV_grafana.1                                grafana/grafana:5.0.0       worker-2  Running
8v23z8uz47yj  DEV_post.1                                   frodox/post:latest          worker-2  Running
i89r1txxmyrd  DEV_mongo.1                                  mongo:3.2                   master-1  Running
3mom8norwgph  DEV_comment.1                                frodox/comment:latest       worker-2  Running
t7194nkrid2e  DEV_ui.1                                     frodox/ui:latest            worker-2  Running
p66a0kf53f9x  DEV_post.2                                   frodox/post:latest          worker-1  Running
2orf6j5co7tv  DEV_comment.2                                frodox/comment:latest       worker-1  Running
8x1xybteqx8o  DEV_ui.2                                     frodox/ui:latest            worker-1  Running
```

* Добавляем ещё одного worker
```
sudo docker swarm join --token SWMTKN-1-3k3gaf7p52vqld8zqvnfm53pual9q81cec1kg00kll3kw97inz-0um7ij57wpv1jbshlzg3qnzdz 10.132.0.2:2377
```

Текущая картина:
```
➜  docker (swarm-1) docker stack ps DEV
lhekvyex0g0h  DEV_node-exporter.wtg17ff84v6k38h63l8q2rwsh  prom/node-exporter:v0.15.2  worker-2
3trjbxcl1pxw  DEV_node-exporter.pdk3l24byngvyqdshvyobu35x  prom/node-exporter:v0.15.2  worker-1
oakbqxk4p8go  DEV_node-exporter.kdndoqmjq4ovgyen7bjgqkh5n  prom/node-exporter:v0.15.2  master-1
rtwd48j46lc5  DEV_node-exporter.09kmoxlvtrhzx75c3719plobm  prom/node-exporter:v0.15.2  worker-3
53yduslwbhya  DEV_cadvisor.1                               google/cadvisor:v0.29.0     worker-1
4705deb9uzp9  DEV_alertmanager.1                           frodox/alertmanager         master-1
sgo6gi2ldpq4  DEV_prometheus.1                             frodox/prometheus           master-1
gl2qvv0fkrs6  DEV_grafana.1                                grafana/grafana:5.0.0       worker-3
8v23z8uz47yj  DEV_post.1                                   frodox/post:latest          worker-2
i89r1txxmyrd  DEV_mongo.1                                  mongo:3.2                   master-1
3mom8norwgph  DEV_comment.1                                frodox/comment:latest       worker-3
t7194nkrid2e  DEV_ui.1                                     frodox/ui:latest            worker-2
p66a0kf53f9x  DEV_post.2                                   frodox/post:latest          worker-1
2orf6j5co7tv  DEV_comment.2                                frodox/comment:latest       worker-1
8x1xybteqx8o  DEV_ui.2                                     frodox/ui:latest            worker-3
➜  docker (swarm-1)
```

т.е. переехали сервисы:
* node_exporter
* grafana
* comment
* ui

Увеличиваем в конфиге число реплик до трёх, запускаем, смотрим:
```
➜  docker (swarm-1) docker stack ps DEV
lhekvyex0g0h  DEV_node-exporter.wtg17ff84v6k38h63l8q2rwsh  prom/node-exporter:v0.15.2  worker-2 
3trjbxcl1pxw  DEV_node-exporter.pdk3l24byngvyqdshvyobu35x  prom/node-exporter:v0.15.2  worker-1 
oakbqxk4p8go  DEV_node-exporter.kdndoqmjq4ovgyen7bjgqkh5n  prom/node-exporter:v0.15.2  master-1 
rtwd48j46lc5  DEV_node-exporter.09kmoxlvtrhzx75c3719plobm  prom/node-exporter:v0.15.2  worker-3 
53yduslwbhya  DEV_cadvisor.1                               google/cadvisor:v0.29.0     worker-1 
4705deb9uzp9  DEV_alertmanager.1                           frodox/alertmanager         master-1 
sgo6gi2ldpq4  DEV_prometheus.1                             frodox/prometheus           master-1 
gl2qvv0fkrs6  DEV_grafana.1                                grafana/grafana:5.0.0       worker-3 
8v23z8uz47yj  DEV_post.1                                   frodox/post:latest          worker-2 
i89r1txxmyrd  DEV_mongo.1                                  mongo:3.2                   master-1 
3mom8norwgph  DEV_comment.1                                frodox/comment:latest       worker-3 
t7194nkrid2e  DEV_ui.1                                     frodox/ui:latest            worker-2 
p66a0kf53f9x  DEV_post.2                                   frodox/post:latest          worker-1 
2orf6j5co7tv  DEV_comment.2                                frodox/comment:latest       worker-1 
8x1xybteqx8o  DEV_ui.2                                     frodox/ui:latest            worker-3 
vuiand8p6h98  DEV_post.3                                   frodox/post:latest          worker-3 
u1jncjfcam72  DEV_comment.3                                frodox/comment:latest       worker-2 
3cgwagw5eie0  DEV_ui.3                                     frodox/ui:latest            worker-1 
```

теперь на третем воркере добавился сервис `post`. Все предыдущие продолжают быть запущены.
Таким образом, при увеличении количества реплик новые контейнеры с сервисом занимабт тех
воркеров, где они ещё не запущены (экземпляр этого сервиса). Уже запущенные экземпляры остаются там,
где и были запущены.


* Запуск приложений вместе с мониторингом:
```
docker stack deploy --compose-file=<(docker-compose -f docker-compose-swarm.yml -f docker-compose.monitoring.yml config 2>/dev/null) DEV
```
