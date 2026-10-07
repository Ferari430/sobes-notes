От всех рекомендация можно отходить. Вы же инженер со своей волей и понимаете что делаете. Верно?)
# Общие рекомендации
- Надо базово разбираться в GNU/Linux
- Посмотрите плейлист, чтобы понимать основное: [romnero](https://youtu.be/O8N1lvkIjig?si=cJhoiR4nmseUwOS8)
- Прочитай эти статьи:
	- https://labs.iximiuz.com/tutorials/pitfalls-of-from-scratch-images
	- https://labs.iximiuz.com/tutorials/gcr-distroless-container-images - distroless must have
	- https://docs.docker.com/reference/dockerfile/ - чтобы понимать основные инструкции при написание докерфайлов.
	- https://docs.docker.com/build/building/best-practices/
	- Недавно вышла статья про DHI [ссылка](https://t.me/poxek/5850)
	- https://www.container-security.site/
- При написание докерфайлов, надо понимать зависимости приложения(что ему надо для работы, что надо перенести в следующий stage). В финальном образе не должно быть лишнего (что не нужно для работы приложения). Поэтому используй multi-stage.
- Для финального слоя можно использовать образы от гугл или chainguard: [пример](https://github.com/GoogleContainerTools/distroless)
- Почитай, что такое динамически и статистически слинкованное приложение.
- В ENTRYPOINT и CMD используй exec формат. Посмотри в чём различия exec и shell формата команд.
- Не используй alpine, т.к. там musl, а не glibc.
- Не используй ADD, т.к. неочевидная логика работы.
- Используй эту утилиту при написание(помогает выявить ошибки): [ссылка](https://github.com/wagoodman/dive)
- Обновление индекса пакетов и скачивание пакета должно быть в одном слое(RUN), и очищайте кеш в этом же слое.
- Создавай пользователя и назначай его владельцем вашего кода. (переноси /etc/group и /etc/passwd).
- Используй buildkit, т.к. он более современный.
- Для создания одного слоя в финальном стейдже можно формировать rootfs:
```Dockerfile
...
#---------------------------------------------------------------
FROM scratch AS rootfs
WORKDIR /rootfs
COPY --from=build /opt/openssl/bin/ bin/
COPY --from=build /opt/openssl/lib64/ lib/
COPY --from=build \
     /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 \
     lib64/
COPY --from=build \
     /lib/x86_64-linux-gnu/libc.so.6 \
     lib/
COPY --from=build \
    /opt/openssl/include/ usr/include/
#---------------------------------------------------------------
FROM scratch AS final
LABEL org.opencontainers.image.base.name="scratch"
COPY --from=rootfs /rootfs /
```
- Используй .dockerignore, т.к. так ты уменьшаешь контекст, ускоряешь загрузку контекста, не добавляешь лишнее при COPY: [ссылка1](https://docs.docker.com/build/concepts/context/#dockerignore-files)
- Используй максимально точный тег. Пример: не **golang:1.25**, а **golang:1.25.6-trixie**. Так будет более точная воспроизводимость.
- Проверяйте образы на уязвимости: [grype](https://github.com/anchore/grype), [trivy](https://github.com/aquasecurity/trivy)
- С помощью ldd можно проверить зависимости бинарника. А вот так можно скопировать все библиотеки бинарника
```Dockerfile
RUN mkdir /bin-libs \
    &&  cp -v $(ldd /path/to/bin | cut -s -d'>' -f2 | sed 's/ (.*//') /bin-libs/
```
## Лейблы
Советую использовать этот формат лейблов: [ссылка](https://github.com/opencontainers/image-spec/blob/v1.1.1/annotations.md) 
Лейблы помогают идентифицировать образ.
Думаю, можно начать с этих лейблов:
```
org.opencontainers.image.authors
org.opencontainers.image.created
org.opencontainers.image.version
org.opencontainers.image.commit
```
# Рекомендации по языкам
Надо много практики, чтобы по без ГПТ писать хорошо. Так что практикуйся. Можно брать любой проект с github и пытаться написать для него хороший докерфайл.
## Golang
Голанг позволяет собирать статистически слинкованное приложение. Это значит, что можно скопировать только бинарник в scratch образ и всё будет работать(не всегда [ссылка](https://labs.iximiuz.com/tutorials/pitfalls-of-from-scratch-images) ).
Мы используем -ldflags="-s -w", чтобы уменьшить размер бинарника (это никак не влияет на работоспособность).
Важно задавать переменные окружения при компиляции:
CGO_ENABLED
GOOS
GOARCH
Варианты можно посмотреть [ссылка](https://github.com/golang/go/blob/master/src/cmd/dist/build.go) или [тут](https://github.com/golang/go/blob/master/src/internal/syslist/syslist.go)
### Пример
```Dockerfile
FROM golang:1.25.6-bookworm AS builder

ARG PROJECT_VENDOR
ARG PROJECT_NAME
ARG PROJECT_VERSION
ARG PROJECT_COMMIT_ID

WORKDIR /app

COPY src src
# Можно ещё типо зависимости закешировать
RUN cd src && go mod dowload

RUN && apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
        tzdata \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* 

ENV CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    LDFLAGS_COMMON="-s -w" \
    LDFLAGS_VERSION="-X github.com/${PROJECT_VENDOR:?}/${PROJECT_NAME:?}/cmd.CopyrightYear=2023 \
    -X github.com/${PROJECT_VENDOR}/${PROJECT_NAME}/cmd.ReleaseTag=${PROJECT_VERSION:?} \
    -X github.com/${PROJECT_VENDOR}/${PROJECT_NAME}/cmd.CommitID=${PROJECT_COMMIT_ID:?}"

WORKDIR /app/src

RUN go build -a -o ./bin/minio -ldflags "${LDFLAGS_COMMON} ${LDFLAGS_VERSION}" ./main.go

#-------------------------------------------------------------------------------

FROM scratch

LABEL org.opencontainers.image.base.name="scratch"

COPY --from=builder /app/src/bin/ /bin/
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# create dirs for non-root executing
RUN --mount=from=busybox:1.36.1-glibc,src=/bin,dst=/bin \
    mkdir -m 777 /tmp /minio.data  

ENV TZ=UTC

EXPOSE 9000

ENTRYPOINT ["minio"]
CMD [ "server", "/minio.data"]
```
## Python
Для работы питон приложения нужен: интерпретатор(с библиотеками), ваш код, зависимости. 
Зависимости скачиваются в:
Без venv
1) `/usr/local/lib/pythonX.Y/site-packages/` 
2) Иногда и в `/usr/lib/pythonX.Y/dist-packages/` - если пакеты ставятся через apt.
С venv
- `<venv>/lib/pythonX.Y/site-packages/`
Важно, чтобы версия питона была одинаковая в патч версии по semver для `builder` и `final`. Вероятно, вам надо будет собирать свой дистролесс образ питона)
```
pip install --no-cache-dir -r requirements.txt
```
Выставляем переменные окружения:
	PYTHONDONTWRITEBYTECODE=1
		Запрещает Python писать **байткод** (`.pyc`) и создавать папки `__pycache__`.
	PYTHONUNBUFFERED=1
		Отключает буферизацию stdout/stderr для Python.
### Пример
```Dockerfile
FROM python:3.13.11-slim-bookworm AS builder

FROM gcr.io/distroless/python3-debian12:nonroot AS final
```
### Pip
TODO
### Poetry
TODO
### UV
TODO
## JS
### npm
TODO
### yarn
TODO
### lerna
TODO
## Java
TODO
## Rust
TODO
## PHP
TODO
## .NET
TODO