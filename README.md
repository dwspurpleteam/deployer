<p align="right">
  <img align="left" src="https://hits.sh/github.com/dwspurpleteam/deployer.svg?style=flat&amp;label=views&amp;color=007ec6" alt="views" />
  powered by <a href="https://github.com/dmironenko/gitsail">gitsail</a>
</p>

# Deployer

Инфраструктура сервиса плагинов для OpenVidu

## Архитектура

```mermaid
flowchart LR
  browser[Client] -->|HTTPS| proxy
  browser <-->|WebRTC / TURN| openvidu

  subgraph calls_prod["Calls (prod)"]
    proxy[NGINX] --> meet[OpenVidu Meet]
    proxy --> openvidu[OpenVidu LiveKit]
    proxy -->|HTTP / gRPC| streamapi[streamapi]
    proxy --> manager[Plugin Manager]
    proxy --> registry[Plugin Registry]
    meet --> openvidu
    meet --> manager
    openvidu -->|webhooks| meet
    openvidu -->|webhooks| proxy
    streamapi <-->|media / control| openvidu
    streamapi -->|gRPC permissions| manager
    manager --> registry
  end

  subgraph attendance_prod["Attendance (prod)"]
    attendance_nginx[NGINX] --> attendance[Attendance]
  end

  browser -->|HTTPS| attendance_nginx
  manager -->|activation / health| attendance_nginx
  attendance -->|gRPC events| proxy

  subgraph storage["Storage (prod)"]
    mongodb[MongoDB]
    redis[Redis]
    minio[MinIO]
    nats["NATS JetStream"]
    postgres[PostgreSQL]
  end

  meet --> mongodb
  meet --> redis
  meet --> minio
  openvidu --> redis
  manager --> redis
  streamapi <-->|events| nats
  registry --> postgres
  manager --> postgres
```

Разработчики плагинов подключаются к grpc по TLS через `calls.vareonix.cc:443`; ingress проксирует подписки на медиа и события в nginx выбранного окружения. Webhooks для streamapi и служебные HTTP-ручки доступны только внутри инфраструктуры. Подписки одного звонка направляются по `x-call-id` на одну реплику. После обрыва grpc stream клиент подписывается заново

## Доступ на stage

Откройте [звонки](https://calls.vareonix.cc/meet/) и выполните в консоли браузера `document.cookie = "gitsail_stage=<secret>:stage; Path=/; Secure; SameSite=Lax"; location.reload();`. Секрет запросите у ответственного за инфраструктуру: это расшифрованное значение `app.stage_token` из [конфига ingress](apps/ingress/nginx/vars/base.yaml)

Для grpc на stage передавайте metadata `cookie: gitsail_stage=<secret>:stage`; без нее запросы идут в prod. Для привязки подписок передавайте `x-call-id` со значением `call_id` запроса

## Плагин посещаемости

Приложение [`apps/calls/attendance`](apps/calls/attendance) собирает backend и frontend из закреплённого коммита репозитория `plugins`. GitSail разворачивает prod из `main`; stage для посещаемости отключён через `compose.x-gitsail.enabled: false`. Сборка из исходников сохранена, поскольку CI плагина пока не публикует образ в GHCR.

Адрес: `https://calls.vareonix.cc/apps/plugin-attendance/?callId=RM_…`. Пути `/apps/plugin-<name>/` предназначены для интерфейсов плагинов; `/plugins` остаётся каталогом реестра. Cookie окружения выбирает prod или stage; backend каждого окружения подписывается на свой streamapi через внутренний nginx. Проверки, обновление версии и откат описаны в [инструкции](docs/attendance-deployment.md).
