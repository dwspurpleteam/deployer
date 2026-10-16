<p align="right">
  <img align="left" src="https://hits.sh/github.com/dwspurpleteam/deployer.svg?style=flat&amp;label=views&amp;color=007ec6" alt="views" />
  powered by <a href="https://github.com/dmironenko/composer">composer</a>
</p>

# Deployer

Инфраструктура сервиса плагинов для OpenVidu

## Архитектура

```mermaid
flowchart LR
  browser[Client] -->|HTTPS| ingress[NGINX]
  ingress -->|default| proxy
  ingress -->|stage cookie| stage
  browser <-->|WebRTC / TURN| openvidu
  browser <-->|WebRTC / TURN| stage_openvidu
  plugin_backend[Plugin backend] <-->|grpc / TLS| ingress
  ingress <-->|grpc, default| proxy
  ingress <-->|grpc, stage cookie| stage

  subgraph calls_stage["Calls (stage)"]
    stage[NGINX] --> stage_meet[OpenVidu Meet]
    stage --> stage_openvidu[OpenVidu LiveKit]
    stage_meet --> stage_openvidu
    stage_openvidu -->|webhooks, s2s| stage
    stage -->|webhooks / grpc| stage_streamapi["streamapi (1 replica)"]
    stage_openvidu -->|media| stage_streamapi
  end

  subgraph calls_prod["Calls (prod)"]
    proxy["NGINX"] --> meet[OpenVidu Meet]
    proxy --> openvidu[OpenVidu LiveKit]
    meet --> openvidu
    openvidu -->|webhooks, s2s| proxy
    proxy -->|webhooks / grpc| streamapi["streamapi (2 replicas)"]
    openvidu -->|media| streamapi
  end

  subgraph storage["Storage (prod)"]
    mongodb[MongoDB]
    redis[Redis]
    minio[MinIO]
    nats["NATS JetStream"]
  end

  meet --> mongodb
  meet --> redis
  meet --> minio
  stage_meet --> mongodb
  stage_meet --> redis
  stage_meet --> minio
  openvidu --> redis
  streamapi <-->|events| nats
  stage_streamapi <-->|events| nats
```

Разработчики плагинов подключаются к grpc по TLS через `calls.vareonix.cc:443`; ingress проксирует подписки на медиа и события в nginx выбранного окружения. Webhooks для streamapi и служебные HTTP-ручки доступны только внутри инфраструктуры. Подписки одного звонка направляются по `x-call-id` на одну реплику. После обрыва grpc stream клиент подписывается заново

## Доступ на stage

Откройте [звонки](https://calls.vareonix.cc/meet/) и выполните в консоли браузера `document.cookie = "composer_stage=<secret>:stage; Path=/; Secure; SameSite=Lax"; location.reload();`. Секрет запросите у ответственного за инфраструктуру: это расшифрованное значение `app.stage_token` из [конфига ingress](apps/ingress/nginx/vars/base.yaml)

Для grpc на stage передавайте metadata `cookie: composer_stage=<secret>:stage`; без нее запросы идут в prod. Для привязки подписок передавайте `x-call-id` со значением `call_id` запроса
