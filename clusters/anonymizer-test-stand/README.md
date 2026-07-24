# Anonymizer Test Stand

## Зависимости

Данный вариант развертывания использует CR `monitoring.coreos.com/v1:PrometheusRule`.

## Инструкция по развертыванию

### 1. Копирование конфигурации

Скопируй себе в получившийся fluxcd репозиторий фолдер `anonymizer`.

### 2. Проверка Helm values

Убедись что правильно указаны Values для хельм релиза. Основные values есть в примере, но если чего-то не хватает можно проверить весь чарт с вальюсами вот так:

```bash
helm pull oci://cr.yandex/crp2cvbrp76d7dmfegco/helm-charts/anonymizer-app
```

### 3. Деплой и настройка секретов

Запушь получившийся код в main, fluxcd должен подхватить и попытаться задеплоить приложение, однако повиснет на отсутствующих секретах. Задеплой эти секреты любым удобным способом.

Приложение ожидает увидеть следующие секреты:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: manual-secrets
  namespace: anonymizer
data:
  MindboxAuth__AnonymizerPrivateKey: >-
    <example>==
  MindboxAuth__MindboxPublicKey: >-
    <example>==
  S3__Private__AccessKey: <example>==
  S3__Private__SecretKey: <example>==
  S3__Public__AccessKey: <example>==
  S3__Public__SecretKey: <example>==
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  name: teleport-secret
  namespace: anonymizer
data:
  auth_token: <example>==
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  name: db-anonymizer-anonymizer-user-password
  namespace: anonymizer
data:
  password: <example>==
  username: <example>==
type: kubernetes.io/basic-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: oidc-credentials
  namespace: anonymizer
data:
  client_secret: <example>==
type: Opaque
```

### Описание секретов

- **manual-secrets** - секреты для авторизации между сервисами Mindbox и anonymizer, а также s3. Паблик от пары ключей Mindbox вам предоставит менеджер, вашу пару необходимо сгенерировать по инструкции [docs/key-generation.md](../../docs/key-generation.md), приватную часть положить в секрет, а публичную передать менеджеру Mindbox.
Ключи к s3 - обычные статик ключи для s3, обе пары никак не передаются в Mindbox, но наше облако будет ходить в public бакет по signed url.

> ⚠️ `public` в названии бакета не означает анонимный доступ из интернета — это лишь про то, что с бакетом работают ещё и микросервисы Mindbox (по подписанным ссылкам). Про термины `public` / `private` и требования к доступу — см. [корневой README](../../README.md#3-s3-хранилище).
- **teleport-secret** - токен для teleport агента, предоставляется менеджером Mindbox.
- **db-anonymizer-anonymizer-user-password** - логин и пароль пользователя PostgreSQL
- **oidc-credentials** - client secret OIDC-клиента в IdP. Значение предоставляет администратор IdP. Необходим только при использовании OIDC-аутентификации.

### 4. Завершение деплоя

После того как секреты были добавлены, деплой должен успешно завершиться.

### 5. Настройка ингресса

Настройка ингресса остается за вами, в данном примере использован external NLB yandex-cloud, но использовать его в продакшен окружении крайне не рекомендуется, т.к. он не имеет функционала терминации TLS. В конечном итоге должны получиться 2 адреса (пример: anonymizer-regular.company-name.ru & anonymizer-important.company-name.ru), которые следует передать менеджеру Mindbox.
