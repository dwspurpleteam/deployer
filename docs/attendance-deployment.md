# Деплой плагина посещаемости

GitSail разворачивает `apps/calls/attendance` из `main` в prod. NGINX публикует страницу и API по адресу `https://calls.vareonix.cc/apps/plugin-attendance/`, удаляя префикс при проксировании на `attendance:8081`. `/plugins` остаётся каталогом реестра. Порт плагина на хост не публикуется.

## Сборка и версия

`build.tag` закреплён на `5132cf88a9d6513f93d92229b09cc03552e96419` — доступный локально `plugins/main`. Dockerfile собирает frontend и backend в один jar на Java 25. В runtime копируется только jar, без локального `application-stage.yml` и cookie.

Остальные сервисы используют готовые образы GHCR, но доступный CI `plugins` пока только собирает и тестирует jar, без публикации Docker-образа. Поэтому attendance сохраняет сборку из исходников через GitSail. После добавления публикации образа можно заменить `build` и Dockerfile на `compose.services.attendance.image` с закреплённым SHA. Текущий workflow `images.yml` обновляет только поле `image` и не обновляет `build.tag`.

Для обновления измените `build.tag` в deployer. Пуш в `plugins` сам по себе не обновляет закреплённую версию на стенде.

Одна реплика обязательна: состояние звонков хранится в памяти и теряется при перезапуске attendance. LiveKit при этом не перезапускается.

## Ограничения перед выкладкой

Текущий prod streamapi включает проверку прав через Plugin Manager. Закреплённый attendance отправляет `call_id`, типы событий и `x-call-id`, но не передаёт контекст конкретной активации. Исходников SHA развёрнутого streamapi в локальном репозитории нет; совместимость с его проверкой прав не подтверждена. Перед слиянием проверьте реальную подписку на звонок. `UP` у health и HTTP 200 у `/api/info` подтверждают запуск процесса, но не доступ к событиям. При `UNAUTHENTICATED` или `PERMISSION_DENIED` нужна доработка интеграции в `plugins`, а затем обновление закреплённой версии. Не отключайте проверку прав streamapi для этого.

Конфигурация публикует прямой URL плагина. В `MEET_PLUGINS` остаётся прежний внешний URL; автоматическое появление рабочего окна посещаемости в Meet этим деплоем не обеспечивается. Каталог и активацию через платформу нужно проверять отдельно.

Result Gateway этим деплоем не запускается. По умолчанию плагин обращается к `localhost:8089` внутри своего контейнера, поэтому публикация сводки не работает. Для доступного Result Gateway задайте `ATTENDANCE_RESULT_GATEWAY_URL` и `ATTENDANCE_RESULT_GATEWAY_CALLBACK_URL` в `app.env`.

## Проверка prod

С машины с доступом к стенду выполните без stage-cookie (в PowerShell используйте `curl.exe`):

```sh
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/actuator/health
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/api/info
curl -fsSI https://calls.vareonix.cc/apps/plugin-attendance/
```

Ожидается health `UP`, в info `source: streamapi`, target с `nginx:19090`, у страницы HTTP 200. Во время сборки и запуска возможен временный 502. Отсутствие маршрута обычно даёт 404.

Создайте звонок в том же окружении и откройте плагин до входа проверяемого участника:

```text
https://calls.vareonix.cc/apps/plugin-attendance/?callId=RM_…
```

`callId` — SID комнаты LiveKit, а не имя комнаты. События до подписки не воспроизводятся. Вход и выход участника должны обновить список. Для диагностики можно несколько раз запросить состояние подписки:

```sh
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/api/calls/RM_REPLACE_WITH_ROOM_SID/participants
```

Проверяйте `status`, `statusMessage`, `trackingSince` и изменения `present`/`left`; ошибки прав нельзя считать успешной проверкой даже при HTTP 200.

На сервере:

```sh
docker ps -a --filter label=com.docker.compose.service=attendance
docker logs --tail 100 <container>
```

## Stage

Attendance на stage отключён через `compose.x-gitsail.enabled: false` в `apps/calls/attendance/vars/stage.yaml`, как и остальные stage-сервисы в текущем main. Для проверки до prod нужно отдельно включить attendance и необходимые ему nginx, streamapi, Plugin Manager, Plugin Registry, OpenVidu, Meet на stage в отслеживаемой ветке `dev`. Не переносите включение stage в prod PR без необходимости.

Stage использует `attendance-stage:8081` и `nginx-stage:19090`. После включения окружения добавляйте к HTTP-запросам:

```sh
curl -i -H 'Cookie: gitsail_stage=<secret>:stage' https://calls.vareonix.cc/apps/plugin-attendance/api/info
```

Секрет — расшифрованное значение `app.stage_token` из `apps/ingress/nginx/vars/base.yaml`. Не сохраняйте его в Git. Проверяйте target `nginx-stage:19090` и заголовок окружения: в текущем main это `X-GitSail-Environment`, в старой dev-конфигурации — `X-Composer-Environment`. С неверным cookie можно попасть в prod. Сам backend stage подключается к внутреннему nginx без TLS и без cookie.

## Откат

Для отката версии верните предыдущий `build.tag`. Для полного удаления attendance нужно одновременно убрать сервис, upstream и маршрут nginx: оставшийся маршрут будет давать ошибки. Откат выполняется в отслеживаемой ветке соответствующего окружения.
