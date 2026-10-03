# Лаб. П11 — CI/CD: GitHub Actions + Docker Hub + ArgoCD

Cloud-native приложение «с нуля»: исходники, юнит-тесты, Docker-образ, CI (lint → unit-tests →
build-test), CD (публикация образа в Docker Hub и обновление манифеста) и GitOps-развёртывание
в Kubernetes (minikube) через ArgoCD.

## Схема

```
        git push                 pull request                    push в main
 dev ───────────► CI (cicd.yml) ──────────────► main ───────────────────────────► release.yml
                  lint, unit-tests, build-test    │                               1. docker build + push
                                                   │                                 lisandius/devops-psu:latest
                                                   │                               2. дата сборки → манифест,
                                                   ▼                                  коммит в ветку release
                                                 release ◄─────────────────────────────┘
                                                   │  (ArgoCD следит за веткой)
                                                   ▼
                          ArgoCD ──► Kubernetes (namespace devops-psu, Service :12345)
```

## Содержимое

```
.
├── server/
│   ├── application.py            # приложение (http.server на :8000) + класс TestMe
│   ├── test_application.py       # юнит-тесты (pytest)
│   ├── index.html                # стартовая страница (появилась в «version 2»)
│   └── dockerfile
├── requirements.txt              # pylint, pytest
├── server-k8s-manifests/devops-psu.yml   # Namespace, Deployment, Service (LoadBalancer :12345)
├── argocd/application.yaml       # ArgoCD Application (ветка release, auto-sync)
├── .github/workflows/
│   ├── cicd.yml                  # CI: lint -> unit-tests -> build-test
│   └── release.yml               # CD: публикация образа + обновление манифеста
└── screenshots/
```

## CI — `.github/workflows/cicd.yml`

Запускается на push в `dev` и на pull request в `main`:

1. **lint** — `pylint -d C0114,C0115,C0116 server/application.py` (коды отключённых проверок
   про docstring — как в лекции);
2. **unit-tests** — `pytest`;
3. **build-test** (после двух предыдущих) — сборка образа, запуск контейнера, `curl 127.0.0.1:8000`.

## CD — `.github/workflows/release.yml`

Запускается на push в `main` (т.е. после merge pull request'а):

1. **push_to_registry** — логин в Docker Hub и публикация `lisandius/devops-psu:latest`;
2. **touch-k8s-manifest** — подставляет в манифест `release-date: s<unix-time>` и коммитит в ветку
   `release`. Это нужно, чтобы ArgoCD увидел изменение манифеста и пересоздал под
   (`imagePullPolicy: Always` подтягивает новый `latest`).

> Отличие от лекции: вместо `git push --force origin master:release` ветка `release`
> обновляется обычным (fast-forward) push: workflow забирает `origin/release`, делает
> `git merge -X theirs origin/main`, правит дату и пушит. История `release` не переписывается.

Секреты репозитория (Settings → Secrets and variables → Actions): `DOCKER_USERNAME`, `DOCKER_TOKEN`
(токен Docker Hub с правами Read & Write). В репозиторий секреты не попадают.

## ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl scale deploy/argocd-dex-server -n argocd --replicas=0   # SSO не нужен (и образ dex не скачивался)

# публикация UI наружу
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc argocd-server -n argocd -p '{"spec": {"externalIPs": ["192.168.227.15"]}}'
minikube tunnel --bind-address 192.168.227.15       # отдельное окно

# пароль admin
kubectl -n argocd get secret argocd-initial-admin-secret --template={{.data.password}} | base64 -d

# приложение под управлением ArgoCD
kubectl apply -f argocd/application.yaml
```

Приложение подключено **декларативно** (`argocd/application.yaml`), а репозиторий публичный,
поэтому SSH-ключ в ArgoCD не требовался (в лекции — через UI и приватный ключ).
Ветка слежения — `release`, путь — `server-k8s-manifests`, включены `automated` sync, `prune`, `selfHeal`.

## Проверка всей цепочки

1. На ветке `main` первый релиз: образ в Docker Hub, ветка `release`, ArgoCD развернул **v1**
   (приложение отдаёт список файлов каталога).
2. В ветке `dev` — «version 2»: добавлен `index.html` (`Hello, v2`), dockerfile копирует его в образ.
3. `git push` → CI в `dev` — все три задания зелёные; открыт pull request `dev → main`, проверки в PR зелёные.
4. Merge → сработал `release.yml`: новый образ в Docker Hub, обновление `release`.
5. ArgoCD обнаружил новый коммит в `release`, пересоздал под: `curl 192.168.227.15:12345` → `Hello, v2`.

## Что не настроено автоматически

Защита ветки `main` (Settings → Branches → Add rule: *Require a pull request before merging* и
*Require status checks*: `lint`, `unit-tests`, `build-test`) в репозитории включается вручную владельцем.
Merge в `main` в этой работе выполнялся через pull request.

## Стенд

VM Ubuntu 24.04 (VMware), minikube 1.39, Kubernetes 1.37, ArgoCD 3.5.3, GitHub Actions (ubuntu-24.04).

## Скриншоты

1. `01-github-actions.png` — список запусков workflow (CI и релиз).
2. `02-ci-run-details.png` — запуск CI: lint, unit-tests, build-test.
3. `03-pull-request.png` — pull request `dev → main` с пройденными проверками.
4. `04-argocd-before.png` — ArgoCD: Healthy/Synced, версия v1 (коммит `03fa6de` ветки `release`).
5. `05-dockerhub.png` — образ `lisandius/devops-psu` на Docker Hub.
6. `06-argocd-after.png` — ArgoCD после merge: новый коммит `release`, новый ReplicaSet (rev 2).
7. `07-v1-v2-curl.png` — ответ приложения до (список файлов) и после (`Hello, v2`) обновления.
