# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

Если необходимо протестировать приложение в Kubernetes необходимо также установить [Kubernetes](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и [Minikub](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download) для тестов


## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Запуск приложения в кластере Minikube

Для локальной разработки необходимо поднять на хосте БД с помощью docker compose

Создайте в корне проекта файл **.env**

```
POSTGRES_DB=you_name_db
POSTGRES_USER=you_user
POSTGRES_PASSWORD=you_pasword
```

```bash
docker compose up -d db
```

Запустить minikube кластер

```
minikube start
```

Проверяем запуск

```
kubectl get nodes
```

В каталоге **kubernetes** cоздаем файл **secret.yaml** и заполняем переменные **SECRET_KEY**, **DATABASE_URL**, **DEBUG**, **ALLOWED_HOSTS** предварительно конвертируем в формат **base64**

где:
    **DATABASE_URL** в формате **DB_URL=postgres://you_user:password@IP_host:5432/name_db**
IP_host можно получить в настройках вашего соединения

```bash
echo -n 'you_env' | base64
```
Полученное значение вставляем в манифест секрета

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: "bXktc3VwZXItc2VjcmV0LWtleQ=="
  DATABASE_URL: "cG9zdGdyZXM6Ly9taW91c2VyOjIwMThAMTkyLjE2OC4wLjEzOjU0MzIvbWluaWN1YmVfZGI="
  DEBUG: 'RmFsc2U='
  ALLOWED_HOSTS: "Kg=="
```

Применяем манифест

```bash
cd kubernetes
kubectl apply -f secret.yaml
```

Запускаем наш сайт командой

```bash
kubectl apply -f django-deployment-v1.yaml
```

Получаем имя pod

```bash
kubectl get pods
```
Применяем миграции и создаем суперпользователя

```bash
kubectl exec -it django-shell -- /bin/bash
./manage.py migrate
./manage.py createsuperuser
./
exit
```

Запускаем сайт

```bash
minikube service django-service
```

Проверяем что нащ сайт запустился.
