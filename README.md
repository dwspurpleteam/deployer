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

  subgraph calls_stage["Calls (stage)"]
    stage[NGINX] --> stage_meet[OpenVidu Meet]
    stage --> stage_openvidu[OpenVidu LiveKit]
    stage_meet --> stage_openvidu
    stage_openvidu -->|media / events| stage_streamapi[streamapi]
  end

  subgraph calls_prod["Calls (prod)"]
    proxy["NGINX"] --> meet[OpenVidu Meet]
    proxy --> openvidu[OpenVidu LiveKit]
    meet --> openvidu
    openvidu -->|media / events| streamapi
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
  streamapi --> nats
  stage_streamapi --> nats
```
