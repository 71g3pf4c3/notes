# Vault secrets operator
## Инструкция для инициализации vault secrets operator в k8s с использованием kubernetes authorization

Vault secrets operator - офицальная реализация оператора для доставки от самих hashicorp.
Появился давно, хорошо заточен под vault (очевидно).

Преимущественно отличается от других нормально работающей авторизацией через kuberetes serviceaccount. (1) 

## Требования перед настройкой

Нужен работающий vault , в идеале внутри k8s

## Настройка vault

Кладём секрет

```

vault secrets enable -path="kubernetes" kv-v2 | true # Создаём хранилище

vault auth disable kubernetes # идемпотентно создаём авторизацию для k8s
vault auth enable kubernetes

vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://kubernetes.default.svc/ \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt # В случае если vault задеплоен внутри кубера, удобно ипользовать уже имеющийся serviceaccount и jwt, хост кубера внутренний

# Создаём политику, благодаря которой мы ограничиваем доступ к секретом по неймспейсам
ACCESSOR=$(vault auth list | grep kubernetes | awk '{print $3}') # Accessor - уникальное генерируемое имя метода авторизации

vault policy write kubernetes - <<EOF
path "kubernetes/data/{{identity.entity.aliases.$ACCESSOR.metadata.service_account_namespace}}/*" {
  capabilities = ["read"]
}
EOF
```

(1) При деплое сущонсти VaultSecret используется serviceaccount из локального namespace, что позволяет полноценно реализовать политику доступа без создания VaultAuth на каждый namespace в отдельности, в отличие от external secrets operator, где нужно деплоить сущность с авторизацией в каждый ns.

## Деплой оператора

Стандартно деплоим https://github.com/hashicorp/vault-secrets-operator, с такими values:

```
    defaultVaultConnection:
      enabled: true
      address: "http://vault.vault.svc:8200" # Адрес волта
    defaultAuthMethod:
      enabled: true
      namespace: "vault"
      mount: kubernetes
      kubernetes: # Насттройка метода авторизации
        role: default # default роль создали мы
        allowedNamespaces: ["*"] # Разрешаем создавать секреты всем неймспейсам
        method: kubernetes
        serviceAccount: default # Сервисаккаунт default создаётся во всех неймспейсах по умолчанию
```

## Создание секрета

### Создаём в vault секреты

vault kv put -mount="kubernetes"  \
  vso-example/helloworld \
  user=hello \
  password=nomad

vault kv put -mount="kubernetes"  \
  vault/helloworld \
  user=hello \
  password=nomad






### Применяем манифесты:

``` yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: vso-example
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  namespace: vso-example
  name: vault-static-secret-v2-1
spec:
  vaultAuthRef: vault-secrets-operator/default
  mount: kubernetes
  type: kv-v2
  path: vso-example/helloworld # Этот секрет мы пытаемся считать из текущего неймспейса
  refreshAfter: 60s
  destination:
    create: true
    name: static-secret1
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  namespace: vso-example
  name: vault-static-secret-v2-2
spec:
  vaultAuthRef: vault-secrets-operator/default 
  mount: kubernetes
  type: kv-v2
  path: vault/helloworld  # Тут мы из namespace vso-example пытаемся достать секрет по другому пути
  refreshAfter: 60s
  destination:
    create: true
    name: static-secret2
```

PROFIT!!!!!11!!!

Видим статус vault-static-secret-v2-1

│   Normal  SecretSynced   2m32s  VaultStaticSecret  Secret synced                                                                                                     │

Видим статус vault-static-secret-v2-2

│ Code: 403. Errors:                                                                                                                                                   │
│                                                                                                                                                                      │
│ * 1 error occurred:                                                                                                                                                  │
│   * permission denied                                                                                                                                                │

Теперь мы можем получать статические секреты, ограниченные политиками из вольта

## Что дальше?

Тутор от хашикорпа - https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator

Мгновенные обновления секретов - https://developer.hashicorp.com/vault/docs/platform/k8s/vso/sources/vault/instant-updates

Кеширование - https://developer.hashicorp.com/vault/docs/platform/k8s/vso/sources/vault/client-cache

Шаблонизирование секретов - https://developer.hashicorp.com/vault/docs/platform/k8s/vso/secret-transformation

