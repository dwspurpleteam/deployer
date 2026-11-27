<p align="right">
  <img align="left" src="https://hits.sh/github.com/dwspurpleteam/deployer.svg?style=flat&amp;label=views&amp;color=007ec6" alt="views" />
  powered by <a href="https://github.com/dmironenko/gitsail">gitsail</a>
</p>

# Deployer

Инфраструктура сервиса плагинов для OpenVidu

## Архитектура

```mermaid
flowchart LR
  browser[Client] -->|интерфейс и api звонков / https| proxy
  browser <-->|аудио и видео / webrtc, turn| openvidu

  subgraph calls_prod["Calls (prod)"]
    proxy[NGINX] -->|интерфейс и api звонков| meet[OpenVidu Meet]
    proxy -->|сигнализация звонков| openvidu[OpenVidu LiveKit]
    proxy -->|события и подписки / http, grpc| streamapi[Stream API]
    proxy -->|активация и конфигурация плагинов| manager[Plugin Manager]
    proxy -->|каталог плагинов| registry[Plugin Registry]
    meet -->|управление звонками и участниками| openvidu
    meet -->|активация плагинов| manager
    openvidu -->|события звонков / webhook| meet
    openvidu -->|события звонков / webhook| proxy
    streamapi <-->|получение медиа и управление звонком| openvidu
    streamapi -->|проверка прав / grpc| manager
    manager -->|манифесты плагинов| registry
  end

  subgraph attendance_prod["Attendance (prod)"]
    attendance_nginx[NGINX] -->|интерфейс и api плагина| attendance[Attendance]
  end

  browser -->|интерфейс и api плагина / https| attendance_nginx
  manager -->|активация и проверка состояния| attendance_nginx
  attendance -->|подписка на события / grpc| proxy

  subgraph storage["Storage (prod)"]
    mongodb[MongoDB]
    redis[Redis]
    minio[MinIO]
    nats["NATS JetStream"]
    postgres[PostgreSQL]
  end

  meet -->|комнаты и пользователи| mongodb
  meet -->|состояние и блокировки| redis
  meet -->|файлы| minio
  openvidu -->|состояние звонков| redis
  manager -->|кеш манифестов| redis
  streamapi <-->|хранение и доставка событий| nats
  registry -->|данные каталога| postgres
  manager -->|активации и права| postgres
```

Разработчики плагинов подключаются к grpc по TLS через `calls.vareonix.cc:443`; ingress проксирует подписки на медиа и события в nginx выбранного окружения. Webhooks для streamapi и служебные HTTP-ручки доступны только внутри инфраструктуры. Подписки одного звонка направляются по `x-call-id` на одну реплику. После обрыва grpc stream клиент подписывается заново

## Доступ на stage

Откройте [звонки](https://calls.vareonix.cc/meet/) и выполните в консоли браузера `document.cookie = "gitsail_stage=<secret>:stage; Path=/; Secure; SameSite=Lax"; location.reload();`. Секрет запросите у ответственного за инфраструктуру: это расшифрованное значение `app.stage_token` из [конфига ingress](apps/ingress/nginx/vars/base.yaml)

Для grpc на stage передавайте metadata `cookie: gitsail_stage=<secret>:stage`; без нее запросы идут в prod. Для привязки подписок передавайте `x-call-id` со значением `call_id` запроса

## Плагин посещаемости

Приложение [`apps/calls/attendance`](apps/calls/attendance) собирает backend и frontend из закреплённого коммита репозитория `plugins`. GitSail разворачивает prod из `main`; stage для посещаемости отключён через `compose.x-gitsail.enabled: false`. Сборка из исходников сохранена, поскольку CI плагина пока не публикует образ в GHCR.

Адрес: `https://calls.vareonix.cc/apps/plugin-attendance/?callId=RM_…`. Пути `/apps/plugin-<name>/` предназначены для интерфейсов плагинов; `/plugins` остаётся каталогом реестра. Cookie окружения выбирает prod или stage; backend каждого окружения подписывается на свой streamapi через внутренний nginx. Проверки, обновление версии и откат описаны в [инструкции](docs/attendance-deployment.md).
