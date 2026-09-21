
# Конфигурация Kubernetes кластера для дипломного проекта.

Репозиторий содержит манифесты для развёртывания в кластере:
Secret'ы, Service, Ingress, Deployment, Helm-values для системных
компонентов и настройку внешних worker-узлов.

## Связанные репозитории

- [devops-diplom-infra](https://github.com/aleksey-dubrovin/devops-diplom-infra) — инфраструктура (Terraform)
- [devops-diplom-app](https://github.com/aleksey-dubrovin/devops-diplom-app) — код приложения и CI/CD

## Структура

```text
devops-diplom-k8s/
├── config/                     # k8s-манифесты для kubectl apply
│   ├── app/                    # Deployment (рендерится deploy.yml)
│   ├── ingress/                # Ingress-ресурсы
│   ├── secrets/                # реестр Secret-объектов
│   └── service/                # Service-ресурсы
├── external-nodes/             # манифест NodeGroup для внешних узлов
├── helm/                       # values для Helm-чартов
│   ├── charts.yaml             # реестр чартов
│   ├── ingress-nginx-values.yaml
│   └── monitoring-values.yaml
├── monitoring/                 # конфиги для Prometheus-стека
│   ├── alertmanager.yaml       # config Alertmanager
│   └── prometheus-rules.yaml   # тестовые правила алертов
└── .github/workflows/
    ├── deploy.yml              # деплой приложения
    ├── k8s-apply.yml           # подключение внешних узлов
    ├── k8s-config.yml          # Secret, Service, Ingress
    └── k8s-helm.yml            # установка Helm-чартов
```

## Что развёртывается

| Компонент | Namespace | Как |
|-----------|-----------|-----|
| External workers | `yandex-system` | `k8s-apply.yml`, NodeGroup CRD |
| Secret'ы | `default`, `monitoring` | `k8s-config.yml`, реестр `config/secrets/registry.yaml` |
| Service и Ingress | `default`, `monitoring` | `k8s-config.yml` |
| NGINX Ingress | `ingress-nginx` | `k8s-helm.yml`, Helm |
| kube-prometheus-stack | `monitoring` | `k8s-helm.yml`, Helm |
| diplom-app | `default` | `deploy.yml`, envsubst-рендер |
| max-bot | `monitoring` | `deploy.yml`, envsubst-рендер |

## CI/CD

### k8s-apply.yml

Подключение внешних worker-узлов к кластеру:

1. Создаёт SSH-секрет `external-node-ssh-key`.
2. Применяет манифест `external-nodes/nodegroup.yaml`.
3. Ждёт готовности узлов.

Триггер: push в main в папку `external-nodes/`.

### k8s-config.yml

Применяет Secret'ы, Service и Ingress:

1. Читает реестр `config/secrets/registry.yaml`.
2. Создаёт Secret-объекты в нужных namespace.
3. Создаёт Secret `alertmanager-config` из `monitoring/alertmanager.yaml`.
4. Применяет Service и Ingress.
5. Перезапускает Grafana при обновлении её Secret.

Триггер: push в main в папки `config/` или `monitoring/`.

### k8s-helm.yml

Универсальный workflow для Helm-чартов:

1. Читает реестр `helm/charts.yaml`.
2. Добавляет репозитории, обновляет индекс.
3. Устанавливает чарты через `helm upgrade --install --atomic`.
4. Снимает pending-релизы перед установкой.

Триггер: push в main в папку `helm/`.

### deploy.yml

Деплой приложения по событию от `devops-diplom-app`:

1. Принимает `repository_dispatch` с типом `release-published`.
2. Читает URL реестра и имена образов из Secret `app-registry`.
3. Рендерит манифесты из `config/app/` через envsubst.
4. Применяет в кластер.
5. Ждёт завершения rollout для обоих Deployment.

Триггер: `repository_dispatch` или `workflow_dispatch`.

## GitHub Secrets

Для работы workflow нужны:

| Secret | Назначение |
|--------|-----------|
| `YC_SERVICE_ACCOUNT_KEY_B64` | JSON-ключ SA в base64 |
| `YC_CLOUD_ID` | ID облака |
| `YC_FOLDER_ID` | ID папки |
| `YC_SSH_PRIVATE_KEY_B64` | Приватный SSH-ключ для external nodes |
| `REGISTRY_URL` | `cr.yandex/crph52se6qjtjg937i7h` |
| `GRAFANA_ADMIN_PASSWORD` | Пароль Grafana |
| `WILDCARD_TLS_CRT` | TLS-сертификат `*.dubrovins.ru` |
| `WILDCARD_TLS_KEY` | Приватный ключ сертификата |
| `MAX_BOT_TOKEN` | Токен бота в MAX |
| `MAX_CHAT_ID` | ID чата MAX для уведомлений |
| `K8S_REPO_PAT` | PAT для деплоя из app-репозитория |

## Развёртывание с нуля

1. Убедиться, что инфраструктура развёрнута (см.
   [devops-diplom-infra](https://github.com/aleksey-dubrovin/devops-diplom-infra)).
2. Смержить изменения `external-nodes/` → запустится `k8s-apply.yml`.
3. Смержить изменения `helm/` → запустится `k8s-helm.yml`.
4. Смержить изменения `config/` → запустится `k8s-config.yml`.
5. Создать тег в `devops-diplom-app` → запустится `deploy.yml`.

## Проверка

```bash
# Узлы кластера
kubectl get nodes -o wide

# Все поды
kubectl get pods -A

# Приложение
kubectl -n default get deploy diplom-app
kubectl -n monitoring get deploy max-bot

# Мониторинг
kubectl -n monitoring get pods
```

## Особенности

- **Data-driven подход.** Реестры `charts.yaml` и `registry.yaml`
  вместо дублирующегося кода. Новый чарт или Secret добавляется
  одной записью.
- **envsubst.** Манифесты в `config/app/` содержат плейсхолдеры
  `${REGISTRY_URL}`, `${IMAGE_TAG}`, которые подставляются
  в workflow. Это убирает хардкод.
- **Sensitive-секреты.** Ничего не хранится в Git. Все значения
  идут из GitHub Secrets.
- **Идемпотентность.** `kubectl apply` и `helm upgrade --install`
  безопасны при повторных запусках.

## Известные ограничения

- **Cilium MTU.** В tunnelled mode необходимо вручную задавать
  MTU 1450 в `cilium-config`. Иначе пакеты между узлами теряются.
- **Admission webhook ingress-nginx.** Отключён, потому что мастер
  не может достучаться до webhook-пода в tunnelled mode.
- **PVC на external nodes.** CSI-драйвер Yandex Cloud не работает
  с внешними узлами. Prometheus и Grafana используют `emptyDir`.

## Документация

Полная пояснительная записка:
[https://app.dubrovins.ru/docs.html](https://app.dubrovins.ru/docs.html)