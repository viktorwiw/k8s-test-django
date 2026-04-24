## Деплой в production (Kubernetes)

### Архитектура

В production используется следующая схема:

Браузер → ALB → Nginx → Nginx Unit → Django

- Внешний доступ осуществляется по HTTPS
- Трафик проходит через балансировщик (ALB)
- Далее попадает в Nginx
- Nginx проксирует запросы в Nginx-Unit. Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI.
- Nginx Unit проксирует запросы в Django

---

## Требования для работы приложения

Для запуска приложения необходимы:

- PostgreSQL (Managed PostgreSQL в Яндекс Облаке)
- SSL-сертификат для подключения к базе
- Kubernetes Secret с переменными для Django
- Kubernetes Secret с SSL-сертификатом (`pg-ssl-cert`)
- Nginx (как reverse proxy)

---

## Переменные окружения

Приложение использует следующие переменные, которые необходимо добавить в секрет

```
SECRET_KEY: Django secret key
DEBUG: режим отладки
ALLOWED_HOSTS: список разрешённых хостов
HOST: Хост - выданный вам домен
DJANGO_SUPERUSER_EMAIL: Логин Супер юзера
DJANGO_SUPERUSER_PASSWORD: Пароль Супер Юзера
DJANGO_SUPERUSER_USERNAME: Почта Супер Юзера
```

Создайте в папке **/deploy/yc-sirius-dev/dev-django-site-2/manifests** манифест секрета **secret.yaml** и вставьте значения переменных в необходимые поля, предварительно перекодировав значения в **base64**

```bash
echo -n 'you_env' | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: "bXktc3VwZXItc2VjcmV0LWtleQ=="
  DEBUG: 'RmFsc2U='
  ALLOWED_HOSTS: "ZWR1LXZpY3Rvci1zaGlzaGthbG92LnljLXNpcml1cy1kZXYucGVsaWQudGVhbQ=="
  HOST: 'ZWR1LXZpY3Rvci1zaGlzaGthbG92LnljLXNpcml1cy1kZXYucGVsaWQudGVhbQ=='
  DJANGO_SUPERUSER_USERNAME: dmljdG9y
  DJANGO_SUPERUSER_EMAIL: YWRtaW5AZXhhbXBsZS5jb20=
  DJANGO_SUPERUSER_PASSWORD: MTIzNA==
```

Примените манифест секрета 

```bash
kubectl apply -f secret.yaml -n <YOU-Namespace>
```

---

## Получение сертификата и его добавление в кластер

Скачиваем [сертефекат](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect/)

```bash
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0655 ~/.postgresql/root.crt
```

Создаем секрет с SSL-сертификатом для подключения к нашей базе данных PostgresSQL

```bash
kubectl create secret generic pg-ssl-cert \
  --from-file=root.crt=$HOME/.postgresql/root.crt \
  -n <YOU-NAMESPACE>
```

Проверка

```bash
kubectl get secrets -n <YOU-NAMESPACE>
```

Важные особенности

* SSL-сертификат PostgreSQL монтируется через Kubernetes Secret
* Сертификат НЕ хранится внутри Docker-образа
* При смене сертификата пересборка образа не требуется

---

## Сборка и публикация Docker-образов для дальнейшей установки приложения в кластер

Для dev-окружения Docker-образы публикуются в Docker Hub с тегом, равным короткому хэшу git-коммита.

### Сборка и публикация текущей версии

```bash
git rev-parse --short HEAD ## Получаем короткий хэш текущего коммита

cd backend_main_django

docker build -t dev-django-site:<commit_hash> .

docker tag dev-django-site:<commit_hash> <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>

docker push <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>
```

Проверка загрузки

```bash
docker pull <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>
```

---

## Деплой сайта

### Применим манифест деплоймента и сервиса

```bash
cd ./deploy/yc-sirius-dev/dev-django-site-2/manifests
kubectl apply -f django-deployment.yaml
kubectl apply -f django-service.yaml
```

Проверка деплоймента

```bash
kubectl get pods
kubectl get deployments
```

Проверка логов

```bash
kubectl logs <pod-name>
```

### Настройка NGINX

Заменим настройки Config Maps

```yaml
      user nginx;
      worker_processes  2;
      events {
        worker_connections  10240;
      }
      http {
        server {
            listen       80;
            server_name  localhost;
            location / {
                proxy_pass http://django-app-service:80;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
      }
```

Перезагрузить под **Nginx-main**

### Запуск management-команд

Применим миграции

```bash
kubectl apply -f django-migrate.yaml
```

Создадим суперпользователя

```bash
kubectl apply -f django-superuser.yaml
```

---

После успешного деплоя:

* сайт доступен по [https://edu-victor-shishkalov.yc-sirius-dev.pelid.team/](домену)
* админка открывается
* можно авторизоваться под суперпользователем

---

## Обновление приложения

1. Собрать новый Docker-образ
2. Запушить его в Docker Hub
3. Обновить тег образа в `django-deployment.yaml`
4. Применить изменения:

```bash
kubectl apply -f django-deployment.yaml
```

Или перезапустить деплоймент:

```bash
kubectl rollout restart deployment django-app-deployment -n <YOU-NAMESPACE>
```

---

## Если что-то сломалось:

### Логи приложения

```bash
kubectl logs <pod-name> -n <YOU-NAMESPACE>
```

### Описание pod

```bash
kubectl describe pod <pod-name> -n <YOU-NAMESPACE>
```

### Подключение внутрь контейнера

```bash
kubectl exec -it <pod-name> -n <YOU-NAMESPACE> -- bash
```

### Проверка базы данных

Убедитесь, что SSL-сертификат доступен:

```bash
ls /certs
```
