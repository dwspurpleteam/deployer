# Деплой плагина посещаемости

GitSail разворачивает `apps/calls/attendance` из `main` в prod. NGINX публикует страницу и API по адресу `https://calls.vareonix.cc/apps/plugin-attendance/`, удаляя префикс перед передачей запроса в `attendance:8081`. Порт контейнера на хост не публикуется.

## Сборка и обновление образа

Workflow `plugin-attendance` в репозитории `plugins` собирает frontend и backend в один JAR на Java 25 и выполняет тесты. После push в `main` общий workflow `image.yml` упаковывает готовый `backend/build/libs/plugin-attendance.jar` через `plugin-attendance/Dockerfile`, публикует образ `ghcr.io/dwspurpleteam/attendance:<commit SHA>` и отправляет событие `image` в deployer.

Workflow `images.yml` в deployer создаёт PR с обновлением `compose.services.attendance.image` в `apps/calls/attendance/vars/base.yaml`. После слияния PR GitSail применяет новую версию. Сборка из исходников на сервере не нужна; образ закрепляется полным SHA коммита.

Для CI в `plugins` должны быть настроены:

- Доступ репозитория к Maven-пакету `pluginsdk:plugin-sdk:1.1.0` в GitHub Packages. Gradle использует `GITHUB_ACTOR` и стандартный `GITHUB_TOKEN` с `packages: read`.
- Секрет `DEPLOYER_TOKEN`, который используется общим workflow для отправки события в репозиторий deployer.
- Доступ сервера GitSail к образу attendance в GHCR, как для остальных сервисов.

## Профили и порядок первого обновления

`app.env.SPRING_PROFILES_ACTIVE` выбирает настройки внутри JAR:

| Окружение | Профиль | StreamAPI | TLS |
|---|---|---|---|
| Prod | `prod` | `nginx:19090` | Отключён внутри Docker-сети |
| Stage | `stage` | `nginx-stage:19090` | Отключён внутри Docker-сети |

Оба серверных профиля слушают `0.0.0.0:8081`. Локальный профиль находится во внешней папке `backend/config` и в образ не попадает. Секрет конкретной активации поступает от Plugin Manager в `POST /control`; глобальный ключ менеджера плагину не нужен.

При первом переходе с прежней сборки соблюдайте порядок:

1. Слейте изменения attendance в `plugins/main` и дождитесь публикации образа.
2. Слейте созданный PR deployer с новым тегом образа. Существующие `ATTENDANCE_STREAMAPI_TARGET` и `ATTENDANCE_STREAMAPI_PLAINTEXT` позволяют новой сборке работать до переключения на профили.
3. Примените изменение deployer, которое задаёт `SPRING_PROFILES_ACTIVE` и удаляет эти дублирующие переменные.
4. Зарегистрируйте в Plugin Registry новую версию `1.0.1` по манифесту `plugin-attendance/manifests/manifest-v1.1_plugin-v1.0.1.json` из `plugins`.

В манифесте должны быть серверные адреса:

```text
frontend.url-web:    https://calls.vareonix.cc/apps/plugin-attendance/
backend.health-url: https://calls.vareonix.cc/apps/plugin-attendance/actuator/health
backend.control-url: https://calls.vareonix.cc/apps/plugin-attendance/control
```

Опубликованные версии Registry неизменяемы: прежнюю версию с URL локального туннеля заменяет новая запись версии. Манифест запрашивает `participant.joined`, `participant.left`, `call.ended` и задаёт `initial-state: []`.

## Проверка prod

С машины с доступом к серверу выполните без stage-cookie (в PowerShell используйте `curl.exe`):

```sh
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/actuator/health
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/api/info
curl -fsSI https://calls.vareonix.cc/apps/plugin-attendance/
```

Ожидается health `UP`, в info `source: streamapi`, версия `1.0.1` и target с `nginx:19090`, у страницы HTTP 200. Health подтверждает запуск приложения; подписку нужно проверить отдельно.

Создайте звонок и запросите активацию через Plugin Manager, заменив callId и idempotencyKey. Для новой попытки активации используйте новый ключ; повтор того же запроса с прежним ключом возвращает прежний activationId.

```sh
curl --fail-with-body --request POST 'https://calls.vareonix.cc/v1/activations' \
  --header 'Content-Type: application/json' \
  --data '{"callId":"RM_REPLACE_WITH_ROOM_SID","pluginId":"attendance","pluginVersion":"1.0.1","pluginRegistryId":"local","requestedBy":"postman","idempotencyKey":"REPLACE_WITH_NEW_UNIQUE_KEY"}'
```

`callId` — SID комнаты LiveKit вида `RM_...`. По возвращённому `activationId` проверьте состояние менеджера:

```sh
curl -fsS https://calls.vareonix.cc/v1/activations/REPLACE_WITH_ACTIVATION_ID
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/api/calls/RM_REPLACE_WITH_ROOM_SID/participants
```

Ожидается `ACTIVE` у менеджера и `TRACKING` у плагина. Менеджер сам доставляет ID и секрет в `/control`, плагин подписывается через Java SDK; вручную придумывать секрет не нужно. До активации API участников возвращает 404.

Откройте `https://calls.vareonix.cc/apps/plugin-attendance/?callId=RM_REPLACE_WITH_ROOM_SID`, затем выполните вход и выход проверяемого участника. Должны обновляться `present` и `left`. Начальный состав пока не обрабатывается, поэтому участники, вошедшие до подписки, могут отсутствовать. Открытие страницы не запускает подписку. Встраивание интерфейса в Meet настраивается отдельно.

Для диагностики на сервере:

```sh
docker ps -a --filter label=com.docker.compose.service=attendance
docker logs --tail 100 <container>
```

## Stage и откат

Stage остаётся отключённым через `compose.x-gitsail.enabled: false`. После отдельного включения окружения attendance использует профиль `stage`, адрес `attendance-stage:8081` и StreamAPI `nginx-stage:19090`. Для внешних запросов нужен cookie окружения `gitsail_stage=<secret>:stage`; значение секрета в Git не сохраняется. Проверяйте заголовок `X-GitSail-Environment` и target в `/api/info`.

Для отката верните предыдущий тег `compose.services.attendance.image`. При откате на сборку без серверных профилей верните и прежние переменные `ATTENDANCE_STREAMAPI_TARGET`/`ATTENDANCE_STREAMAPI_PLAINTEXT` из Git. Одна реплика обязательна: состояние хранится в памяти и теряется при перезапуске; для продолжения потребуется новая доставка команды активации менеджером.
