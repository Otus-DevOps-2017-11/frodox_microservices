# frodox_microservices

## Homework #14 (docker)

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
