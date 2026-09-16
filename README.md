# Дипломный практикум в Yandex.Cloud - Kubernetes

Репозиторий с Kubernetes-манифестами для дипломного проекта DevOps.

## Назначение

Этот репозиторий управляет ресурсами Kubernetes-кластера:
- Манифесты для подключения внешних worker-узлов (`NodeGroup`)
- (позже) манифесты приложения, Ingress, Helm-чарты мониторинга

## Связанные репозитории

- [devops-diplom-infra](https://github.com/aleksey-dubrovin/devops-diplom-infra) — инфраструктура (Terraform)
- [devops-diplom-app](https://github.com/aleksey-dubrovin/devops-diplom-app) — код приложения

## Структура

```
kubernetes/
└── external-nodes/
    └── nodegroup.yaml   # NodeGroup для внешних worker-узлов
```

## Как это работает

### Изменение инфраструктуры (добавление/пересоздание worker-узлов)

1. В репозитории `devops-diplom-infra` вносите изменения (например, новую зону в `worker_config.zones`).
2. Merge в `main` → CI/CD запускает `terraform apply`.
3. Terraform генерирует свежий `infrastructure/k8s-nodes/nodegroup.yaml`.
4. Скопируйте его содержимое в `kubernetes/external-nodes/nodegroup.yaml`.
5. Создайте PR в этом репозитории → CI применяет манифест → кластер сам подключает узлы.

### Ручное обновление

Если нужно применить манифесты без изменений в infra:
- Actions → **K8s Apply** → **Run workflow**.

## Секреты

В репозитории → Settings → Secrets and variables → Actions:

| Секрет | Описание |
|--------|----------|
| `YC_SERVICE_ACCOUNT_KEY_B64` | Base64 от JSON-ключа сервисного аккаунта |
| `YC_SSH_PRIVATE_KEY_B64` | Base64 от приватного SSH-ключа для доступа к worker-узлам |

### Как получить значения

```bash
# Service account key
cat ~/.yandex/avdubrovin-prod.json | base64 -w 0

# SSH private key
cat ~/.ssh/aleksey | base64 -w 0
```

## Проверка результата

После успешного workflow:

```bash
kubectl get nodes
# Ожидаем:
# NAME               STATUS   ROLES    AGE   VERSION
# ext-node-10.0.4.20 Ready    <none>   2m    v1.35.x
# ext-node-10.0.5.33 Ready    <none>   2m    v1.35.x
```