# Запуск приложения в кластере Minikube

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
Перед запуском приложения создадим контролер сервиса Ingress в нашем minikube кластере

```
minikube addons enable ingress
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

Запускаем наше приложения внутри кластера

```bash
minikube service django-service --url
```
В манифесте **ingress.yaml** замените в строке **- host: myapp.local** на ваш тестовый домен, затем добавьте ваш домен в файл (host)[https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit]

Применим манифест для Ingress и проверим что он создан

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

Откроем тунель для миникуб кластера, чтобы увидеть наше приложение в браузере, при переходе по нашему домену

```bash
minikube tunnel
```

---
### Регулярное удаление сессий

Для регулярной очистки сессий, примените манифест **django-clearsessions.yaml**, настроенный на регулярный запуск задачи 1 раз в месяц

```bash
kubectl apply -f django-clearsessions.yaml
```

Для проверки задачи, выполните по очереди команды

```bash
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-test
kubectl get jobs
kubectl get pods
kubectl logs <pod-name>
```
