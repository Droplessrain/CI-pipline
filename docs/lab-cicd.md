# Лабораторная работа. CI и CD для CI_pip

## Цель

Собрать для этого приложения конвейер, в котором каждая проверка — отдельная джоба. Нужно увидеть, какая проверка за что отвечает, в каком порядке они стоят и как одна красная джоба останавливает следующие.

Платформа: **GitHub Actions**. Сборку ведёт только `./mvnw` (в репозитории зафиксирован Maven 3.10.0). Системный `mvn` в джобах не вызывать.

## Что уже есть

- Spring Boot 4.1.1, Java 21, упаковка в jar.
- Зависимости: `spring-boot-starter-webmvc`, Lombok, тестовый стартер, DevTools (`optional`, в исполняемый jar он попасть не должен).
- Порт приложения: `server.port=8084` в `src/main/resources/application.properties`.
- Единственный тест: `CiPipApplicationTests.contextLoads`.
- HTTP-эндпоинтов нет, Dockerfile нет, workflow нет.

## Что сделать до пайплайна

Без живого HTTP-ответа нечем отличить «контейнер запустился» от «приложение отвечает». Добавьте минимальную поверхность и только её.

1. `GET /ping` возвращает `200` и тело `ok`.
2. Модульный тест на MockMvc: статус 200 и тело `ok`.
3. Интеграционный тест `*IT` (`@SpringBootTest` с `RANDOM_PORT`): реальный запрос на `/ping`.
4. Существующий `contextLoads` оставьте.

DevTools в образ и в запускаемый на smoke jar не класть. Порт внутри контейнера оставьте 8084.

## Где лежат workflow

Два файла:

- `.github/workflows/ci.yml` — проверки исходников и сборка jar.
- `.github/workflows/cd.yml` — образ, его проверка, пробный запуск, публикация.

Общие правила обоих файлов:

- `permissions` заданы у workflow и сужены до того, что джобе реально нужно. По умолчанию `contents: read`.
- Кэш Maven (`~/.m2`) включён через `actions/setup-java` (Temurin 21).
- Команды Maven: `./mvnw -B -ntp ...`.
- `concurrency` отменяет предыдущий прогон той же ветки.
- Джобы связаны через `needs`. Джоба без `needs` стартует сразу и не ждёт сборку, даже если так задумано только для сканеров, которые читают дерево файлов.

## CI

Файл `ci.yml`. Запуск: `pull_request` и `push` в `main`.

Порядок и состав джоб фиксированы. Имена джоб в GitHub должны совпадать с именами в таблице: по ним потом включаются required checks.

| Джоба | От чего зависит | Что делает | Чем доказывается, что проверка живая |
| --- | --- | --- | --- |
| `toolchain` | — | `actions/setup-java` 21 и `./mvnw -v`. Падает, если в выводе нет Maven 3.10. | В логе виден `Apache Maven 3.10` и Java 21. |
| `validate` | `toolchain` | `./mvnw validate` | Джоба зелёная на текущем `pom.xml`. |
| `compile` | `validate` | `./mvnw -DskipTests compile` | Временная синтаксическая ошибка в `main` красит только её и всех, кто стоит дальше. |
| `unit-test` | `compile` | `./mvnw test` только Surefire. `failIfNoTests=true`. Артефакт: `target/surefire-reports`. | Удаление `@Test` красит джобу. Отчёт есть во вкладке артефактов. |
| `integration-test` | `compile` | Failsafe: `*IT.java`, `./mvnw failsafe:integration-test failsafe:verify`. `failIfNoTests=true`. Артефакт: `target/failsafe-reports`. | Та же проверка на интеграционном тесте. |
| `coverage` | `unit-test`, `integration-test` | JaCoCo. Правило: у класса с `/ping` line coverage не ниже 80%. Отчёт `target/site/jacoco` уходит в артефакты. | Снятие MockMvc-теста опускает покрытие и красит джобу, даже если сборка тестов ещё зелёная. |
| `enforcer` | `validate` | `maven-enforcer-plugin`: Java 21, Maven не ниже 3.9, `dependencyConvergence`, запрет дубликатов зависимостей. | Правило `requireJavaVersion` на `[1.8,17]` локально красит цель. В workflow оставьте реальный порог 21. |
| `spotbugs` | `compile` | `spotbugs-maven-plugin:check`, порог Medium. | Джоба зелёная на текущем коде и подключена в граф, а не закомментирована. |
| `sbom` | `validate` | CycloneDX `makeAggregateBom`. Артефакт: `target/bom.json`. | В артефактах прогона лежит непустой `bom.json` с `com.test:CI_pip`. |
| `vulnerabilities` | — | Trivy, `scan-type: fs`, падение только на `CRITICAL`. HIGH попадает в summary прогона и сборку не валит. | В логе есть таблица сканера. Падение на HIGH в эту лабораторную не входит: у транзитивных зависимостей Spring они бывают и без вашего кода. |
| `secrets` | — | Gitleaks по дереву коммита. | Отдельный коммит с фиктивной строкой вида секрета красит джобу. Коммит в `main` не оставлять: ветка для проверки, затем удаление. |
| `dependency-review` | — | `actions/dependency-review-action`. Условие: `github.event_name == 'pull_request'`. | На push в `main` джоба skipped. На PR она выполняется. |
| `actionlint` | — | `rhysd/actionlint` по `.github/workflows`. | Опечатка в `needs` красит эту джобу. |
| `package` | `unit-test`, `integration-test`, `coverage`, `enforcer`, `spotbugs` | `./mvnw -DskipTests package`. Артефакт: `target/CI_pip-*.jar`. | Jar скачивается из прогона. `jar tf` не содержит `spring-boot-devtools`. |

`vulnerabilities`, `secrets`, `dependency-review` и `actionlint` намеренно не стоят за `compile`: они читают репозиторий, а не `target/`.

`package` не стартует, пока красные тесты, покрытие, enforcer или SpotBugs.

## CD

Файл `cd.yml`.

Запуск:

- `workflow_run` после успешного `ci.yml` на `main`;
- `pull_request` — собрать и проверить образ, без публикации;
- `push` тега `v*`;
- `workflow_dispatch`.

Образ собирается из того же коммита, который только что прошёл CI. Публикация берёт уже проверенный образ, а не собирает второй.

| Джоба | От чего зависит | Когда выполняется | Что делает |
| --- | --- | --- | --- |
| `image` | успешный CI (на PR — джобы этого же прогона не ждут второй workflow; на `main` CD стартует из `workflow_run`) | PR, `main`, тег, ручной запуск | Multi-stage Dockerfile: сборка `./mvnw -DskipTests package`, runtime-образ Temurin 21 JRE, пользователь не root, `EXPOSE 8084`. |
| `image-scan` | `image` | всегда, когда собрался образ | Trivy по локальному образу. Падение на `CRITICAL`. |
| `smoke` | `image` | всегда, когда собрался образ | `docker run` с пробросом 8084, затем `curl -fsS http://127.0.0.1:8084/ping`. Ожидается тело `ok`. Контейнер останавливается и в успехе, и в падении (`if: always()` на шаге остановки). |
| `publish` | `image-scan`, `smoke` | только `main` | Вход в GHCR, push `ghcr.io/<owner>/ci-pip` с тегами `latest` и короткий SHA. |
| `release` | `image-scan`, `smoke` | только тег `v*` | GitHub Release и вложение jar этого коммита. |

На pull request `publish` и `release` в статусе skipped. Это часть сдачи: на скриншоте PR видно, что образ проверен, а в реестр не отправлен.

Права `publish`: `packages: write` и `contents: read`. Права `release`: `contents: write`. Остальные CD-джобы пакеты не публикуют и этих прав не имеют.

## Граф

Так должен выглядеть прогон pull request:

```text
toolchain
  -> validate -> compile -> unit-test --------\
                 |        -> integration-test --+-> coverage -> package
                 |        -> spotbugs ----------/
                 -> enforcer -------------------/
  vulnerabilities
  secrets
  dependency-review
  actionlint

package не ведёт в publish на PR

image -> image-scan -> (publish skipped)
      -> smoke ------^
                     -> (release skipped)
```

На `main` после зелёного CI тот же хвост доходит до `publish`. На теге `v*` — до `release`.

## Что на этом этапе не делается

Приложение без базы, без нескольких контуров и без кластера. В лабораторную не входят:

- выкладка в Kubernetes или на виртуальную машину;
- миграции схемы;
- нагрузочный прогон и контрактные тесты внешнего API;
- публикация артефакта в Maven Central;
- подпись jar и multi-arch образ;
- OWASP Dependency-Check с ключом NVD.

CodeQL имеет смысл на публичном репозитории. На приватном без GitHub Advanced Security джоба не запустится, поэтому в обязательный набор она не входит.

## Как прощупать, а не только включить

Нужны четыре коротких прогона, кроме финального зелёного. Каждый — отдельный коммит в учебной ветке. В `main` остаётся только зелёное состояние.

1. Сломанный модульный тест. Красные: `unit-test` и всё, что указано в `needs` от неё, включая `package`. `vulnerabilities` при этом может быть зелёной.
2. Временно завышенный порог JaCoCo. Красная только `coverage` (и кто от неё зависит).
3. Фиктивный секрет в файле. Красная `secrets`. Файл в историю `main` не попадает.
4. В smoke-запросе неверный путь, например `/pong`. Красная `smoke`, `publish` не стартует.

После каждой проверки коммит-поломку уберите. В отчёте достаточно ссылки на неуспешный run.

## Сдача

В репозитории:

- код `/ping`, модульный тест и `*IT`;
- плагины JaCoCo, Enforcer, SpotBugs, CycloneDX и Failsafe в `pom.xml`;
- `Dockerfile`;
- `.github/workflows/ci.yml` и `.github/workflows/cd.yml`.

В отчёте:

- таблица джоб со ссылкой на зелёный run `main` или итогового PR;
- скрин или ссылка, где на PR `publish` и `release` skipped;
- четыре ссылки на красные прогоны из раздела выше;
- одна фраза на джобу: что именно она не пропустила бы.

## Зачёт

Работа зачтена, когда одновременно выполняется следующее.

- На PR выполняются все CI-джобы из таблицы, кроме тех, чей `if` их пропускает.
- `package` не запускается при красном тесте.
- Скачивается jar без DevTools, скачивается `bom.json`, скачиваются отчёты Surefire, Failsafe и JaCoCo.
- На PR образ проходит `image-scan` и `smoke` и не уходит в GHCR.
- На `main` образ с тегом SHA есть в GHCR, `curl` на опубликованный образ в лабораторной не требуется.
- Четыре отрицательных прогона показывают отказ нужной джобы, а не общий `exit 1` без имени проверки.
