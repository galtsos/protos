# Protocol buffers

Единое место хранения всех `.proto`-файлов, используемые нашими приложениями.

Следовательно, другие репозитории должны получать `.proto`-файлы именно из этого.

Совокупность всех `.proto`-файлов называется протоколом определенной версии.

## Структура файлов

Имя директории - это имя сервиса в нашей архитектуре и обычно должно совпадать с именем репозитория данного сервиса. Однако, конечное приложение, использующее протокол, может реализовывать API одновременно нескольких сервисов.

Сущности `service` должны определяться в файле с именем `api.proto`. Имена этих сущностей должны иметь формат `<ИмяВебСервиса>Api`. Данный суффикс позволяет устранить возможный конфликт одинаковых названий `service` и `message`.

Сущности `message` могут определяться как в `api.proto`, так и `entities.proto` - это зависит от типа и назначения сущностей. В файле `api.proto` могут объявляться только те `message`, которые необходимы лишь для описания входящих и исходящих сообщений для определенных в нём `service`. В файле `entities.proto` должны определяться сущности, являющиеся понятиями предметной области, которыми оперирует конечный сервис, даже если такие сущности используются в качестве входящих/исходящих для `service`.

Сущности `message` и `enum`, не подходящие сами по себе под описанные выше критерии, но используемые другими сущностями `message`, должны определяться в том файле, в котором находятся использующие их сущности. А если таковые есть и в `api.proto`, и в `entities.proto`, то - в `entities.proto`.

Недопустимо импортировать файлы `api.proto` в файлы `entities.proto`. Следовательно, если какая-то сущность `message` использует сущность из файла `api.proto`, последняя должна быть перенесена в `entities.proto`.

## Требования к содержимому `.proto`-файлов

Имена `message`, `service` и прочих сущностей должны быть уникальными среди всех `.proto`-файлов в любых директориях репозитория. Это обусловлено тем, что генератор кода `grpc_tools.protoc` будет ругаться на конфликт имён, если при импорте сущностей в `.proto`-файле появятся дубликаты.

1. Порядок импорта файлов:
    1. Находящиеся в библиотеках, например, `google/protobuf/empty.proto`
    1. Находящиеся в локальной директории сервиса, например, `entities.proto`
    1. Находящиеся в других директориях проекта, например, `common.proto`
    1. В оставшихся случаях порядок определяется сортировкой по алфавиту без учёта регистра букв
1. Для пустых сообщений использовать тип `google.protobuf.Empty`
1. Для полей со временем использовать тип `google.protobuf.Timestamp`
1. Для полей с денежными суммами использовать тип `string`

## Процесс выпуска новой версии протокола (release-process)

Действия в данном репозитории:

1. Проектирование и согласование изменений
1. Внесение изменений в `.proto`-файлы через новую ветку
1. Приёмка пул реквеста, прохождение автоматических проверок, слияние ветки с master
1. Указание новой версии протокола с помощью tag репозитория (обычно на merge-коммит ветки master) в формате "v<порядковый номер>"

В качестве порядкового номера используется либо ручной счётчик, либо номер билда из TeamCity.

Действия в приложении, использующем протокол:

1. Выбор необходимой для приложения версии протокола
1. Скачивание `.proto`-файлов данной версии
1. Генерация программных файлов клиента/сервера в момент сборки образа приложения

## Сборка образа приложения, использующего протокол

Данные инструкции скачивают `.proto`-файлы конкретной версии из репозитория `protos` (см. [процесс выпуска новой версии протокола](#release-process)):

```dockerfile
FROM alpine/git:1.0.7 AS protos-gitter

RUN mkdir /root/.ssh

# A private part of SSH key to access the protos repo
COPY protos-id_rsa /root/.ssh/id_rsa

ARG PROTOS_HOST=bitbucket.org
ARG PROTOS_DSN=git@${PROTOS_HOST}:globalforexsystems/protos.git
ARG PROTOS_BRANCH

RUN set -e; \
    chmod 600 /root/.ssh/*; \
    echo -e "Host ${PROTOS_HOST}\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config; \
    # Unfortunately there is a glitch when store result in the default /git directory
    mkdir /protos; \
    cd /protos; \
    git clone --depth 1 -b $PROTOS_BRANCH -- $PROTOS_DSN .; \
    rm -rf .git
```

Для их выполнения понадобится файл `protos-id_rsa` с приватной частью SSH-ключа, дающего доступ к репозиторию `protos`. Чтобы скачать необходимую версию протокола (тег/ветку репозитория), нужно указать её в значении аргумента сборки `PROTOS_BRANCH` либо прямо в указанных выше инструкциях (предпочтительно), либо в `docker build` в аргументе `--build-arg`.

Далее нужно разместить инструкции для сборки целевого образа; например, Python-приложения:

```dockerfile
FROM python:3.7

RUN pip install pipenv

WORKDIR /var/app/src

COPY src/Pipfile* ./

RUN pipenv install --deploy --system

COPY src/ .

COPY --from=protos-gitter /protos protos_orig/protos
# Just for inject a different proto-files for development
#COPY protos protos_orig/protos

RUN set -e; \
    cd protos_orig; \
    python -m grpc_tools.protoc -I. --python_out=.. protos/common.proto; \
    cp protos/common.proto ../protos/common.proto ; \
    SERVICES="deal exchange-trading"; \
    for service in $SERVICES; do \
        python -m grpc_tools.protoc -I. --python_out=.. \
            protos/$service/api.proto protos/$service/entities.proto; \
        python -m grpc_tools.protoc -I. --grpc_python_out=.. protos/$service/api.proto; \
        dir_name=$(echo $service | sed 's/-/_/g'); \
        cp protos/$service/*.proto ../protos/$dir_name ; \
    done; \
    rm -rf ../protos_orig
```

Чтобы последнее заработало, нужно добавить пакет `grpcio-tools` в `Pipfile`. Также нужно не забыть отредактировать список `SERVICES` в соответствии с нужными сервисами и в обоих наборах инструкций указать `LABEL maintainer="..."`.

В результате выполнения этих наборов инструкций в директории `/var/app/src/protos` будут находиться поддиректории с именами сервисов, например `exchange_trading`, в которых будут находиться `pb2`- и `.proto`-файлы:

```
/var/app/src/protos# ls -R
.:
common_pb2.py  common.proto  deal  exchange_trading

./deal:
api.proto  api_pb2.py  api_pb2_grpc.py  entities.proto  entities_pb2.py

./exchange_trading:
api.proto  api_pb2.py  api_pb2_grpc.py  entities.proto  entities_pb2.py
```

## Процесс разработки приложений, использующих протокол

При разработке приложения может быть удобно получить сгенерированные файлы клиента и сервера из образа на локальный диск разработчика. Воспользуйтесь командами:

```shell
cd $LOCAL_SERVICE_DIR/src/
docker run -d --rm $IMAGE bash -c 'sleep 10' | xargs -i{} docker cp {}:/var/app/src/protos - | tar -x
```

Добавьте эти строчки в `.gitignore`, чтобы случайно не закоммитить лишние файлы в репозиторий приложения:

```gitignore
protos
protos-id_rsa
```

В указанных выше инструкциях сборки образа есть комментарий, как использовать локальные `.proto`-файлы вместо хранящихся в репозитории `protos`.

Если при разработке приложения требуется внести изменения в репозиторий с `.proto`-файлами, то процесс должен быть следующим:

1. В репозитории `protos` создаётся новая ветка с желаемыми изменениями
1. В аргументе сборки `PROTOS_BRANCH` (репозиторий приложения) указывается имя данной ветки
1. Изменения протокола проверяются на пригодность для приложения
1. Ветка в репозитории `protos` проходит согласование по [процессу выпуска новой версии протокола](#release-process)
1. После этого в аргументе сборки `PROTOS_BRANCH` указывается новая версия протокола
