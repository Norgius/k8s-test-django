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

## Kubernetes 

Установим `minikube` и `kubectl`. Для этого перейдите по ссылкам и скачайте их:
[kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и [minikube](https://minikube.sigs.k8s.io/docs/start/).

Если вы работает на `Windows`, то вам необходимо прописать путь до папки с программами `minikube` и `kubectl` в переменной среде `Path`.

Также не забудьте установить [VirtualBox](https://www.virtualbox.org/).

Запустим сам `minikube` (настройки `--cpus`, `--memory` и `--disk-size` опциональны):
```
minikube start --driver=virtualbox --no-vtx-check --cpus=3 --memory=4gb --disk-size=10gb
```

Установите `helm`:
```
choco install kubernetes-helm
```
Установите `PostgreSQL` с помощью `helm` (`my-release` имя вашей `postgresql`):
```
helm install my-release oci://registry-1.docker.io/bitnamicharts/postgresql
```
После установки вам будут представлены команды для взаимодействия с `postgresql`. 

__Внимание!!!__ Команды ниже могут отличаться от тех, которые вы получите после установки `postgresql`.

Получите пароль вашей `postgresql` командой:
```
kubectl get secret --namespace default my-release-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d
```
Второй командой вы сможете подключиться к `postgresql` (вместо `YOUR_PASSWORD` подставьте ваш полученный пароль):
```
kubectl run my-release-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r4 --env="PGPASSWORD=YOUR_PASSWORD" --command -- psql --host my-release-postgresql -U postgres -d postgres -p 5432
```
Создайте базу данных и пользователя:
```
CREATE DATABASE kuber_db;
CREATE USER db_user WITH PASSWORD 'db_password';
GRANT ALL PRIVILEGES ON DATABASE kuber_db TO db_user;
```

Перейдите в директорию `kuber`.
Заполните файл `kuber_configmap.yaml`. О переменных окружения можно почитать в `README` выше.

Также заполните `kuber_secrets.yaml`. Для `DATABASE_URL` значение будет вида:
```
"postgres://db_user:db_password@my-release-postgresql:5432/kuber_db"
```
`my-release-postgresql` - это сервис, который создал `helm`, именно к нему мы и будем подключаться.

Теперь запустим все `yaml-файлы` по очереди:
```
kubectl apply -f kuber_configmap.yaml
kubectl apply -f kuber_deployment.yaml
kubectl apply -f kuber_service.yaml
kubectl apply -f kuber_ingress.yaml
```
Запустим `jobs`:
```
kubectl apply -f kuber_migrate.yaml
kubectl apply -f kuber_clearsessions.yaml
```
Если миграции не применяются, а внутри самого пода при применении позникают ошибки вида:
```
LINE 1: CREATE TABLE "django_migrations" ("id" serial NOT NULL PRIMA...
```
Тогда вам необходимо поключиться к вашей созданной БД:
```
kubectl run my-release-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r4 --env="PGPASSWORD=YOUR_PASSWORD" --command -- psql --host my-release-postgresql -U postgres -d <ВАША_БАЗА_ДАННЫХ> -p 5432
```
И применить команды:
```
GRANT ALL ON SCHEMA public TO <ВАШ_USER>;
GRANT ALL ON SCHEMA public TO public;
```
После этого снова примените `job` с миграцией.

Для того, чтобы сайт был доступен необходимо настроить локальный DNS. Если вы работаете на Windows, то путь к нужному файлу будет `Windows\System32\drivers\etc\hosts`. Добивим строку с `IP` миникуба в файл (в моём случае IP будет такой - 192.168.59.105):
```
192.168.59.105 star-burger.test
```

Сайт будет доступен по адресу [star-burger.test](http://star-burger.test)