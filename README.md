# Proto buffers

Все .proto файлы, используемые нашими приложениями.

## Структура

Имена `message`, `service` и прочих сущностей должны быть уникальными среди всех `.proto`-файлов в любых директориях репозитория. Это обусловлено тем, что генератор кода `grpc_tools.protoc` будет ругаться на конфликт имён, если при импорте сущностей в `.proto`-файле появятся дубликаты.

Имя директории - это имя веб-сервиса. При этом конечное приложение, использующее сгенерированный код, может реализовывать API одновременно нескольких веб-сервисов.

Сущности `service` должны определяться в файле с именем `api.proto`. Имена этих сущностей должны иметь формат `<ИмяВебСервиса>Api`. Данный суффикс позволяет устранить возможный конфликт одинаковых названий `service` и `message`. Кроме этого в файле `api.proto` могут объявляться только те `message`, которые необходимы лишь для описания входящих и исходящих сообщений для определенных в нём `service`.

Сущности `message`, которые не являются входящими и исходящими сообщения для `service`, должны определяться в файле с именем `entities.proto`.

## Процесс выпуска новой версии и использования её в приложении

1. Проектирование и согласование изменений
1. Внесение изменений в `.proto`-файлы через ветку и приёмку пул реквеста
1. Слияние ветки с master после успешной проверки изменений через автоматические средства
1. Указание нужной версии клиента/сервера в зависимом приложении
1. Скачивание указанной версии `.proto`-файлов и генерация файлов клиента/сервера из них в момент сборки образа зависимого приложения

## Сборка образа приложения

Данные инструкции скачивают `.proto`-файлы из репозитория `protos`:

```dockerfile
FROM alpine/git:1.0.7 AS protos-gitter

RUN mkdir /root/.ssh

# Put here SSH keys to access protos repo on Bitbucket
COPY ssh-keys/* /root/.ssh/

ARG PROTOS_BRANCH

RUN set -e; \
    chmod 600 /root/.ssh/*; \
    echo -e "Host bitbucket.org\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config; \
    # Unfortunately there is a glitch when store result in the default /git directory
    mkdir /protos; \
    cd /protos; \
    git clone --depth 1 -b $PROTOS_BRANCH git@bitbucket.org:globalforexsystems/protos.git .; \
    rm -rf .git
```

Для их выполнения понадобятся SSH-ключи в локальной директории `ssh-keys` для прохождения авторизации. Также при запуске сборки потребуется указать `--build-arg PROTOS_BRANCH=foo`, чтобы выбрать ветку (версию) репозитория `protos`, которая будет скачана.

После них нужно разместить инструкции для сборки целового образа; например, Python-приложения:

```dockerfile
FROM python:3.7

RUN pip install pipenv

WORKDIR /var/app/src

COPY Pipfile* ./

RUN pipenv install --deploy --system

COPY --from=protos-gitter /protos ./protos
# Just for inject a different proto-files for development
#COPY protos ./protos

RUN set -e; \
    cd protos; \
    python -m grpc_tools.protoc -I. --python_out=.. \
        market-info/api.proto market-info/entities.proto; \
    python -m grpc_tools.protoc -I. --grpc_python_out=.. \
        market-info/api.proto; \
    # Build another proto-files here
    rm -rf ../protos
```

Чтобы последнее заработало, нужно добавить пакет `grpcio-tools` в `Pipfile`. Также не забудьте в обоих наборах инструкций указать `LABEL maintainer="..."`

В результат выполнения этих наборов инструкций в директории `/var/app/src` будут находиться поддиректории с именами веб-сервисов, например `market-info`, в которых будут находиться только `pb2`-файлы:

```
/var/app/src# ls -aR
.:
Pipfile  Pipfile.lock  market_info

./market_info:
api_pb2.py  api_pb2_grpc.py  entities_pb2.py
```
