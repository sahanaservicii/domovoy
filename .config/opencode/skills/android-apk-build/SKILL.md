---
name: android-apk-build
description: "Сборка Android APK/AAB (debug/release, подписание, продакшен-чеклист) в локальной среде: JDK, Android SDK, Gradle/gradlew, офлайн-ограничения."
---

# Сборка Android APK (в т.ч. production)

Навык для подготовки и сборки Android-приложений в локальной (преимущественно офлайн) среде. Сначала проверяй факты: версии JDK/SDK, наличие Gradle wrapper и кэша зависимостей.

## 1. Факты и окружение (проверить в начале)

```bash
java -version; javac -version
echo $ANDROID_HOME; echo $ANDROID_SDK_ROOT
ls /opt/android-sdk 2>/dev/null
ls /opt/android-sdk/build-tools 2>/dev/null
ls /opt/android-sdk/platforms 2>/dev/null
which adb apksigner zipalign 2>/dev/null || find /opt/android-sdk -name apksigner -o -name zipalign 2>/dev/null
ls gradlew 2>/dev/null || echo "нет gradlew в текущей папке"
ls ~/.gradle/caches ~/.gradle/wrapper 2>/dev/null | head
```

### Типичная конфигурация в этой среде (может отличаться — сверяй)
- JDK: OpenJDK 21 (`java/javac` в PATH).
- Android SDK: `/opt/android-sdk` (platform-tools, build-tools, platforms, licenses).
- Инструменты подписи/выравнивания: `build-tools/<ver>/apksigner`, `zipalign`.
- Gradle: глобального `gradle` может не быть; rely на `./gradlew` в проекте. Кэш Gradle: `~/.gradle/caches`, дистрибуции в `~/.gradle/wrapper` (если скачаны ранее).

Рекомендуемые переменные (для сессии/скрипта сборки):
```bash
export ANDROID_HOME=/opt/android-sdk
export ANDROID_SDK_ROOT=/opt/android-sdk
export PATH="$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"
# build-tools добавить по конкретной версии, напр.:
export PATH="$ANDROID_HOME/build-tools/34.0.0:$PATH"
```

## 2. Офлайн-ограничения (важно)

- **Нельзя** скачивать Gradle-дистрибутив, Maven/Google зависимости, NDK/SDK-компоненты, плагины из интернета, если среда без сети. Сборка возможна только если всё уже есть в проекте/кэше (`~/.gradle/caches`, `gradle/wrapper/gradle-wrapper.jar` + локальный distribution).
- Перед сборкой: `./gradlew --version` (если wrapper есть) и при необходимости `./gradlew tasks --offline`.
- Если зависимостей нет в кэше — честно сообщить, что для production-сборки нужен доступ к репозиториям или заранее подготовленный офлайн-кэш/локальный Maven (`mavenLocal()`).
- Установка пакетов (`apt`, `sdkmanager --install` из сети) в офлайн-режиме недоступна.

## 3. Debug-сборка (быстрая проверка)

В корне Android/Gradle-проекта (где `gradlew`, `settings.gradle*`, `app/`):
```bash
chmod +x ./gradlew
./gradlew :app:assembleDebug --stacktrace
# при наличии кэша, без сети:
./gradlew :app:assembleDebug --offline --stacktrace
```
Артефакт обычно: `app/build/outputs/apk/debug/app-debug.apk` (путь может зависеть от модулей/flavors).

Проверка APK:
```bash
"$ANDROID_HOME/build-tools/34.0.0/aapt2" dump badging app/build/outputs/apk/debug/app-debug.apk | head
apksigner verify --print-certs app/build/outputs/apk/debug/app-debug.apk
```

## 4. Release-сборка (production)

### 4.1 Подготовка проекта
- `app/build.gradle`/`build.gradle.kts`: корректные `applicationId`, `versionCode`, `versionName`, `minSdk`, `targetSdk`, `compileSdk` (должны соответствовать установленным платформам/build-tools).
- Включить минификацию/оптимизацию для release при готовности: R8/ProGuard (`minifyEnabled true`, `shrinkResources true`), правила в `proguard-rules.pro`.
- Проверить разрешения, иконки, `networkSecurityConfig`, подписи схем, `application`/`exported` для Android 12+.

### 4.2 Keystore (подпись релиза)
Создаётся один раз и **хранится безопасно** (не коммитить, не публиковать). Пароли/ключи — секреты, не печатать в чат/логи.
```bash
keytool -genkeypair -v \
  -keystore release.keystore -alias release \
  -keyalg RSA -keysize 2048 -validity 10000
```
Настройка в `gradle.properties` (локально, не в VCS) или через переменные окружения/CI-secrets:
- `RELEASE_STORE_FILE`, `RELEASE_STORE_PASSWORD`, `RELEASE_KEY_ALIAS`, `RELEASE_KEY_PASSWORD`
В `build.gradle` — `signingConfigs.release` читает эти значения.

### 4.3 Сборка подписанного APK/AAB
```bash
./gradlew :app:assembleRelease --offline --stacktrace
# или App Bundle (предпочтительно для Play/распространения):
./gradlew :app:bundleRelease --offline --stacktrace
```
Типовые артефакты:
- `app/build/outputs/apk/release/app-release.apk` (если signingConfigs настроен) или unsigned
- `app/build/outputs/bundle/release/app-release.aab`

### 4.4 Если нужен ручной sign/align (unsigned APK)
```bash
BT="$ANDROID_HOME/build-tools/34.0.0"
"$BT/zipalign" -f -v 4 app-release-unsigned.apk app-release-aligned.apk
"$BT/apksigner" sign --ks release.keystore --ks-key-alias release app-release-aligned.apk
"$BT/apksigner" verify --verbose --print-certs app-release-aligned.apk
```

## 5. Production-чеклист

- [ ] `versionCode` увеличен, `versionName` актуален.
- [ ] Release подписан release-ключом (не debug), keystore сохранён/забэкаплен отдельно.
- [ ] `minify/shrink` включены и протестированы (нет падений из-за обфускации; keep rules при необходимости).
- [ ] `targetSdk`/`compileSdk` соответствуют требованиям; разрешения минимальны.
- [ ] Проверка на реальном устройстве/эмуляторе (если доступен): установка, запуск, критические сценарии, краш-логи.
- [ ] Размер APK/AAB, отсутствие секретов в коде/ресурсах/`BuildConfig`.
- [ ] Для установки вне магазина может потребоваться разрешение на неизвестные источники; для распространения — выбран формат (APK/AAB) под канал доставки.

## 6. Диагностика проблем

- `JAVA_HOME`/несовместимость JDK и AGP/Gradle: сверить требования проекта (AGP ↔ Gradle ↔ JDK).
- `SDK location not found`: задать `sdk.dir` в `local.properties` или `ANDROID_HOME`.
- Ошибки зависимостей в офлайне: проверить наличие в `~/.gradle/caches/modules-2`; иначе нужен подготовленный кэш.
- Конфликты ресурсов/дубликаты классов: читать `--stacktrace`/`--info`, править зависимости и packaging options.
- Не выдумывать версии инструментов — брать из фактического окружения и файлов проекта.
