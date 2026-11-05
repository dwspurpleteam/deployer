# Деплой плагина посещаемости

Плагин разворачивает существующий Composer, который отслеживает `dev` для stage и `main` для prod. Настройки сервера, systemd и GitHub Actions менять не нужно. Новое приложение находится в `apps/calls/attendance`; URL публикует `apps/calls/nginx`.

Страница и API плагина публикуются под `/apps/plugin-attendance/`. Схема `/apps/plugin-<name>/` явно обозначает плагин и не пересекается с каталогом `/plugins` и страницами версий `/plugins/<plugin>/versions`. NGINX удаляет `/apps/plugin-attendance/` при проксировании; относительные пути frontend продолжают работать. Запрос без завершающего `/` перенаправляется с сохранением query-параметров.

## Окружения

| Ветка deployer | Сервис | Источник событий |
| --- | --- | --- |
| `main` | `attendance:8081` | `nginx:19090` → streamapi prod |
| `dev` | `attendance-stage:8081` | `nginx-stage:19090` → streamapi stage |

Dockerfile собирает frontend и backend в один jar на Java 25. В runtime попадает только jar; локальный `application-stage.yml` с cookie не копируется. Прямое подключение к внутреннему nginx не требует stage-cookie или TLS. Порт 8081 на хост не публикуется.

Одна реплика обязательна для этой версии: состав звонков хранится в памяти. Перезапуск плагина очищает его состояние и не перезапускает LiveKit. Развёртывание меняет приложение attendance и конфигурацию nginx; пересобирать Meet для публикации URL не нужно.

## Публикация

1. Закоммитьте изменения attendance, nginx, README и этой инструкции в `deployer`.
2. Доставьте коммит в `dev` и отправьте ветку на origin. Composer соберёт образ из версии `build.tag` и запустит stage.
3. Проверьте stage с cookie окружения, как описано в README.
4. Перенесите тот же коммит в `main` и отправьте ветку на origin. Composer запустит prod.

Если в `dev` есть другие изменения, переносите в `main` только коммит посещаемости. `vars/stage.yaml` задаёт настройки окружения и сам по себе не включает stage в prod.

Автодеплой запускается изменениями в **deployer**. Пуш в репозиторий `plugins` запускает его CI, но не обновляет уже закреплённый `build.tag`. Для новой версии замените SHA в `apps/calls/attendance/vars/base.yaml`, проверьте в `dev` и затем перенесите в `main`.

## Проверка prod

После завершения автодеплоя выполните на сервере или другой машине с доступом к домену:

```sh
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/actuator/health
curl -fsS https://calls.vareonix.cc/apps/plugin-attendance/api/info
curl -fsSI https://calls.vareonix.cc/apps/plugin-attendance/
```

Ожидается: health `UP`, в info `source: streamapi` и `target` с `nginx:19090`, у страницы HTTP 200. Не передавайте stage-cookie при проверке prod. Первый reconcile может временно отдавать 502 до запуска нового сервиса.

Для stage добавьте к запросам `-H 'Cookie: composer_stage=<secret>:stage'`. В info должен быть target `nginx-stage:19090`, в HTTP-ответе — `X-Composer-Environment: stage`. Секрет берётся из существующего окружения; не добавляйте его в Git.

Создайте звонок в том же окружении и откройте:

```text
https://calls.vareonix.cc/apps/plugin-attendance/?callId=RM_…
```

`callId` — SID комнаты LiveKit, а не её имя. Откройте плагин до входа проверяемого участника: события, произошедшие до подписки, не воспроизводятся. При входе и выходе участника список должен обновиться.

Для диагностики найдите контейнер нужного окружения и проверьте его журнал:

```sh
docker ps --filter label=com.docker.compose.service=attendance
docker ps --filter label=com.docker.compose.service=attendance-stage
docker logs --tail 100 <container>
```

## Подключение к Meet и сводки

Этот деплой публикует страницу и API плагина. Текущий `vareonix/main` Meet использует mock-каталог и не читает `MEET_PLUGINS`; автоматического появления кнопки посещаемости от этих изменений не будет. Интеграция каталога/окна выполняется отдельно. Ранее используемый внешний туннель плагину для доступа по новому URL больше не нужен.

Настоящий Result Gateway пока не реализован. Этот деплой не запускает WireMock: учёт посещаемости работает, но кнопка публикации сводки возвращает `FAILED`, пока не настроен доступный Result Gateway. Позже его URL и callback можно задать через `ATTENDANCE_RESULT_GATEWAY_URL` и `ATTENDANCE_RESULT_GATEWAY_CALLBACK_URL` в `app.env`.

## Откат

Сделайте `git revert <attendance-deployment-commit>` в нужной отслеживаемой ветке и отправьте её на origin. Composer уберёт новый сервис и маршрут. Для отката только версии поменяйте `build.tag` на предыдущий SHA и отправьте коммит. Другие сервисы приложения звонков не откатывайте.

Механизм сборки и обновления описан в [документации Composer](https://github.com/dmironenko/composer/blob/master/docs/builds.md).
