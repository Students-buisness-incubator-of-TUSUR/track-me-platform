# track-me-platform

Общая Gradle-платформа (BOM) и библиотека `commons` для сервисов Track Me.
Здесь централизованно управляются версии Spring Boot, Spring Cloud, Spring Security
и других зависимостей всех сервисов.

## Модули

| Модуль | Координаты | Что это |
|---|---|---|
| `platform` | `net.trackme.platform:platform` | Gradle BOM (`java-platform`): версии Spring Boot, Spring Cloud, security, lombok, mapstruct и т.д. |
| `commons` | `net.trackme.commons:commons` | Общая библиотека: базовые JPA-сущности (`CoreEntity`, `BusinessEntity`, `VersionedBusinessEntity`), Spring Security ACL (`AclService`), абстракция фильтрации (`Filter`, `FilterRequest`) |

## Подключение

Артефакты публикуются в GitHub Packages. Для чтения нужен любой валидный GitHub-токен
(для PAT достаточно scope `read:packages`).

В `build.gradle` потребителя:

```gradle
repositories {
    mavenCentral()
    maven {
        url = uri('https://maven.pkg.github.com/Students-buisness-incubator-of-TUSUR/track-me-platform')
        credentials {
            username = project.findProperty('gpr.user') ?: System.getenv('GITHUB_ACTOR')
            password = project.findProperty('gpr.key') ?: System.getenv('GITHUB_TOKEN')
        }
    }
}

dependencies {
    implementation platform('net.trackme.platform:platform:1.0.0')
    implementation 'net.trackme.commons:commons:1.0.0'
}
```

Локально токен задаётся в `~/.gradle/gradle.properties`:

```properties
gpr.user=<github-username>
gpr.key=<personal-access-token>
```

либо через переменные окружения `GITHUB_ACTOR` / `GITHUB_TOKEN`.

## Сборка

```bash
./gradlew build                 # сборка + тесты
./gradlew publishToMavenLocal   # локальная публикация в ~/.m2 (версия 1.0.0-SNAPSHOT)
```

Java 21 (toolchain), Gradle 8.8 (wrapper).

## Релиз

Релиз делается git-тегом — workflow `publish.yml` публикует обе библиотеки
в GitHub Packages с версией из тега:

```bash
git tag v1.2.3
git push origin v1.2.3
```

SNAPSHOT-версии в GitHub Packages не публикуются. Для локальной разработки против
неопубликованных изменений используйте `publishToMavenLocal` и временно добавьте
`mavenLocal()` первым репозиторием у потребителя (не коммитить).
