# track-me-platform

Общая Gradle-платформа (BOM) и библиотека `commons` для сервисов Track Me.
Здесь централизованно управляются версии Spring Boot, Spring Cloud, Spring Security
и других зависимостей всех сервисов.

## Модули

| Модуль | Координаты | Что это |
|---|---|---|
| `platform` | `net.trackme.platform:platform` | Gradle BOM (`java-platform`): версии Spring Boot, Spring Cloud, security, lombok, mapstruct и т.д. |
| `commons` | `net.trackme.commons:commons` | Общая библиотека: базовые JPA-сущности (`CoreEntity`, `BusinessEntity`, `VersionedBusinessEntity`), Spring Security ACL (`AclService`), абстракция фильтрации (`Filter`, `FilterRequest`) |
| `conventions` | Gradle-плагин `net.trackme.java-quality` | Convention-плагин качества кода |

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

## Плагин качества кода

Плагин `net.trackme.java-quality`.
В `settings.gradle` потребителя нужен репозиторий плагинов
(тот же GitHub Packages):

```gradle
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven {
            url = uri('https://maven.pkg.github.com/Students-buisness-incubator-of-TUSUR/track-me-platform')
            credentials {
                username = providers.gradleProperty('gpr.user').orNull ?: System.getenv('GITHUB_ACTOR')
                password = providers.gradleProperty('gpr.key').orNull ?: System.getenv('GITHUB_TOKEN')
            }
        }
    }
}
```

В `build.gradle` потребителя:

```gradle
plugins {
    id 'net.trackme.java-quality' version '1.0.0'
}
```

Порог покрытия переопределяется в `gradle.properties` проекта:

```properties
trackme.coverage.minimum=0.45
```

## Сборка

```bash
./gradlew build                              # сборка + тесты
./gradlew publishToMavenLocal                # локальная публикация platform и commons в ~/.m2
./gradlew publishConventionsToMavenLocal     # локальная публикация плагина net.trackme.java-quality
```

Java 21 (toolchain), Gradle 8.8 (wrapper).

## Релиз

Релиз делается git-тегом — workflow `publish.yml` публикует platform, commons
и плагин conventions в GitHub Packages с версией из тега:

```bash
git tag v1.2.3
git push origin v1.2.3
```

SNAPSHOT-версии в GitHub Packages не публикуются. Для локальной разработки против
неопубликованных изменений используйте `publishToMavenLocal` и временно добавьте
`mavenLocal()` первым репозиторием у потребителя (не коммитить).
