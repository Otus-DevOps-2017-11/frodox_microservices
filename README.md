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

## Homework #28 (Kubernetes. The Hard Way)

[Стартуем туториал](https://github.com/kelseyhightower/kubernetes-the-hard-way) . Labs

### Подготовка

* ansible setup GCE ([based on](http://docs.ansible.com/ansible/devel/scenario_guides/guide_gce.html))
** [Создаём сервисную учётку GCE](https://console.cloud.google.com/iam-admin/serviceaccounts/project?project=docker-196320&authuser=1)
** Качаем от учётки JSON ключ доступа. Почту
** Ставим зависимости
```
# pip install apache-libcloud

export GCE_INI_PATH=$(pwd)/kubernetes/ansible/inventory/gce.ini
```
** в каталоге ansible/inventory используем gce.ini/py из офф.репы ansible
** фиксим поля `gce_*` в файле `gce.ini`

* установка и логин GCloud SDK, tmux
```
# set default region
gcloud config set compute/region europe-west2
```
* установка cfssl, cfssljson
* установка kubectl

#### Настройка GCloud окружения

* Настройка сети и firewall
```
# VPC и подсеть
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24

# firewall
## internal
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24

## external
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24

## show rules
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

* Создаём статический внешний IP для LB
```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)

# убеждаемся что всё ок
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

* Создание 3 compute instances для k8s контроллеров
```
# выбираем зону europe-west2-c (21)
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

* Создаём 3 инстанса для воркеров
```
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
    --tags kubernetes-the-hard-way,worker \
    --zone europe-west2-c
done

# проверяем результат
gcloud compute instances list
```

* Настройка SSH-доступа

```
gcloud compute ssh controller-0
```

#### Создание CA и генерация TLS сертификатов

* Создание CA
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
> ca-key.pem
> ca.pem
```

* Создание клиентских и серверных сертификатов
** `admin` клиентский сертификат
```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

> admin-key.pem
> admin.pem
```

** `kubelet` клиентский сертификат и приватный ключ

Result: `worker-{0,2}.pem`, `worker-{0,2}-key.pem`

** `kube-controller-manager` клиентский сертификат и приватный ключ

Result: `kube-controller-manager.pem`, `kube-controller-manager-key.pem`

** `kube-proxy` клиентский сертификат и приватный ключ

Result: `kube-proxy.pem`, `kube-proxy-key.pem`

** `kube-scheduler` клиентский сертификат и приватный ключ

Result: `kube-scheduler.pem`, `kube-scheduler-key.pem`

** `kube-api-server` клиентский сертификат и приватный ключ

Result: `kubernetes.pem`, `kubernetes-key.pem`

** `service-account` клиентский сертификат и приватный ключ

Result: `service-account.pem`, `service-account-key.pem`

* Распространяем ключи по соответствующим нодам:

```
# по воркерам
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done

# по контроллерам
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

#### Создание kubernetes конфигов для аутентификации

* Конфиги для воркеров `worker-{0,2}.kubeconfig`
* Конфиг для `kube-proxy` --- `kube-proxy.kubeconfig`
* Конфиг для `kube-controller-manager` --- `kube-controller-manager.kubeconfig`
* Конфиг для `kube-scheduler` --- `kube-scheduler.kubeconfig`
* Конфиг для user `admin` --- `admin..kubeconfig`
* Распространяем конфиги
```
# для воркеров
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done

# для контроллеров
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

#### Создание конфига и ключа для шифрования данных

* Создание ключа шифрования из `/dev/urandom`
* Создание конфига шифрования
* Распространяем конфиг по контроллерам
```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

### Поднятие кластера etcd

```
cd ansible
ansible-playbook k8s-thw-etcd.yml
```

### Bootstrapping the Kubernetes Control Plane

```
cd ansible
ansible-playbook -v k8s-thw-control-plane.yml
```

* Вручную настраиваем k8s FrontEnd LoadBalancer с хоста
* Проверка, что всё OK
```
$ KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

$ curl --cacert ../kubernetes_the_hard_way/certs/ca.em https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version

{
  "major": "1",
  "minor": "10",
  "gitVersion": "v1.10.2",
  "gitCommit": "81753b10df112992bf51bbc2c2f85208aad78335",
  "gitTreeState": "clean",
  "buildDate": "2018-04-27T09:10:24Z",
  "goVersion": "go1.9.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

### Bootstrapping the Kubernetes Worker Nodes

```
cd ansible
ansible-playbook -v k8s-thw-workers.yml

# Проверка
gcloud compute ssh controller-0 --command "kubectl get nodes --kubeconfig admin.kubeconfig"

kubecfrodox@controller-1:~$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    53s       v1.10.2
worker-1   Ready     <none>    54s       v1.10.2
worker-2   Ready     <none>    51s       v1.10.2
```

### Настройка kubectl для удалённого доступа

* Настраиваем локальный kubeconfig для доступа к k8s в GCloud

Проверка
```
kubectl get nodes
```

### Provisioning Pod Network Routes

* Настраиваем по гайду.
* Проверка:
```
gcloud compute routes list --filter "network: kubernetes-the-hard-way"

NAME                                NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-22e04d3521a7a799      kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000 default-route-c5a01c0fdf69c863      kubernetes-the-hard-way  10.240.0.0/24                            1000
kubernetes-route-10-200-0-0-24      kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000 kubernetes-route-10-200-1-0-24      kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24      kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```

### Adding DNS add-on

```
ansible-playbook k8s-thw-dns-addon.yml

kubectl run busybox --image=busybox --command -- sleep 3600
# тестируесм DNS запрос изнутри конетйнера
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

### Smoke test

* шифрование
* деплои
* проброс портов
* Логи контейнера
* Exec
* Сервисы
* Ненадёжные источники

### Создание своих деплойментов для сервиса reddit

```
kubectl apply -f comment-deployment.yml
kubectl apply -f mongo-deployment.yml
kubectl apply -f post-deployment.yml
kubectl apply -f ui-deployment.yml


frodox:.bin/ $ kubectl get pods                                         
NAME                                 READY     STATUS    RESTARTS   AGE
busybox-68654f944b-8hn6s             1/1       Running   0          29m
comment-deployment-5f984959d-k2rf2   1/1       Running   0          2m
comment-deployment-5f984959d-vbttz   1/1       Running   0          2m
comment-deployment-5f984959d-vrrrt   1/1       Running   0          2m
mongo-deployment-778dcd865b-7tqlq    1/1       Running   0          2m
mongo-deployment-778dcd865b-87wbk    1/1       Running   0          2m
mongo-deployment-778dcd865b-twkpv    1/1       Running   0          2m
nginx-65899c769f-c2rp9               1/1       Running   0          13m
post-deployment-8476bcc7d-2hzl6      1/1       Running   0          2m
ui-deployment-849d948776-4ck44       1/1       Running   0          2m
ui-deployment-849d948776-r79lr       1/1       Running   0          2m
ui-deployment-849d948776-s9mqg       1/1       Running   0          2m
untrusted                            1/1       Running   0          8m
```

### Удаление всего кластера

* виртуальные машины
* сетевые пулы
* файрволы
* network VPC
