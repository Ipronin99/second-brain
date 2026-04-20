# ESS — Полный контекст
Обновлено: 2026-04-21
Источник: `/home/igor/go/ess/KNOWLEDGE.md`

## Что за проект
**ESS (ess-server)** — форк OpenBao/Vault (модуль `gitlab.clearwayintegration.com/cwi/ess-group/ess-server`). Secrets management / PKI / Vault-совместимый сервер. Хранит и раздаёт секреты, сертификаты, ключи; поддерживает динамические секреты, leasing/renewal, revocation, transit-шифрование.

**Рабочий проект** (не личный). Гитлаб: `gitlab.clearwayintegration.com`. Автор: Игорь (IgorPr@clearwayintegration.com) — только для этого проекта.

## Технический стек
- **Go 1.24.13** (см. go.mod); на ветке `release-1.4.0-pilot` go.mod уже 1.25.7.
- Модуль `gitlab.clearwayintegration.com/cwi/ess-group/ess-server`
- Ключевые зависимости: `caddyserver/certmagic`, `hashicorp/cap`, `go-jose/go-jose/v4`, `golang-jwt/jwt`, `denisenkom/go-mssqldb`, `go-sql-driver/mysql`, `gocql/gocql`, `google/cel-go`, `cenkalti/backoff`, `alitto/pond/v2`, `armon/go-metrics|go-radix`, `go-ldap`, `ProtonMail/go-crypto`
- Локальные replace: `api`, `sdk`, `helper`, `api/auth/{approle,kubernetes,userpass}`
- Storage backends: PostgreSQL (основной), file и др.

## Архитектура

### Основные каталоги
- `api/`, `sdk/`, `helper/` — клиентские/вспомогательные модули
- `vault/` — **ядро**: namespace_store, seal, unseal, router, policy, token store, expiration/leases, rollback manager, barrier
- `command/` — CLI и `ess server`
- `http/` — HTTP handlers/роутинг, wrapping (rateLimit, audit)
- `audit/` — audit log
- `physical/` — storage backends (postgresql, cache layer)
- `builtin/logical/` — secrets engines: `pki`, `transit`, **essca** (кастомный CA), `kv`
- `builtin/credential/` — auth: approle, token, userpass, kubernetes
- `builtin/audit/` — audit backends
- `plugins/`, `serviceregistration/`, `internalshared/`

### Критичные правила архитектуры

**Namespace lock protocol** (`vault/namespace_store.go`):
- Глобальный RWMutex `ns.lock` защищает in-memory map/tree.
- Любой HTTP-запрос берёт `RLock` через `ResolveNamespaceFromRequest`.
- **НИКОГДА** не делать slow I/O (DB transaction, network, HSM) под `ns.lock.Lock()`. Нарушение → глобальное зависание HTTP (ESS-2099).

**Split write-and-persist**: in-memory mutation под `ns.lock.Lock()` (микросекунды) → явный `unlock()` → DB-персистенс. Паттерн: `taintNamespaceInMemory` + последующий `writeNamespace(ctx, nil, nsCopy)` вне лока. Идемпотентность критична — при падении между unlock и DB-записью операция должна быть повторяема.

**Lock ordering**: `ns.lock` → `cache.lock` безопасно. Никогда не брать `ns.lock.RLock()` под уже захваченным `cache.RLock()` (циклический deadlock через pending writer на RWMutex).

**Defer для unlock — осторожно**: если под локом делается slow I/O → вручную `unlock()` перед I/O.

### Namespace deletion pipeline
`DeleteNamespace` handler → taint (in-memory + persist) → асинхронная горутина `func1` под `deletionLock` → `clearNamespaceResources` (unmount mounts, revoke tokens/leases) → `s.Delete(uuid)` **ВНЕ `ns.lock`** → `ns.lock.Lock()` → удаление из трёх in-memory индексов (`namespacesByPath`, `namespacesByUUID`, `namespacesByAccessor`) + снятие `deletionMap[uuid]` → `ns.lock.Unlock()` → `RemoveDomainBackend` вне `ns.lock`. `deletionJobGroup` отслеживает активные удаления.

### Two-phase write в setNamespaceLocked (`vault/namespace_store.go:404`)
Функция сама управляет `ns.lock` через defer/unlocked-флаг. Caller уже держит `ns.lock.Lock()` и передаёт ответственность по раскладке (паттерн `unlock = nil` перед вызовом). Внутри: валидация → insert → `writeNamespace` (DB Put) → commit. При создании net-new — `Unlock()` между memory-insert и `writeNamespace` (чтобы `initializeNamespace` мог взять lock), `cleanupFailed` возвращает lock и откатывает in-memory. Используется из: `SetNamespace`, `CreateNamespaceInternal`, `LockNamespace`, `UnlockNamespace`, `CustomLockNamespace`, `CustomUnlockNamespace`, `syncCustomLockState`, `ModifyNamespaceByPath`.

### Защита от модификации tainted namespace (ESS-2099 resurrection fix, 2026-04-17)
`ModifyNamespaceByPath`, `LockNamespace`, `UnlockNamespace`, `CustomLockNamespace`, `CustomUnlockNamespace`, `syncCustomLockState` — все отклоняют операции на tainted NS:
```go
if nsEntry.Tainted || ns.deletionMap[nsEntry.UUID] {
    return goerrors.Errorf("namespace %q is being deleted", nsEntry.Path)
}
```
Сразу после `getNamespaceByPathLocked` и до любой мутации. Импорт `goerrors "github.com/go-errors/errors"`. `setNamespaceLocked` проверку не содержит (вызывается с tainted=false из `SetNamespace`/`CreateNamespaceInternal` + через `taintNamespace` → `writeNamespace` напрямую).

### deletionMap как in-memory маркер
`map[uuid]bool` под `ns.lock`. Ставится в `DeleteNamespace` до запуска goroutine, снимается goroutine после попытки DB-delete. Используется только в `DeleteNamespace` (возврат `"in-progress"` при повторном DELETE).

### Namespace.Clone(withUnlock)
(`helper/namespace/namespace.go:151`) Копирует ВСЕ поля включая `Tainted`, `Locked`, `CustomLocked`. `withUnlock` контролирует только копирование `UnlockKey`/`CustomUnlockKey`. Клон из `getNamespaceByPathLocked` сохраняет taint-флаг → его **можно и нужно проверять** в вызывающем коде.

### Transit auto-unseal
См. `local/transit/` — отдельный Vault-инстанс для unseal основного. Требует CryptoPro HSM.

### Migration engine
См. `MIGRATION_ENGINE_DOC.md` (ESS-1152, ESS-1861, ESS-1980, ESS-2035) — миграция под custom-lock, revoke leases, retry_policy.

## ESSCA engine (builtin/logical/essca)

**Factory** (`backend.go:22`): вызывается для каждого mount на любом узле, включая performance standby. Создаёт `CertificateMonitor`, сохраняет в `b.monitor`, запускает goroutine с джиттером.

**RoleMonitor.run** (`monitor.go:392`): внутренний jitter `interval/10`, бесконечный цикл `ticker.C` → `performMonitoringCycle` → `checkAllCertificates` → `issueCertificateIfNeeded` → `pathIssueSingle`.

**startCRLRotation** (`crl_cache.go:358`) и **startPolicyRotation** (`esaus_policies.go:173`) — стартуют фоновые goroutines-тикеры на всех узлах.

### ESS-1144 фикс: per-tick gate на perf-standby
- **Проблема**: монитор/CRL/policy rotation работали на perf-standby → ошибки записи в read-only барьер.
- **Фикс**: per-tick проверка `consts.ReplicationPerformanceStandby | ReplicationPerformanceBootstrapping` в трёх ticker-сайтах (`monitor.go:431`, `crl_cache.go:391`, `esaus_policies.go:197`). Debug-лог «skipping ... on performance standby», `continue`.
- **Важно**: jitter-goroutine в `backend.go:50-76` **СОЗНАТЕЛЬНО** не гейтится — RoleMonitor goroutines должны существовать на standby для авто-возобновления при failover. В ESS-форке НЕТ NotifyActive-хука, единственный механизм — опрос `ReplicationState()`.

### ESS-1991 TCP-leak + WorkerPool lifecycle
Правильный паттерн `doWithRetry`:
```go
resp, err := c.httpClient.Do(req)
if err != nil { return nil, err }
if !isSuccess(resp.StatusCode) {
    io.Copy(io.Discard, resp.Body)  // читаем до EOF
    resp.Body.Close()               // возвращаем коннект в пул
    return nil, someError
}
return resp, nil
```
- CloseIdleConnections при замене клиентов (`pathWriteAttrs`, `Clean()`).
- `b.monitor.Stop()` в `backend.Clean()` — закрывает WorkerPool и timer-горутину.
- Nil-guard на `workerPool.Stopped()` если `wpool.NewPool` вернёт ошибку.
- `crlUrlsMu sync.RWMutex` + `getCRLUrls()/setCRLUrls()` — закрытие data race на `backend.crlUrls`.
- `RoleMonitor.run` jitter должен уважать `ctx.Done()`: `select { case <-time.After(jitter): ; case <-rm.ctx.Done(): return }` — иначе goroutine leak с defaultInterval=24h.

## Performance standby (КРИТИЧНО)

### При failover perf-standby → active НЕ вызывается postUnseal
- `ha.go:779` — если `perfStandby.Load()==true` и `syncBackend!=nil` → `unsealMasterAfterBeingPerformance` (НЕ `postUnseal`).
- `performance_unseal.go:15-100` делает только: `stopStandbyHealthWatcher` → `dirtyReloadAuditLogs` → `physicalCache.Purge` → `postUnsealPhysical` → `startRollback` → `setupExpiration` → `setupSync` → `setupRaftActiveNode` → `startForwarding`.
- **`backend.Initialize()` у mount-ов НЕ вызывается**. `postUnsealFuncs` не реплеются.

### Что это значит для backends
Если не запускать горутины на perf-standby → нет хука чтобы их запустить при промоции. Варианты:
(a) горутины стартуют всегда + per-tick gate (ESS-1144 подход) ✅
(b) ленивый старт при первом HTTP-запросе (не запустит фон пока нет запроса)
(c) patching core — чужая зона

### replicationState при promotion
`ha.go:778` — `c.replicationState.Store(ReplicationPerformancePrimary)` вызывается **ДО** `unsealMasterAfterBeingPerformance`. Живая горутина на следующем тике сразу видит Primary, per-tick gate перестаёт срабатывать без рестарта. Весь «механизм уведомления» = атомарный флаг + опрос.

### Механизмы, которые НЕ являются NotifyActive-хуком
- `c.serviceRegistration.NotifyActiveStateChange(bool)` — только для service discovery (Consul/K8s/etcd).
- `newLeaderCh` — внутренний канал HA, недоступен backends.
- `InvalidateKey(key)` — per-key invalidation, не role event.
- `standbyStopCh/standbyDoneCh/standbyRestartCh` — внутренние каналы runStandby.
- SDK `PeriodicFunc` — вызывается на всех нодах, тоже требует per-tick проверки.

## Локальная разработка

### Docker-окружение
См. `LOCAL-DEPLOYMENT.md`. `make docker-up` → автоматический init/unseal/login. Health: `curl http://127.0.0.1:8200/v1/sys/health`.

### Конфиги
- `config.hcl` (основной), `config.local.hcl` — PostgreSQL storage, TCP listener на 8200.
- `.env`: `POSTGRES_USER=exc_user`, `POSTGRES_PASSWORD=exc`, `POSTGRES_DB=exc_db`.
- `~/.netrc` и `~/gitlab.clearwayintegration.com.crt` для доступа к приватному gitlab.

### Директория local/
- `local/nodes/{node1,node2}` — multi-node тесты с transit auto-unseal через CryptoPro HSM
- `local/transit/` — vault.hcl, env.sh, unseal.sh для auto-unseal
- `local/test_hang/` — **двухнодовый кластер БЕЗ transit (shamir seal)** — для ESS-2099 и ESS-1144
  - `node{1,2}/config.hcl` — `enable_performance_standby=true`, блок `sync` (иначе init упадёт), N1:8210, N2:8220, PG:5432
  - `scripts/` — helper-скрипты repro→fix→verify
  - `logs/` — артефакты прогонов

### Cluster reproduction scripts (`local/test_hang/scripts/`)
- `00_recreate_dbs.sh` — DROP+CREATE двух БД (storage + sync)
- `01_build_binaries.sh` — `bin/ess_fixed` (текущее дерево) + `bin/ess_nofx` (стэш 3 файлов)
- `02_start_cluster.sh <bin>` — поднимает N1+N2, init+unseal, токены в `local/test_hang/state/`
- `03_configure_essca.sh` — 7 curl-команд ESS-1144 (mock CA на 8081, interval 5с)
- `04_observe.sh` — сводка: счётчики skip-логов, реальной работы, leader/health. **Главный диагностический**
- `05_failover_test.sh <bin>` — kill N1 → промоция N2 → N1 как standby
- `99_stop_cluster.sh` — pkill обеих

### Подводные камни кластера
- В форке `disable_performance_standby` **не распознаётся**, правильное имя `enable_performance_standby=true` (`command/server/config.go:154`).
- Без блока `sync { conn_url = "..." }` `operator init` падает. Sync backend требует **отдельную БД** от storage.
- Health endpoint **не заполняет `performance_standby`** поле. Чтобы убедиться что N2 реально perf-standby — смотреть лог `successful mount: ... essca/` (на обычном standby engine не монтируется).
- Unmount в Vault стирает config из storage (config/attrs не в LocalStorage у essca). При перетестировании: configure → restart N2 (а не unmount→config→remount).
- HA lock в postgres имеет **TTL 15s**. После kill master ждать ~25-30s до промоции.

### Java-тесты (`exc-at-tests-java`)
- Запуск: `mvn test` (JUnit5, RestAssured, Allure). Конфиг: `src/main/resources/test.properties`.
- Maven без sudo: `~/tools/apache-maven-3.9.6/bin`.
- **Перед каждым прогоном `mvn clean`** — в `target/test-classes/` stale классы от предыдущих веток (`NoSuchFieldError`).
- ESSCA покрытие: `ess/secret/essca/e2e/DriverUCTest.java`, `EsausTest.java`, `function/attrs/*`, `function/role/*`, `function/cert/*`, `function/esausPolicies/*`.
- Скрипт: `local/test_hang/run_java_tests.sh <bin> <label>`.

### Ожидаемые падения тестов на локалке (НЕ регрессии)
- `CrlRotateTests.test1` — требует VTB внутренняя сеть
- `DeleteAttrsFunctionTests.test8/9/10`, `FetchPEMCaDriverTests.test11/13/224` — identity entity disable/delete + token TTL (флейк на локалке)
- `ess.secret.essca.e2e.*` — требуют реальный VTB CA + ESAUS

## Команды
- `make bootstrap` — build tools
- `make` / `make dev` — собрать бинарь с `testonly` тегом в `bin/`
- `make static-dist dev-ui` — с UI
- `make test TEST=./vault` — тесты пакета (требует Docker)
- `make testacc TEST=./builtin/logical/pki` — acceptance
- `make testrace`, `make lint`, `make lint-new`, `make vet`, `make cover`
- `go test -run TestX ./pkg/...` — один тест
- Java integration: `VAULT_BINARY=$(pwd)/bin/ess go test -run 'TestRaft_Configuration_Docker' ./vault/external_tests/raft/raft_binary`

## Соглашения
- Commits: `feat: ...`, `fix: ...`, часто с тикетом `ESS-NNNN`, на русском
- Branches: `feature/ESS-NNNN-<slug>`, `fix/...`, `bugfix/...`. Основная — `develop`, прод — `master`.
- gofmt + golangci-lint. Без кастомных линтеров.
- Логирование: `hclog`-совместимое. Уровни — `TRACE` для отладки (см. `fix/ess-cryptopro-trace`), не INFO/DEBUG.
- Секреты **никогда** в логах/error-messages. `crypto/subtle` для сравнения.
- `defer` для unlock осторожно — при slow I/O явный `unlock()` до I/O.

## Зоны ответственности
- **Зона Игоря (текущая)**: ESS-2099 master-hang (namespace delete, ns.lock, cache layer). Ветка `feature/ESS-2099-master-stop`. См. `BUGFIX_REPORT.md`.
- **Исторические Игоря**: ESS-1861 (migration unmount), ESS-1975, ESS-1980 (custom namespaces migration), ESS-2035 (leas revoke migration), ESS-2096 (mTLS migration) — все unstaged .md в корне.
- **Чужие зоны (НЕ ТРОГАТЬ, только фиксировать)**:
  - CryptoPro trace logging (`fix/ess-cryptopro-trace`)
  - readinessProbe/mTLS (ESS-CI `feature/readinessProbe`)
  - ESS-1152 ExpiresAt/Cleanup/retry_policy (`fix/ExpiresAt-fix`)
  - ESS-2071 pgxpool label
- **Общее**: `api/`, `sdk/` — shared contract. Любое изменение exported API → security + architect review.

## Правила ревью (`.reviewers/ess/REVIEW_PROCESS.md`)

### Уровень 1 — Senior Dev (точка входа и выхода)
Senior Dev — единственная роль, которая запускает и завершает ревью. Читает ВЕСЬ изменённый код (не diff). Цикл:
1. Senior Dev читает код → находит проблему → возврат на доработку
2. Код исправляется
3. Senior Dev снова читает
4. Если ок — переходит к ролям
5. Вызывает нужные роли (security, architect, tester, techlead, devops)
6. Каждая проводит анализ
7. Проблема → Senior Dev → возврат
8. Все АППРУВ → Senior Dev выносит итоговый вердикт «МОЖНО ПЕРЕДАВАТЬ ИГОРЮ»

**Коммит и push делает только Игорь.** Chain собирает аппрувы, не коммитит.

### Маппинг ролей на скиллы
- senior_dev → `/review`
- security → `/cso` (OWASP+STRIDE)
- tester → `/qa`, `/health`
- architect → `/investigate`, `/health`
- techlead → `/health`, `/retro`
- devops → `/health`

### При обнаружении проблем вне задачи Игоря
Только фиксируем и сообщаем, не правим. **Чужой код не трогаем никогда.**

## Известные баги и решения

### ESS-2099: Master hang при массовом delete namespace
- **Симптом**: все HTTP (health, pprof) в timeout >30s после ~150-200 удалений.
- **Причина**: `ns.lock.Lock()` удерживался во время DB-транзакции (`taintNamespace` → `writeNamespace` → `cacheTransaction.Commit`). Pending writer на RWMutex блокирует readers.
- **Места блокировки**: `DeleteNamespace` handler (~1053) + goroutine `func1` (~1099-1100).
- **Фикс v2**: `taintNamespaceInMemory` — только maps/tree под лок; `writeNamespace` + `RemoveDomainBackend` после явного `unlock()`; `defer unlock()` заменён на явный.
- **Результат**: avg health 169→21ms; max 30020ms→98ms; 0 зависаний на 1025 Java-тестах.
- **Инсайт**: любой новый код под `ns.lock.Lock()` → обязательный чек «есть ли I/O?». Split in-memory/persistence phases.

### ESS-2099 resurrection race (2026-04-17)
- **Race**: между `s.Delete(uuid)` (~1156 в goroutine, 10-200ms) и `ns.lock.Lock()` (~1165) `ns.lock` свободен. Параллельный Lock/Unlock/CustomLock/syncCustomLockState мутирует клон → `setNamespaceLocked` → `s.Put` — запись восстаёт.
- **Фикс**: в 5 функций после `getNamespaceByPathLocked`+nil-check добавлена проверка `Tainted || deletionMap[UUID]` с ошибкой `"is being deleted"`.
- Test: `TestNamespaceStore_RejectsOperationsOnTaintedNamespace` (5 функций × 3 сценария).

### ESS-??? Restart panic: nil entityPacker on half-deleted namespace (2026-04-19)
- **Симптом**: после Java-тестов с `IS_PARENT_NAMESPACE=true` и рестарта ESS — `failed to get entityPacker` → panic в `storagepacker.(*StoragePacker).View` из `loadEntities` в `readonlyStrategy.unsealInternal`.
- **Root-cause A (основной)**: `DeleteNamespace` (1134-1139) persist Tainted=true **best-effort** — ошибка логируется и проглатывается. Goroutine идёт на `clearNamespaceResources` → `unmountInternal("identity/")` → `RemoveNamespaceView`. Если `s.Delete` падает или рестарт между этим и `ns.lock.Lock()` — на диске NS с Tainted=false, identity-mount удалён. После рестарта: NS без тента → `setupMounts` не находит mount → `AddNamespaceView` не вызван → при `loadEntities` `i.views` пуст → **nil-deref**.
- **Root-cause B (custom storage)**: для top-level NS с custom storage (`namespacePhysicalMux`) factory identity-mount может вернуть `NamespaceDomainBackendUnavailableError` → placeholder backend без `AddNamespaceView` → аналогичная панNYCI. Вектор не подтверждён в тестах (`IS_PARENT_NAMESPACE` — про вложенность, не про custom storage), оставлен как дополнительный.
- **Фикс B** (`vault/identity_store_util.go`): в `loadIdentityStoreArtifacts` — view-check `i.views.Load(ns.UUID)` перед `loadEntities`. Внутри `loadEntities`/`loadGroups` — nil-guard на `entityPacker`/`groupPacker` (returns `goerrors.Errorf(...)`).
- **Фикс C** (`vault/namespace_store.go`): `DeleteNamespace` делает persist taint **блокирующим**. При провале — helper `rollbackTaint(nsEntry)` (сброс Tainted в 3 индексах + `delete(deletionMap)` + `deletionJobGroup.Done()`) + клиенту возвращается error. Идемпотентность: повторный DELETE после восстановления storage работает штатно.
- **Коммит**: `861a601a` (название «fix: поправил утечку tcp коннектов» — фактически содержит и B+C).

### Массовый delete: `context canceled` при removeCredEntry (QA 2026-04-20)
- **Лог**: `failed to remove credential entry for path being unmounted: error="failed to update auth table: requested removal of auth mount from namespace DyzujM (831) but failed: context canceled"`.
- **Граф**: `DeleteNamespace` goroutine → `clearNamespaceResources` → `disableCredentialInternal` (игнорирует incoming ctx, строит `revokeCtx := ContextWithNamespace(c.activeContext, ns)` на `auth.go:293`) → `removeCredEntry(revokeCtx)` → `persistAuth` (PG транзакция) → `writeTable` цикл по ВСЕМ namespaces (`auth.go:989-995`) → `view.Delete(ctx, path)` на итерации 831 возвращает `context.Canceled`.
- **Кто отменяет**: **`c.activeContext`**, а не `deletionJobContext`. Отменяется в `ha.go:859,870` (потеря HA-lock), `core.go:2254-2278` (seal), `core.go:2658` (postUnseal error defer). В 4000-NS сценарии — HA-lock churn под write-нагрузкой (PG TTL 15s, heartbeat опаздывает).
- **Связь с паникой сценария A: НЕТ**. Taint уже персистнут ДО запуска goroutine (fix C). NS в БД остаётся Tainted=true → при рестарте `loadIdentityStoreArtifacts` пропускает через `if ns.Tainted { continue }`.
- **Финальное состояние**: NS `Tainted=true` ✓; auth entry Tainted=true (персист до removeCredEntry); view auth очищен; identity mount НЕ удалён; `deletionMap[uuid]` очищен.
- **Идемпотентность повтора**: повторный DELETE подхватывает tainted NS и продолжает cleanup.
- **Вердикт**: **наблюдение, не проблема**. Симптом leader-instability + pre-existing O(N) цикл в writeTable. Оба — вне ESS-2099. В бэклог: (1) исследование HA-lock churn под 4000 NS, (2) оптимизация writeTable цикла, (3) пробросить `deletionJobContext` вместо `activeContext` в `disableCredentialInternal`.

## Важные файлы
- `vault/namespace_store.go` — namespace CRUD, lock protocol; **hot path ESS-2099**
- `physical/cache.go` — storage cache layer с RWMutex
- `command/server/config.go` — конфиг сервера (в корне лежат `.orig/.rej` — незавершённый merge)
- `builtin/logical/essca/` — кастомный CA engine
- `BUGFIX_REPORT.md` — полный разбор ESS-2099
- `MIGRATION_ENGINE_DOC.md`, `HARD_LOCK.md`, `ESS-*.md` — per-ticket дизайн-доки
- `local/test_hang/run_java_tests.sh` — воспроизведение ESS-2099
- `config.hcl` + `LOCAL-DEPLOYMENT.md` — локальный запуск

## Правило обновления второго мозга
После каждой задачи по ESS — обновлять `/home/igor/go/second-brain/`:
- `hot.md` — всегда (что сделал, статус)
- `knowledge/patterns.md` — если новый паттерн/грабля
- `projects/ess/context.md` — при изменении архитектуры, правил, зон
- `decisions/` — важные архитектурные решения
