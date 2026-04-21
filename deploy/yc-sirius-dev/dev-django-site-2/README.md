## Сборка и публикация Docker-образов для дальнейшей установки приложения в кластер

Для dev-окружения Docker-образы публикуются в Docker Hub с тегом, равным короткому хэшу git-коммита.

### Сборка и публикация текущей версии

```bash

git rev-parse --short HEAD

cd backend_main_django

docker build -t dev-django-site:<commit_hash> .

docker tag dev-django-site:<commit_hash> <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>

docker push <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>
```

Проверка загрузки

```bash
docker pull <YOU_DOCKER_HUB_REPO>/dev-django-site:<commit_hash>
```

