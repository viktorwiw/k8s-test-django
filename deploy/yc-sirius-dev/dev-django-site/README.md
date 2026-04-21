## Добавляем сертефекат в secret

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

После запуска pod сертификат будет доступен внутри контейнера:

```bash
/root/.postgresql/root.crt
```

Для проверки работоспособности сертификата, создадим тестовый под и запустим его

```bash
kubectl apply -f ubuntu-test.yaml -n <YOU-NAMESPACE>
kubectl exec -it ubuntu-pg-test-dev -n <YOU-NAMESPACE> -- bash
```

Установим **postgres**

```
apt update
apt install --yes postgresql-client
```

Подключаемся к запущенному поду и проверяем, что мы подключились к нашей БД

```bash
psql "host=<список_хостов_кластера> \
      port=6432 \
      sslmode=verify-full \
      dbname=<имя_БД> \
      user=<имя_пользователя> \
      target_session_attrs=read-write"
```

Где **host**,  **dbname**, **user** получаем из нашего секрета, с помощью Lens.



