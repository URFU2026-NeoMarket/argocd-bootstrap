# argocd-bootstrap

Точка входа GitOps для k3s-кластера: корневой app-of-apps `Application` и ApplicationSet'ы
(платформенный конфиг ArgoCD). Делает ApplicationSet `team-services` самоприменяемым из git —
без ручного `kubectl apply`.

```
push в этот репо
     │
     ▼
  root (Application, apply 1 раз)  ──sync──►  team-services (ApplicationSet)
     каталог apps/                                 │ git-generator
                                                    ▼
                          teamdemo-svc-1-frontend / -backend / svc-2-admin
```

## Структура

```
.
├── bootstrap/root.yaml        корневой Application — kubectl apply ОДИН раз (bootstrap)
└── apps/team-services.yaml    ApplicationSet под GitOps (источник истины)
```

В `apps/` кладём **только** Argo-CR (Application/ApplicationSet) — `root` синкает каталог
с `recurse: true` и применит всё, что найдёт.

> `apps/team-services.yaml` должен **байт-в-байт** совпадать с живым ApplicationSet в кластере —
> тогда первый синк `root` усыновит его без пересоздания `teamdemo-*` (см. шаг 1 ниже).

## Bootstrap (одноразовый)

Применяется один раз против целевого кластера (`kubectl` настроен на нужный контекст).
Это единственное ручное действие — дальше всё через `git push`.

1. **Сверка adoption** — diff должен быть только по tracking-аннотациям, НЕ по `spec`:
   ```bash
   kubectl diff -f apps/team-services.yaml
   ```
2. **Применить `root`** (сначала dry-run, затем реально):
   ```bash
   kubectl apply --dry-run=server -f bootstrap/root.yaml
   kubectl apply -f bootstrap/root.yaml
   ```
3. **Проверка:**
   ```bash
   kubectl get application root -n argocd          # Synced / Healthy
   kubectl get applications -n argocd              # teamdemo-* без двойников
   ```

Дальше: правка `apps/*.yaml` → `git push` → авто-синк `root` (или `argocd app sync root`).

## Предпосылки

- Репозиторий **публичный** (ArgoCD читает без учёток). Приватный → `Secret` типа
  `repository` в ns `argocd`.
- Ветка — **`master`** (на неё указывает `bootstrap/root.yaml: targetRevision`).
  Если переедете на `main` — поправьте `targetRevision` там же.

## Грабли

- **Каскадный prune.** Удаление `team-services.yaml` из `apps/` → `root` снесёт ApplicationSet →
  тот по `prune` снесёт все `teamdemo-*` и их ворклоады. `finalizer` на `root` спасает только сам `root`.
- **`destination.namespace: argocd`** у `root` обязателен — Application/ApplicationSet создаются там.
- **selfHeal** окончательно переносит источник истины в git: ручной `kubectl edit` откатится.
- **Откат:** безопаснее не `kubectl delete application root --cascade`, а снять `automated`
  или закоммитить удаление с `preserveResourcesOnDeletion`, чтобы не утянуть нижний уровень.

## Соседние репозитории

| Репо | Что хранит |
|---|---|
| **argocd-bootstrap** (этот) | `root` + ApplicationSet'ы — платформенный конфиг ArgoCD |
| `argocd-teams-manifests` | `values.yaml` команд (`<commandID>/<service>/<component>/`) |
| `helm-charts` | общий чарт `simple` |
