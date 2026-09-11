# Android CI/CD с GitHub Actions (Project 14)

Сквозной CI/CD-пайплайн для Android-приложения на GitHub Actions.

Мультимодульное Android-демо (`app`, `first`, `second`) и три workflow,
которые собирают, тестируют, сканируют и разворачивают приложение.

## Структура репозитория

```
.
├── .github/workflows/
│   ├── android.yml          # Основной пайплайн: сборка + deploy
│   ├── clear-caches.yml     # Очистка кэша Actions после прогона
│   └── delete-artifacts.yml # Плановая очистка артефактов сборки
├── app/                     # Основной модуль приложения
├── first/                   # Библиотечный модуль (AAR)
├── second/                  # Библиотечный модуль (AAR)
├── scripts/                 # Общие Gradle-задачи
├── build.gradle             # Корневой Gradle-файл (AGP + Kotlin + SonarQube)
├── settings.gradle
├── gradle/wrapper/          # Gradle wrapper
└── gradlew / gradlew.bat    # Скрипты Gradle wrapper
```

## Пайплайн (`android.yml`)

Workflow запускается при `push` в `main`, `qa`, `develop` и при
`pull_request` в `main` и `qa`. Состоит из двух джоб:

### Джоба 1: `build`

Выполняется на `ubuntu-latest` с **JDK 11**:

1. Checkout репозитория
2. Установка JDK 11 (Temurin) с кэшированием Gradle
3. `./gradlew clean` — очистка результатов сборки
4. `./gradlew lint` — статический анализ (Android Lint)
5. `./gradlew build` — компиляция и сборка APK (debug/release, flavors dev/prod)
6. `./gradlew testDevDebugUnitTest` — юнит-тесты с покрытием (JaCoCo)
7. Сканирование SonarQube (`./gradlew sonarqube`) — только если заданы
   `SONAR_TOKEN` и `SONAR_HOST_URL`
8. Файлы APK получают отметку времени и загружаются как **артефакт**
   (`apk-files-artifactory`)
9. Уведомление в **Microsoft Teams** через webhook — только если заданы
   `MS_TEAMS_WEBHOOK_URI` и `CI_GITHUB_TOKEN`

### Джоба 2: `deploy`

Запускается после успешной `build`, и только на ветках `qa` и `master`:

1. Скачивание артефакта `apk-files-artifactory`
2. Получение публичного IP раннера и временное открытие **порта 8082**
   в **AWS security group** (чтобы раннер мог достучаться до JFrog)
3. Настройка **JFrog CLI** (Artifactory)
4. Загрузка debug/release APK в репозиторий JFrog (`android-artifact/`)
5. Удаление IP раннера из security group (очистка, выполняется всегда)
6. Уведомление в Teams

> Шаги deploy, работающие с внешними сервисами (AWS/JFrog/Teams), пропускаются,
> пока соответствующие секреты не настроены — пайплайн остаётся зелёным, а CD
> включается автоматически после добавления credentials.

## Требуемые секреты репозитория

| Секрет | Назначение |
|--------|------------|
| `SONAR_TOKEN` | Токен для SonarQube (джоба build) |
| `SONAR_HOST_URL` | URL SonarQube-сервера (джоба build) |
| `CI_GITHUB_TOKEN` | GitHub-токен для уведомлений Teams (build + deploy) |
| `MS_TEAMS_WEBHOOK_URI` | Webhook для уведомлений в Teams (build + deploy) |
| `JF_URL` | URL платформы JFrog (джоба deploy) |
| `JF_ACCESS_TOKEN` | Access-токен JFrog (джоба deploy) |
| `JF_USER` | Логин JFrog (джоба deploy) |
| `JF_PASSWORD` | Пароль JFrog (джоба deploy) |
| `JFROG_SG_ID` | ID AWS security group для открытия порта (джоба deploy) |
| `AWS_ACCESS_KEY_ID` | Ключи AWS (джоба deploy) |
| `AWS_SECRET_ACCESS_KEY` | Ключи AWS (джоба deploy) |

## Как добавить секреты

GitHub: **Settings → Secrets and variables → Actions → New repository secret**

## Локальная сборка

```bash
./gradlew clean build        # Linux / macOS
.\gradlew.bat clean build    # Windows
```

Локальные требования: JDK 11+ и Android SDK с `platform 30` и
`build-tools 30.0.3` (задаются через `local.properties` / `ANDROID_HOME`).

> `local.properties` привязан к машине и **не должен** попадать в git.

## Замечания по версиям

- Используется **Gradle 7.0.2** — это версия, совместимая с
  AGP `7.0.0-beta04` проекта (Gradle 8.x вызывает ошибки на этапе lint).
- В репозиторий добавлен `gradle/wrapper/gradle-wrapper.jar` (в исходном
  проекте отсутствовал).

---
На основе [DevOps-Projects · Project 14](https://github.com/DevCloudNinjas/DevOps-Projects/tree/master/project-14-github-actions-android).