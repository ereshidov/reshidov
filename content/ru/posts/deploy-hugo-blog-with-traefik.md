+++
date = '2025-04-15T01:47:01+05:00'
draft = false
title = 'Деплоим блог на Hugo с помощью Traefik'
+++

> Репозиторий с кодом - https://github.com/ereshidov/reshidov/tree/setup-deployment

**Дисклеймер**: В целом задача раздавать статику достаточно простая и банальная и тут можно (и даже нужно наверное) не использовать Traefik. Но мне захотелось на простом примере показать базовые его возможности.

Основная фишка _Traefik_ - автоматическое обнаружение контейнеров и настройка маршрутизации на основе их меток. В отличие от _Nginx_, где конфигурации нужно прописывать вручную.

### Создаем блог

Для создания блога буду использовать _Hugo_. Мне понравилась скорость его работы и большое количество тем из коробки. Очень просто настраивается и можно сконцентрироваться на основном - контенте.

1. Установим hugo - https://gohugo.io/installation/

```bash
brew install hugo
```

2. Создаем новый проект

```bash
hugo new site <site-name>
cd <site-name>
```

3. Я выбрал тему для блога hugo-blog-awesome. Настроим её.

```bash
hugo mod init github.com/USER/REPO
hugo mod get github.com/hugo-sid/hugo-blog-awesome
```

4. Подготовлю _favicon_ для блога. Для этого использую _[realfavicongenerator](https://realfavicongenerator.net/)_. Результат генерации переношу в `assets/icons`.

5. _Hugo_ конфигурируется с помощью отдельного файла - `hugo.toml`. Добавим основные элементы конфигурации. Пример с конфигурацией и подробным объяснением каждого поля можно посмотреть у автора в репозитории. После конфигурации, у меня получился следующий файл

```toml
# Базовый урл, ниже покажу как создать такой поддомен в duckdns.org
baseURL = 'https://reshidov.duckdns.org'

# Указываю основной язык для блога
languageCode = 'ru-RU'
defaultContentLanguage = "ru-RU"

# Title будет отображаться в названии вкладки браузера
title = 'Reshidov Blog'

# Указываем тему, которую будем использовать
[module]
	[[module.imports]]
		path = "github.com/hugo-sid/hugo-blog-awesome"

# Ссылки на социальные сети
[[params.socialIcons]]
name = "github"
url = "https://github.com/ereshidov"

[[params.socialIcons]]
name = "telegram"
url = "https://t.me/newfaceof"

# avatar.png размещен в папке assets/
[Languages.ru-RU.params.author]
	avatar = "avatar.png"
	intro = ""
	name = "Edem Reshidov"
	description = "Иногда пишу, чаще не пишу"
```

Запустим локально, чтобы посмотреть промежуточный результат:

```bash
hugo server
```

### Доставляем блог на сервер

Для следующего шага нам понадобится:

- _VPS_ - Использую самый дешевый вариант от _perfect quality hosting_

- Домен - Использую поддомен от _duckdns.org_

Приобретаем доступ к _VPS_ и подключаемся к нему. О первоначальной настройке _VPS_ есть замечательная _[статья](https://www.kkyri.com/p/how-to-secure-your-new-vps-a-step-by-step-guide)_, рекомендую ознакомиться, в ней покрыты базовые вещи.

```bash
ssh <user>@<ip_address>
```

После подключения, сразу создадим необходимую сеть для _docker_, которая будет использоваться позже.

```bash
docker network create traefik
```

Идем в _duckdns.org_ и создаем поддомен. Я выбрал _reshidov.duckdns.org_. В поле _ip address_ необходимо указать адрес, который вы получили после покупки доступа к _VPS_ серверу.

Теперь перейдем к основному - настройке `docker-compose.yml` файла. В первую очередь настроим _Traefik_.

Создаем отдельный сервис _traefik_ . Входная точка (entrypoint) с именем `web` будет слушать весь _http_ трафки на порту 80. В качестве провайдера указываем _docker_ - все сервисы будут конфигурироваться на основе меток (_labels_). В этом и заключается основное отличие от nginx, где нам приходилось бы императивно указывать конфигурацию.

```yml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    command:

      - "--entrypoints.web.address=:80"
      - "--providers.docker"

Смонтируем сокет Docker, чтобы Traefik мог взаимодействовать с Docker API.

    ports: # Пробрасываем порты из контейров наружу
      - "80:80"
      - "8080:8080"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

    networks:
      - traefik
```

Мы настроили _Traefik_. Но нам нужно еще добавить отдельный сервис, в котором мы будем раздавать статику с нашего блога. Сервис построим на основе легковесного образа _nginx:alpine_. И смонтируем результат сборки нашего приложения внутрь nginx

```yml
static-server:
  image: nginx:alpine

  container_name: blog-static
  volumes:
    - ./public:/usr/share/nginx/html:ro
```

Настроим сервис _static-server_ с помощью _labels_ . Нам потребуется явно указать, чтобы данный сервис использовал traefik, а так же создадим _Traefik Route_ в котором укажем, что хотим перенаправлять весь трафик приходящий на host - _reshidov.duckdns.org_ на наш порт 80

```yml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.blog.rule=Host(reshidov.duckdns.org)"
      - "traefik.http.routers.blog.entrypoints=web"
      - "traefik.http.services.blog.loadbalancer.server.port=80"
docker-compose.yml файл к этому моменту:

version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--entrypoints.web.address=:80"
      - "--providers.docker"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

    networks:
      - traefik
    container_name: traefik
  static-server:
    image: nginx:alpine
    container_name: blog-static
    volumes:
      - ./public:/usr/share/nginx/html:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.blog.rule=Host(reshidov.duckdns.org)"
      - "traefik.http.routers.blog.entrypoints=web"
      - "traefik.http.services.blog.loadbalancer.server.port=80"
    networks:
      - traefik

networks:
  traefik:
    external: true
```

Блог готов к публикации, подготовим workflow для доставки его на сервер

```yml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - master
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-22.04
    concurrency:
      group: ${{github.workflow}}-${{github.ref}}
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.119.0"
          extended: true
      - name: Build
        run: hugo --minify
      - name: Copy build result to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "./public,./docker-compose.yml"
          target: "./blog"
          overwrite: true
      - name: Run docker compose
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{secrets.HOST}}
          username: ${{secrets.USERNAME}}
          key: ${{secrets.SSH_PRIVATE_KEY}}
          script: |
            cd ./blog
            docker compose stop
            docker compose up -d
```

Создаем отдельную ветку и пушим наш код. После того как пайплайн пройдет, мой блог будет доступен в интернете по адресу - *http://reshidov.duckdns.org*.

С _Traefik_ перейти на _https_ не составит труда, этого нужно будет добавить всего пару строк в static-server и traefik сервисы. Детальную документацию можно найти по ссылке.

В _traefik_ сервис добавляем следующие commands, для генерации сертификатов. Так же добавляем новый entrypoints который будет слушать запросы по 443 порту.

```yml
- "--entryPoints.websecure.address=:443"
- "--certificatesresolvers.myresolver.acme.tlschallenge=true"
- "--certificatesresolvers.myresolver.acme.email=<your-email>"
- "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
```

В _static-service_ необходимо будет добавить отдельный роут который будет обрабатывать запросы на websecure _entrypoint_ (указали шагом выше)

```yml
- "traefik.http.routers.blog-https.rule=Host(reshidov.duckdns.org)"
- "traefik.http.routers.blog-https.entrypoints=websecure"
- "traefik.http.routers.blog-https.tls.certresolver=myresolver"
```

Так же для нашего _entrypoint_ по умолчанию, который слушает запросы на 80 порту добавим принудительный редирект на https. По запросу на такой порт, пользователь будет получать 301 статус код.

```yml
- "traefik.http.routers.blog.middlewares=redirect-to-https"
- "traefik.http.middlewares.redirect-to-https.redirectscheme=https"
- "traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true"
```

Итоговый файл

```yml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--entrypoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--providers.docker"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=ereshidov.dev@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./letsencrypt:/letsencrypt
    networks:
      - traefik
    container_name: traefik
  static-server:
    image: nginx:alpine
    container_name: blog-static
    volumes:
      - ./public:/usr/share/nginx/html:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.blog.rule=Host(reshidov.duckdns.org)"
      - "traefik.http.routers.blog.entrypoints=web"
      - "traefik.http.routers.blog.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme=https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true"

      - "traefik.http.routers.blog-https.rule=Host(reshidov.duckdns.org)"
      - "traefik.http.routers.blog-https.entrypoints=websecure"
      - "traefik.http.routers.blog-https.tls.certresolver=myresolver"
    networks:
      - traefik

networks:
  traefik:
    external: true
```

Пушим изменения в репозиторий и ждем деплоя. После успешного деплоя блог должен быть доступен по адресу *https://reshidov.duckdns.org*.

Первая статья в моем блоге будет как раз о том, как я его задеплоил. Такая рекурсия. Пока.
