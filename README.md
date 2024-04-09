# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию в kubernetes
### Запустите проект.

Зайдите в папку `backend_main_django` и создайте кластер

```sh
minikube start --driver=virtualbox --no-vtx-check
```

Активируйте Ingress:
```
minikube addons enable ingress
```

Затем создайте образ проекта из `Dockerfile`

```sh
minikube image build -t django_app .
```

## Использование ConfigMap для конфигурации приложения

Необходимые изменения:

`DEBUG`: Установите в `"False"`, если вы развертываете приложение в продакшн.
`ALLOWED_HOSTS`: Замените `"*"` на список доменов, с которых разрешен доступ к вашему приложению.

Примените манифест ConfigMap к Minikube кластеру:

```
kubectl apply -f ConfigMap.yaml
```

## Создание 'Secret' в Kubernetes

Для управления секретными данными, такими как `SECRET_KEY` и `DATABASE_URL`, в Kubernetes используется объект `Secret`. 
Вы можете создать `Secret`, используя следующую команду:

```sh
kubectl create secret generic django-secrets \
--from-literal=SECRET_KEY=<your_secret_key> \
--from-literal=DATABASE_URL=<your_database_url>
```

Замените `<your_secret_key>` и `<your_database_url>` соответствующими значениями.


# Использование Kubernetes Secrets для хранения конфиденциальных данных

## Создание Secret

Для создания объекта `Secret` с вашими конфиденциальными данными выполните следующие шаги:

Вам нужно закодировать ваши конфиденциальные данные в формате base64. Например, используйте команду `echo -n 'your_data_here' | base64` в Linux или MacOS терминале.

В файле django-secrets.yaml замените `<base64_encoded_DATABASE_URL>` и `<base64_encoded_SECRET_KEY>` на ваши закодированные значения:

```sh
DATABASE_URL: <base64_encoded_DATABASE_URL>
SECRET_KEY: <base64_encoded_SECRET_KEY>
```

Примените манифест в вашем кластере Kubernetes с помощью команды:

```sh
kubectl apply -f django-secrets.yaml
```


Запустите процесс развертывания вашего Django проекта в кластере Kubernetes

```sh
kubectl apply -f kubernetes/web-deployment.yaml
```

Установите и настройте [Helm](https://helm.sh/):
```
sudo snap install helm --classic
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres-15 bitnami/postgresql
helm list
```

Следуя инструкциям, появившимся после создания пода базы данных, создайте создайте базу данных и пользователя:
```
CREATE DATABASE yourdbname;
CREATE USER youruser WITH ENCRYPTED PASSWORD 'yourpass';
GRANT ALL PRIVILEGES ON DATABASE yourdbname TO youruser;
ALTER USER youruser SUPERUSER;
```

Для применения миграций базы данных используйте команду:
```
kubectl apply -f kubernetes/django-migrate.yaml
```

Раз в месяц будет запускаться автоматическая очистка сессий Django-приложения.
Если возникла необходимость очистить сессии Django-приложения не по расписанию, то это можно сделать в ручную:
```
kubectl create job --from=cronjob/django-clearsessions any_job_name
```

После внесения всех изменений введите следующую команду:
```
kubectl apply -f kubernetes/
```

## Запуск в кластере

Сайт должен быть доступен пользователям в интернете, а для этого нам понадобится кластер с
публичным IP-адресом и доменом. Такой кластер можно самостоятельно арендовать у облачных хостинг-
провайдеров, настроить всё. 

В данном проекте заранее подготовлен и настроен кластер YandexCloud:

### Ресурсы кластера:
* Выделен домен `edu-mad-jang.sirius-k8s.dvmn.org`. Запросы обрабатывает Yandex Application Load Balancer.
* В Yandex Managed Service for PostgreSQL создана база данных. Доступы лежат в секрете K8s.
* В Yandex Application Load Balancer создан роутер. Он распределяет входящие сетевые запросы на разные NodePort кластера K8s.
* Настроен S3 Bucket. Токены и прочие настройки доступа к Object Storage API лежат в секрете K8s.

### Размещение образа в docker registry
Необходимо разместить образ приложения в [dockerhub](https://hub.docker.com):
* Локально соберите образ:
```shell
docker build -t имя_образа:тег -f путь_к_Dockerfile .
```
* Войдите в Docker Hub:
```shell
docker login
```
* Тегируйте локальный образ:
```shell
docker tag имя_образа:тег ваше_имя_пользователя/имя_репозитория:тег
```
* Загрузите образ в Docker Hub:
```shell
docker push ваше_имя_пользователя/имя_репозитория:тег
```

### Получение SSL-сертификата для подключения к базе данных PostgreSQL
PostgreSQL-хосты с публичным доступом поддерживают только шифрованные соединения. Чтобы использовать их, получите SSL-сертификат

* Linux(Bash)/macOS(Zsh)
```shell
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
* Windows(PowerShell)
```shell
mkdir $HOME\.postgresql; curl.exe -o $HOME\.postgresql\root.crt https://storage.yandexcloud.net/cloud-certs/CA.pem
```

Сертификат будет сохранен в файле `$HOME\.postgresql\root.crt`

### Развёртывание приложения в кластере

Создайте сервис:
```shell
kubectl apply -f kubernetes_dev/django-service.yaml
```
