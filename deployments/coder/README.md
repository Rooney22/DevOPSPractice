# Развёртывание Coder в Kubernetes через ArgoCD (GitOps)

Проект разворачивает платформу удалённых рабочих сред **Coder** в локальном кластере **Minikube** под управлением **ArgoCD** по принципу GitOps. Все компоненты описаны декларативно в этом репозитории; ArgoCD непрерывно синхронизирует состояние кластера с конфигурацией в гите и автоматически восстанавливает удалённые ресурсы.

## Шаг 1. Поднять Minikube

Если на машине уже был старый профиль Minikube — снести его, чтобы не унаследовать сломанные ресурсы:

```powershell
minikube delete
```

Поднять чистый кластер с включённым ingress-аддоном:

```powershell
minikube start --memory=6144 --cpus=2 --addons=ingress
```

Проверить:

```powershell
kubectl get nodes
```

Узел `minikube` должен быть в статусе `Ready`.

## Шаг 2. Установить ArgoCD

```powershell
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> Флаг `--server-side` нужен, чтобы избежать ошибки «annotation too long» — CRD ArgoCD больше 256 КБ и при обычном `apply` падает.

Дождаться готовности подов:

```powershell
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=600s
kubectl get pods -n argocd
```

Все семь подов должны быть `1/1 Running`. Первый запуск тянет крупные образы с `quay.io` и может занять 5–10 минут.

## Шаг 3. Применить Ingress для ArgoCD и root-app

Из корня репозитория:

```powershell
kubectl apply -f deployments/coder/argocd-ingress.yaml
kubectl apply -f deployments/coder/root-app.yaml
```

Через 1–2 минуты появятся четыре Application:

```powershell
kubectl get applications -n argocd
```

Ожидаемый вывод (все строки `Synced` + `Healthy`):

```
NAME             SYNC STATUS   HEALTH STATUS
coder            Synced        Healthy
coder-ingress    Synced        Healthy
coder-postgres   Synced        Healthy
root-app         Synced        Healthy
```

Под Coder в первые минуты будет рестартовать (ждёт Postgres) — это нормально, после старта БД он стабилизируется.

Проверка подов:

```powershell
kubectl get pods -n coder
```

Должны быть `postgres-...` и `coder-...` оба в `Running`.

## Шаг 4. Настроить DNS и доступ из браузера

### Файл hosts

Узнать IP кластера:

```powershell
minikube ip
```

Открыть `C:\Windows\System32\drivers\etc\hosts` **от имени администратора** и добавить (или обновить, если строки уже были):

```
<IP из minikube ip>  argocd.cluster.k8s
<IP из minikube ip>  coder.cluster.k8s
```

Сбросить кеш DNS:

```powershell
ipconfig /flushdns
```

### minikube tunnel (обязательно на Windows + Docker)

На Windows с драйвером Docker IP minikube напрямую с хоста недоступен. Открыть **отдельное окно PowerShell от администратора** и запустить:

```powershell
minikube tunnel
```

Окно не закрывать — пока оно живёт, Ingress доступен с хоста. Если используется tunnel, IP в `hosts` следует заменить на `127.0.0.1`:

```
127.0.0.1  argocd.cluster.k8s
127.0.0.1  coder.cluster.k8s
```

## Шаг 5. Первый вход

### ArgoCD

Получить пароль администратора:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Открыть `https://argocd.cluster.k8s` — браузер предупредит про самоподписанный сертификат, нажать «Дополнительно → Перейти на сайт». Логин: `admin`, пароль из команды выше.

### Coder

Открыть `https://coder.cluster.k8s` — после прохождения предупреждения о сертификате появится экран создания первого пользователя (администратора Coder). Заполнить email, username, пароль.

## Шаг 6. Проверка GitOps-самовосстановления

Удалить deployment Coder руками:

```powershell
kubectl delete deployment coder -n coder
kubectl get pods -n coder -w
```

ArgoCD в течение ~30 секунд обнаружит расхождение с конфигурацией в репозитории и пересоздаст deployment. Под снова появится в `Running`. Это подтверждает работу `selfHeal: true`.

В UI ArgoCD приложение `coder` ненадолго станет `OutOfSync/Progressing` и вернётся в `Synced/Healthy`.
