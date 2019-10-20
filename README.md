# Proto buffers

Все `.proto`-файлы, используемые нашими приложениями.

## Структура

Имена `message`, `service` и прочих сущностей должны быть уникальными среди всех `.proto`-файлов в любых директориях репозитория. Это обусловлено тем, что генератор кода `grpc_tools.protoc` будет ругаться на конфликт имён, если при импорте сущностей в `.proto`-файле появятся дубликаты.

Имя директории - это имя веб-сервиса. При этом конечное приложение, использующее сгенерированный код, может реализовывать API одновременно нескольких веб-сервисов.

Сущности `service` должны определяться в файле с именем `api.proto`. Имена этих сущностей должны иметь формат `<ИмяВебСервиса>Api`. Данный суффикс позволяет устранить возможный конфликт одинаковых названий `service` и `message`.

Сущности `message` могут определяться как в `api.proto`, так и `entities.proto` - это зависит от типа и назначения сущностей. В файле `api.proto` могут объявляться только те `message`, которые необходимы лишь для описания входящих и исходящих сообщений для определенных в нём `service`. В файле `entities.proto` должны определяться сущности, являющиеся понятиями предметной области, которыми оперирует конечный модуль, даже если такие сущности используются в качестве входящих/исходящих для `service`.

Сущности `message` и `enum`, не подходящие сами по себе под описанные выше критерии, но используемые другими сущностями `message`, должны определяться в том файле, в котором находятся использующие их сущности. А если таковые есть и в `api.proto`, и в `entities.proto`, то - в `entities.proto`.

Недопустимо импортировать файлы `api.proto` в файлы `entities.proto`. Следовательно, если какая-то сущность `message` использует сущность из файла `api.proto`, последняя должна быть перенесена в `entities.proto`.

## Процесс выпуска новой версии и использования её в приложении (release-process)

1. Проектирование и согласование изменений
1. Внесение изменений в `.proto`-файлы через ветку и приёмку пул реквеста
1. Слияние ветки с master после успешной проверки изменений через автоматические средства
1. Назначение очередной версии через tag на текущий коммит мастера
1. Указание нужной версии клиента/сервера в зависимом приложении
1. Скачивание указанной версии `.proto`-файлов и генерация файлов клиента/сервера из них в момент сборки образа зависимого приложения

Формат версии в tag: "v<порядковый номер>". В качестве порядкового номера используется либо ручной счётчик, либо номер билда из TeamCity.

## Сборка образа приложения

Данные инструкции скачивают `.proto`-файлы из репозитория `protos`:

```dockerfile
FROM alpine/git:1.0.7 AS protos-gitter

RUN mkdir /root/.ssh

# A private part of SSH key to access the protos repo
COPY protos-id_rsa /root/.ssh/id_rsa

ARG PROTOS_HOST=bitbucket.org
ARG PROTOS_DSN=git@${PROTOS_HOST}:globalforexsystems/protos.git
ARG PROTOS_BRANCH=master

RUN set -e; \
    chmod 600 /root/.ssh/*; \
    echo -e "Host ${PROTOS_HOST}\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config; \
    # Unfortunately there is a glitch when store result in the default /git directory
    mkdir /protos; \
    cd /protos; \
    git clone --depth 1 -b $PROTOS_BRANCH $PROTOS_DSN .; \
    rm -rf .git
```

Для их выполнения понадобится файл `protos-id_rsa` с приватной частью SSH-ключа, дающего доступ к репозиторию `protos`. Также при запуске сборки потребуется указать `--build-arg PROTOS_BRANCH=foo`, чтобы выбрать ветку (версию) репозитория `protos`, которая будет скачана.

После них нужно разместить инструкции для сборки целевого образа; например, Python-приложения:

```dockerfile
FROM python:3.7

RUN pip install pipenv

WORKDIR /var/app/src

COPY src/Pipfile* ./

RUN pipenv install --deploy --system

COPY src/ .

COPY --from=protos-gitter /protos ./protos
# Just for inject a different proto-files for development
#COPY protos ./protos

RUN set -e; \
    mkdir -p protos_out; \
    python -m grpc_tools.protoc -I. --python_out=protos_out ./protos/common.proto; \
    cp ./protos/common.proto ./protos_out/protos/common.proto ; \
    SERVICES="deal exchange-trading trade-account"; \
    for service in $SERVICES; do \
        python -m grpc_tools.protoc -I. --python_out=./protos_out \
            ./protos/$service/api.proto ./protos/$service/entities.proto; \
        python -m grpc_tools.protoc -I. --grpc_python_out=./protos_out \
            ./protos/$service/api.proto; \
        dir_name=$(echo $service | sed 's/-/_/g'); \
        cp ./protos/$service/*.proto ./protos_out/protos/$dir_name ; \
    done; \
    rm -rf protos; \
    mv protos_out/protos ./protos; \
    rm -rf protos_out; \
    echo "import os.path as osp \n\
import sys \n\
\n\
sys.path.append(osp.join(osp.dirname(osp.dirname(osp.abspath(__file__))), 'protos'))" > protos/__init__.py
```

Чтобы последнее заработало, нужно добавить пакет `grpcio-tools` в `Pipfile`. Также не забудьте в обоих наборах инструкций указать `LABEL maintainer="..."`

В результате выполнения этих наборов инструкций в директории `/var/app/src/protos` будут находиться поддиректории с именами веб-сервисов, например `exchange_trading`, в которых будут находиться `pb2`- и `.proto`-файлы:

```
/var/app/src/protos# ls -R
.:
__init__.py  common_pb2.py  common.proto  deal  exchange_trading

./deal:
api.proto  api_pb2.py  api_pb2_grpc.py  entities.proto  entities_pb2.py

./exchange_trading:
api.proto  api_pb2.py  api_pb2_grpc.py  entities.proto  entities_pb2.py
```

## Процесс разработки приложений

При разработке приложения может быть удобно получить сгенерированные файлы клиента и сервера из образа на локальный диск разработчика. Воспользуйтесь командами:

```shell script
cd $LOCAL_SERVICE_DIR/src/
docker run -d --rm $IMAGE bash -c 'sleep 10' | xargs -i{} docker cp {}:/var/app/src/protos - | tar -x
```

Добавьте эту строчку в `.gitignore`, чтобы случайно не закоммитить сгенерированные файлы в репозиторий приложения:

```gitignore
protos
```

В указанных выше инструкциях сборки образа есть комментарий, как использовать локальные `.proto`-файлы вместо хранящихся в репозитории `protos`.

Если при разработке приложения требуется внести изменения в репозиторий с `.proto`-файлами, то процесс должен быть следующим:

1. В репозитории с `.proto`-файлами создаётся новая ветка с желаемыми изменениями
1. В инструкциях сборки образа приложения в значении аргумента `PROTOS_BRANCH` указывается имя данной ветки
1. Весь контент ветки должен пройти согласование по [процессу выпуска новой версии](#release-process)
1. Когда контент ветки будет слит с master и будет назначен tag для новой версии, в инструкциях сборки указывается именно он

## Требования к содержимому `.proto`-файлов

1. Порядок импорта файлов:
    1. Находящиеся в библиотеках, например, `google/protobuf/empty.proto`
    1. Находящиеся в локальной директории модуля, например, `entities.proto`
    1. Находящиеся в других директориях проекта, например, `common.proto`
    1. В оставшихся случаях порядок определяется сортировкой по алфавиту без учёта регистра букв
1. Для пустых сообщений использовать тип `google.protobuf.Empty`
1. Для полей со временем использовать тип `google.protobuf.Timestamp`
1. Для полей с денежными суммами использовать тип `string`
