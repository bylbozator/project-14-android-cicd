# Android CI/CD with GitHub Actions (Project 14)

End-to-end CI/CD pipeline for an Android application using GitHub Actions.

This project contains a multi-module Android demo app (`app`, `first`, `second`)
and three GitHub Actions workflows that build, test, scan and deploy it.

## Repository layout

```
.
├── .github/workflows/
│   ├── android.yml          # Main build + deploy pipeline
│   ├── clear-caches.yml     # Clears GH Actions caches after a run
│   └── delete-artifacts.yml # Scheduled cleanup of build artifacts
├── app/                     # Main Android application module
├── first/                   # Library module (AAR)
├── second/                  # Library module (AAR)
├── scripts/                 # Shared Gradle task definitions
├── build.gradle             # Root Gradle build (AGP + Kotlin + SonarQube)
├── settings.gradle
├── gradle/wrapper/          # Gradle wrapper
└── gradlew / gradlew.bat    # Gradle wrapper scripts
```

## The CI/CD pipeline (`android.yml`)

The workflow triggers on `push` to `main`, `qa`, `develop`, and on
`pull_request` into `main` and `qa`. It has two jobs:

### Job 1: `build`
Runs on `ubuntu-latest` with **JDK 11**:

1. Checkout the repository
2. Set up JDK 11 (Temurin) with Gradle caching
3. `./gradlew clean` — clean build outputs
4. `./gradlew lint` — static analysis (Android Lint)
5. `./gradlew build` — compile + assemble APKs (debug/release, dev/prod flavors)
6. `./gradlew jacocoTest` — code coverage via JaCoCo
7. SonarQube scan (`./gradlew sonarqube`) for code quality (needs `SONAR_TOKEN` + `SONAR_HOST_URL`)
8. Date-stamp the output APKs, zip them and upload them as a **GitHub Actions artifact** (`apk-files-artifactory`)
9. Send a status notification to a **Microsoft Teams** channel via webhook

### Job 2: `deploy`
Runs after `build` succeeds, only on `qa` and `master` branches:

1. Download the `apk-files-artifactory` artifact
2. Get the runner's public IP and temporarily open the **port 8082** of an
   **AWS security group** (for the runner to reach JFrog)
3. Configure the **JFrog CLI** (Artifactory)
4. Upload the debug/release APKs to a JFrog repository (`android-artifact/`)
5. Remove the runner IP from the security group (cleanup, always runs)
6. Send a Teams notification

## Required repository secrets

| Secret | Purpose |
|--------|---------|
| `SONAR_TOKEN` | Token for SonarQube server (build job) |
| `SONAR_HOST_URL` | URL of the SonarQube server (build job) |
| `CI_GITHUB_TOKEN` | GitHub token used by the Teams notification action (build + deploy) |
| `MS_TEAMS_WEBHOOK_URI` | Webhook URL for Teams notifications (build + deploy) |
| `JF_URL` | JFrog platform URL (deploy job) |
| `JF_ACCESS_TOKEN` | JFrog access token (deploy job) |
| `JF_USER` | JFrog username (deploy job) |
| `JF_PASSWORD` | JFrog password (deploy job) |
| `JFROG_SG_ID` | AWS security group ID to open for the runner (deploy job) |
| `AWS_ACCESS_KEY_ID` | AWS credentials (deploy job) |
| `AWS_SECRET_ACCESS_KEY` | AWS credentials (deploy job) |

## Add secrets to the repository

GitHub page: **Settings → Secrets and variables → Actions → New repository secret**

## Build locally

```bash
./gradlew clean build        # Linux / macOS
.\gradlew.bat clean build    # Windows
```

Prerequisites locally: JDK 11+ and an Android SDK with `platform 30` and
`build-tools 30.0.3` (set via `local.properties` / `ANDROID_HOME`).

> `local.properties` is machine-specific and must **not** be committed.

---
Based on [DevOps-Projects · Project 14](https://github.com/DevCloudNinjas/DevOps-Projects/tree/master/project-14-github-actions-android).
