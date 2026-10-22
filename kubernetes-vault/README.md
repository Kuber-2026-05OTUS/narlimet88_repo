# Домашнее задание №11. Настройка Vault и секретов

## Цель

Развернуть Consul и HashiCorp Vault в Kubernetes, настроить хранение секретов в Vault и их автоматическую синхронизацию с Kubernetes через External Secrets Operator.

## Окружение

Работа выполнена в локальном кластере Minikube.

Используемые компоненты:

- Kubernetes (Minikube)
- Helm
- Consul 2.0.3
- HashiCorp Vault 2.0.4
- External Secrets Operator 2.10.0

Для Consul используется namespace `consul`, для Vault и External Secrets Operator — namespace `vault`.

В отличие от многоузлового кластера, все Pod размещены на одном узле Minikube. Поэтому HA реализован на уровне реплик и выбора лидера, но не обеспечивает отказоустойчивость при потере самого узла.

## 1. Установка Consul

Создаём namespace:

```bash
kubectl create namespace consul
```

Для установки используется официальный Helm-чарт Consul из репозитория HashiCorp:

```bash
git clone --branch v2.0.3 https://github.com/hashicorp/consul-k8s.git
```

Устанавливаем Consul:

```bash
helm upgrade --install consul ./consul-k8s/charts/consul \
  -n consul \
  -f consul-values.yaml \
  --wait \
  --timeout 10m
```

Конфигурация `consul-values.yaml` предусматривает три сервера Consul с постоянным хранилищем PVC.

Проверка:

```bash
kubectl get pods -n consul

kubectl exec -n consul consul-server-0 -c consul -- consul members

kubectl exec -n consul consul-server-0 -c consul -- \
  consul operator raft list-peers
```

Результат: три сервера находятся в состоянии `alive`, один сервер является лидером Raft, остальные — followers.

## 2. Установка Vault

Создаём namespace:

```bash
kubectl create namespace vault
```

Vault устанавливается с помощью официального Helm-чарта HashiCorp с конфигурацией из `vault-values.yaml`.

Основные настройки:

- HA включён.
- Количество реплик — 3.
- В качестве backend-хранилища используется Consul.
- Адрес Consul: `consul-server.consul.svc.cluster.local:8500`.
- TLS отключён для локального учебного окружения.

После установки Vault необходимо инициализировать один раз:

```bash
kubectl exec -n vault vault-0 -- \
  vault operator init -key-shares=1 -key-threshold=1
```

**Важно:** unseal key и root token необходимо сохранить вне Git-репозитория. Повторно инициализировать уже существующее хранилище нельзя.

Для распечатывания каждого Pod используется команда:

```bash
kubectl exec -n vault vault-0 -- vault operator unseal
```

Аналогично распечатываются `vault-1` и `vault-2`.

Проверка:

```bash
kubectl get pods -n vault

kubectl exec -n vault vault-0 -- vault status
kubectl exec -n vault vault-1 -- vault status
kubectl exec -n vault vault-2 -- vault status
```

Результат:

| Pod | Sealed | HA Mode |
|---|---|---|
| vault-0 | false | active |
| vault-1 | false | standby |
| vault-2 | false | standby |

Все три Pod находятся в состоянии `1/1 Running`.

## 3. Настройка KV Secret Engine

В Vault включён KV Secret Engine версии 1 по пути `otus/`:

```bash
vault secrets enable -path=otus -version=1 kv
```

Создан тестовый секрет:

```bash
vault kv put otus/cred \
  username=otus \
  password=asajkjkahs
```

Проверка:

```bash
vault kv get otus/cred
```

Результат:

```text
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
```

Команды `vault` выполняются в аутентифицированной сессии Vault, например внутри Pod с установленной переменной `VAULT_TOKEN`.

## 4. Настройка Kubernetes Authentication

Создан ServiceAccount `vault-auth` в namespace `vault`.

Манифест `vault-auth.yaml` также содержит:

- Secret с токеном ServiceAccount;
- ClusterRoleBinding `vault-auth-delegator`;
- привязку к ClusterRole `system:auth-delegator`.

Применение:

```bash
kubectl apply -f vault-auth.yaml
```

В Vault включён метод аутентификации Kubernetes:

```bash
vault auth enable kubernetes
```

Метод Kubernetes Auth настроен для проверки ServiceAccount-токенов Kubernetes.

Создана политика `otus-policy` из файла `otus-policy.hcl`:

```hcl
path "otus/cred" {
  capabilities = ["read", "list"]
}
```

Загрузка политики:

```bash
vault policy write otus-policy otus-policy.hcl
```

Создана роль `otus`, привязанная к ServiceAccount `vault-auth` в namespace `vault`:

```bash
vault write auth/kubernetes/role/otus \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=vault \
  policies=otus-policy \
  ttl=1h
```

Аутентификация через Kubernetes ServiceAccount была успешно проверена.

## 5. Установка External Secrets Operator

External Secrets Operator установлен в namespace `vault` с помощью Helm-чарта `external-secrets/external-secrets`, версия `2.10.0`.

Проверка:

```bash
helm status external-secrets -n vault
kubectl get pods -n vault
```

Результат:

```text
STATUS: deployed
```

Все три компонента ESO работают:

- `external-secrets`
- `external-secrets-cert-controller`
- `external-secrets-webhook`

## 6. Создание SecretStore

Файл `secret-store.yaml` содержит конфигурацию подключения к Vault:

- Vault server: `http://vault.vault.svc.cluster.local:8200`
- KV path: `otus`
- KV version: `v1`
- Kubernetes Auth mount: `kubernetes`
- Vault role: `otus`
- ServiceAccount: `vault-auth`

Применение:

```bash
kubectl apply -f secret-store.yaml
```

Проверка:

```bash
kubectl get secretstore -n vault
kubectl describe secretstore vault -n vault
```

Результат:

```text
Reason:  Valid
Status:  True
Message: store validated
```

## 7. Создание ExternalSecret

Файл `external-secret.yaml` настраивает получение двух значений из Vault:

- `otus/cred` → `username`
- `otus/cred` → `password`

Целевой Kubernetes Secret называется `otus-cred`.

Применение:

```bash
kubectl apply -f external-secret.yaml
```

Проверка:

```bash
kubectl get externalsecret -n vault
```

Результат:

```text
NAME        STORETYPE     STORE   REFRESH INTERVAL   STATUS         READY
otus-cred   SecretStore   vault   1h                 SecretSynced   True
```

Проверяем созданный Kubernetes Secret:

```bash
kubectl get secret otus-cred -n vault
```

Результат:

```text
NAME        TYPE     DATA
otus-cred   Opaque   2
```

Проверка значений:

```bash
kubectl get secret otus-cred -n vault \
  -o jsonpath='{.data.username}' | base64 -d

kubectl get secret otus-cred -n vault \
  -o jsonpath='{.data.password}' | base64 -d
```

Полученные значения:

```text
otus
asajkjkahs
```

## 8. Результаты

В ходе выполнения домашнего задания:

1. Развёрнут Consul из трёх серверов.
2. Развёрнут Vault в режиме HA с хранилищем Consul.
3. Выполнена инициализация и распечатывание Vault.
4. Создан KV v1 Secret Engine `otus/` и секрет `otus/cred`.
5. Настроена аутентификация Vault через Kubernetes ServiceAccount.
6. Созданы политика `otus-policy` и роль `otus`.
7. Установлен External Secrets Operator.
8. Создан SecretStore с подключением к Vault.
9. Создан ExternalSecret для автоматической синхронизации секрета.
10. Подтверждено успешное создание Kubernetes Secret `otus-cred` с правильными значениями.

## Состав файлов

| Файл | Назначение |
|---|---|
| `consul-values.yaml` | Helm values для Consul |
| `vault-values.yaml` | Helm values для Vault |
| `vault-auth.yaml` | ServiceAccount, токен и ClusterRoleBinding |
| `otus-policy.hcl` | Политика доступа Vault |
| `secret-store.yaml` | Подключение ESO к Vault |
| `external-secret.yaml` | Синхронизация секрета из Vault |
| `README.md` | Описание выполнения домашнего задания |

## Безопасность

Root token и unseal key Vault хранятся вне репозитория и не публикуются в GitHub.
TLS отключён только для локального учебного окружения.
