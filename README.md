# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

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

## Как запустить сайт через Minikube

Установите [VirtualBox](https://www.virtualbox.org/).

Установите [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и поднимите кластер [Minikube](https://minikube.sigs.k8s.io/docs/drivers/virtualbox/) на VirtualBox:

```
minikube start --vm-driver=virtualbox
```

Разверните PostgreSQL в кластере с помощью [Helm](https://helm.sh/):

```
choco install kubernetes-helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres-db bitnami/postgresql \
  --set auth.database=dbname \
  --set auth.username=username \
  --set auth.password=userpassword \
  --set primary.persistence.enabled=true \
  --set primary.persistence.size=10Gi
```

Создайте файл `django-secret.yaml`, по образцу `django-secret-template.yaml` и заполните в нем значения переменных `SECRET_KEY`, `DATABASE_URL`, `DEBUG` и `ALLOWED_HOSTS`.

Задав значения переменных, выполнить команды:

```sh
kubectl apply -f django-secret.yaml
kubectl apply -f django-migrate.yaml
kubectl apply -f deployment-django-app.yaml
kubectl apply -f service-django-app.yaml
kubectl apply -f ingress.yaml
kubectl apply -f django-clearsessions.yaml
```

## Как задеплоить код

Пример сайта [по ссылке](https://edu-anna-veselova.sirius-k8s.dvmn.org/).
Данные для доступа к инфраструктуре сайта [по ссылке](https://sirius-env-registry.website.yandexcloud.net/edu-anna-veselova.html).

Подключитесь к кластеру Kubernetes. Пример подключения для [Yandex Cloud](https://yandex.cloud/ru/docs/managed-kubernetes/operations/connect/#kubectl-connect).

Во всех манифестах директории деплоя `yc-sirius-k8s` укажите нужный `namespace`. В манифесте `service-django-app.yaml` укажите `nodePort`, на который настроен ALB-роутинг. 
Создайте файлы `django-secret.yaml` и `django-config.yaml` по образцу `django-secret-template.yaml` и `django-config-template.yaml` соответственно, заполните в них значения переменных `SECRET_KEY`, `DEBUG` и `ALLOWED_HOSTS`.
Для подключения к PostgreSQL необходимо проверить наличие или создать `Secret` с SSL-сертификатом root.crt. Укажите имя этого `Secret` во всех манифестах из директории `yc-sirius-k8s`, где используется том `pg-cert-volume` в поле `secretName` (по умолчанию назван `pg-root-cert`).

Выполните команды:

```sh
kubectl apply -f django-secret.yaml
kubectl apply -f django-config.yaml
kubectl apply -f django-migrate.yaml
kubectl apply -f deployment-django-app.yaml
kubectl apply -f service-django-app.yaml
kubectl apply -f django-clearsessions.yaml
```

### Как создать суперпользователя Django
Если требуется создать суперпользователя Django, создайте файл `django-superuser-secret.yaml` по образцу `django-superuser-secret-template.yaml` и заполните в нём значения переменных `SUPERUSER_USERNAME`, `SUPERUSER_EMAIL` и `SUPERUSER_PASSWORD`. Также укажите `namespace` и `secretName` тома `pg-cert-volume` в `django-createsuperuser.yaml`.

Выполните команды:

```sh
kubectl apply -f django-superuser-secret.yaml
kubectl apply -f django-createsuperuser.yaml
kubectl wait --for=condition=complete job/django-createsuperuser-job --timeout=300s
kubectl delete secret django-superuser-secret
```

### Как сменить версию приложения
По умолчанию устанавливается последняя версия образа `anneg8/django_app`. Если нужно установить другую версию приложения, укажите в манифесте `deployment-django-app.yaml` соответстующий тег [образа](https://hub.docker.com/repository/docker/anneg8/django_app/general) и выполните команду

```sh
kubectl apply -f deployment-django-app.yaml
```

Для проверки успешности обновления можно запустить команду

```sh
kubectl rollout status deployment deployment-django-app
```
