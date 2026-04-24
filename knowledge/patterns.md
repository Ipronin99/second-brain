# Паттерны и грабли — что знать перед задачей
Обновлено: 2026-04-21

> Цель файла: когда садишься за задачу — читаешь этот файл, и не наступаешь на те же грабли. Здесь только неочевидные вещи; то, что derivable из кода, сюда не пишем.

---

## SecAudit — грабли и паттерны

### Git автор в SecAudit — НИКОГДА не менять
```
git config user.name  "Ipronin99"
git config user.email "goldenpython99@gmail.com"
```
Vercel привязан к GitHub `ipronin99`. Рабочий email `IgorPr@clearwayintegration.com` → Vercel **блокирует деплои**. Локальный config репо имеет приоритет над глобальным. Если глобальный когда-либо снова переключится на рабочий email — локальные значения продолжат работать.

### Прямой `supabase.auth.signUp()` из фронта — ЗАПРЕЩЁН
Обходит всю валидацию бэка (disposable email + password blocklist + IP rate limit). Правильно: `POST ${NEXT_PUBLIC_API_URL}/api/v1/auth/register` → затем `supabase.auth.signInWithPassword()` для сессии. Если email confirm включён в Supabase — signIn вернёт ошибку без throw (регистрация уже успешна, сессия поднимется после подтверждения).

Есть якорь `SYNC-ANCHOR` в `lib/passwordValidation.ts` (фронт) и `internal/validation/password.go` (бэк) — правила должны быть 1:1. Бэк — авторитетный, фронт — UX-подсказка.

### `profiles.id` ≠ `auth.users.id`
`profiles.id = gen_random_uuid()`, связь через колонку `auth_user_id`. Все бизнес-операции (documents, companies) идут с `profiles.id`, **не** с auth UUID. Auth middleware на каждом запросе резолвит auth UUID → profiles.id и кладёт в контекст. `companies.user_id` FK → `profiles.id`.

### `documents.content` — обязательно TEXT
Если поле `varchar(N)` — длинные документы **молча обрезаются** Supabase (вернёт 200 без ошибки). Миграция:
```sql
ALTER TABLE documents ALTER COLUMN content TYPE text;
```

### Прокси GNOME ломает Supabase
Бэкенд подхватывает системный прокси → Supabase недоступен → 401. Диагностика: `gsettings get org.gnome.system.proxy mode`. Запуск: `NO_PROXY="*" make run` (уже в Makefile). Для DEBUG-логов: `APP_ENV=development NO_PROXY="*" make run` (zap.NewProduction() скрывает DEBUG).

### `docs/` папка — swaggo генерация, НЕ коммитить
`docs/` генерируется `swag init` локально и в `.gitignore`. Если импортировать `_ "secauditai-backend/docs"` без закоммиченной папки → Railway упадёт с `package ... is not in std`. Swagger-аннотации в коде (`// @title ...`) безвредны без импорта. Swagger UI роут удалён.

### Rate limit — fixed-window, НЕ token bucket
`golang.org/x/time/rate` (token bucket) даёт плавный rate, не «3 в сутки». Для «N per 24h» — custom fixed-window counter (`map[ip]→{count, windowStart}` + `sync.Mutex`). 3 попытки / 24h / IP после `chimiddleware.RealIP`.

### Async document save
AI возвращает контент клиенту **немедленно**; запись в БД — в goroutine с timeout 120s. AI HTTP timeout — 300s. `WriteTimeout` сервера — 360s. Если goroutine упала — клиент уже получил документ, polling покажет `error`. Polling на фронте: `MAX_POLLS=1200` (10 мин), `pollToken` получается **ОДИН РАЗ до loop** (иначе перегрев getSession).

### В тестах только DeepSeek, не Anthropic
- Unit-тесты — через `httptest.NewServer` без реальных ключей.
- Integration DeepSeek — build-tag `integration`, skip если нет ключа.
- Integration Anthropic — build-tag `anthropic_integration` (**не захватывается `integration`**), только ручная команда владельца. В CI не запускается. Если в тесте появился прямой вызов Anthropic API или чтение `ANTHROPIC_API_KEY` для реальной генерации — **это баг**.

### RSS sanitize при поступлении + повторно при промпте
`defaultRSSFetcher` пропускает Title/Description/Link через `validation.SanitizeLine` в момент загрузки (primary boundary). `analyzeChanges` при построении промпта санитизирует ещё раз (defense in depth — на случай injectable fetchRSS в тестах). Тест `TestAnalyzeChanges_SanitizesRSSInputBeforePrompt` захватывает тело запроса и проверяет что `\n\n--- IGNORE ALL PREVIOUS INSTRUCTIONS ---` не проходит.

### `last_checked` обновляется даже если изменений нет
Иначе не отличить «стабильно» от «шедулер упал». `TouchAllLastChecked` делает один PATCH всем активным записям, ставит `last_checked=now()`. Упавший TouchAllLastChecked — `WARN`, не блокирует анализ.

### AI хuckyинирует статистику — запрет
В `director_letter` (и везде) LLM запрещено выдумывать конкретные цифры/проценты и атрибутировать их реальным источникам (Positive Technologies, BI.ZONE, Роскомнадзор). Разрешены hedge-формулировки + whitelisted числа (ФЗ-420 штрафы, 4,2 млн руб как средняя стоимость инцидента для МСБ). Причина — галлюцинации дискредитируют документ при проверке клиентом. 7 вводных hedge-фраз, правило «не повторять».

### ФСТЭК vs суд (юридическая точность)
**ФСТЭК НЕ дисквалифицирует директора напрямую** — это мера суда через КоАП. Во всех UI-тултипах и лендинге: «**предписание ФСТЭК и повторная проверка**», не «дисквалификация». 4 места: `CompanyForm`, `AuditCalculator` (×2), `page.tsx`. При любом новом упоминании санкций ФСТЭК — заменять на эту формулировку.

### Не называть конкурентов в маркетинге
ФЗ-38 «О рекламе». В таблице сравнения на лендинге: «Конструктор шаблонов» (= АльфаДок), «Enterprise-платформа» (= SECURITM) — без имён. Не упоминать Claude/иностранный ИИ клиентам (позиционирование: «интеллектуальная генерация документов»).

### Push — только после всех ролей АППРУВ
Процесс ревью — итерационный через Senior Dev. Senior Dev читает **весь изменённый код** (не diff), может советоваться с любой ролью / gstack-скиллом. Только после всех АППРУВ: `git push origin main && git push origin develop` — **ВСЕГДА в обе ветки**. Платёжные функции (`internal/payment/`, `/pricing`) — Security + Architect + Tester обязательны.

### Supabase Redirect URLs — обязательно `/**`
Reset-password / email confirm ссылки из писем требуют wildcard-суффикс:
- `https://secauditai.ru/**`
- `https://<vercel>.vercel.app/**`
- `http://localhost:3000/**` (локально)

### Two-step форма — AdditionalParams сбрасывается при смене doc_type
`useEffect` на `selectedType` ставит `additionalParams = {}`. Каждый doc_type имеет свой набор вопросов Step 2 (security_policy: has_remote/has_contractors/pd_count/ib_responsible; incident_response_plan: regulators/has_siem/working_hours; risk_matrix: risk_priorities; nda: nda_duration/nda_party; audit_checklist и director_letter — Step 2 не показывается).

### Lead magnet flow — sessionStorage, удалять после использования
`/audit` → `sessionStorage['audit_data']` → `/register` → email confirm → `/login` → `sessionStorage['audit_data'] ? '/generate' : '/documents'` → на `/generate` prefill из sessionStorage, **удалить после использования**, fallback `GET /companies/my`, потом пустая форма.

### Яндекс.Метрика gated на env
`NEXT_PUBLIC_YM_COUNTER_ID` — без ID блок не рендерится (прод не ломается до получения номера). noscript-fallback — img с `eslint-disable @next/next/no-img-element` (Метрика требует native 1×1 пиксель).

### Дизайн: danger vs warning — жёсткое разграничение
- **danger** = регуляторная опасность / необратимое / отказ: FSTEK-алерт, «предписание», «уголовная ответственность», form errors, destructive actions, «просрочен/отсутствует», крестики.
- **warning** = внимание без тревоги: денежные суммы (штрафы, риск в рублях), «частично», «перегенерировать». Деньги ≠ красный — это финансовое внимание.

Почему: на тёмном фоне красного много → перестаёт читаться как сигнал. Банковский/финтех язык позиционирования (trust-blue gosudarstvennyi) требует, чтобы красный был редким.

### Дизайн: эмодзи в UI — ЗАПРЕЩЕНЫ
Эмодзи (🛡️🤝📊🚨) только в user-generated content (юзер сам ввёл в название документа). В продуктовом UI — только `lucide-react`, stroke 1.5-2px.

---

## ESS — грабли и паттерны

### `ns.lock.Lock()` + slow I/O = гарантированный hang
Любой DB-transaction / network / HSM под `ns.lock.Lock()` блокирует pending writer на RWMutex → pending writer блокирует всех readers (health, pprof, API). Глобальное зависание HTTP >30s (ESS-2099). **Всегда**: in-memory mutation под лок → явный `unlock()` → I/O. Split write-and-persist pattern.

### Lock ordering: `ns.lock` → `cache.lock`
**Никогда** не брать `ns.lock.RLock()` под уже захваченным `cache.RLock()` — циклический deadlock через pending writer.

### `defer unlock()` — осторожно, если под локом I/O
Если под локом делается slow I/O — вручную `unlock()` перед I/O, не через defer. Error paths должны вызывать `unlock()` корректно до всех `return err`.

### Tainted namespace — защита после ESS-2099 resurrection fix (2026-04-17)
Все lock/unlock функции (`LockNamespace`, `UnlockNamespace`, `CustomLockNamespace`, `CustomUnlockNamespace`, `syncCustomLockState`) и `ModifyNamespaceByPath` отклоняют операции:
```go
if nsEntry.Tainted || ns.deletionMap[nsEntry.UUID] {
    return goerrors.Errorf("namespace %q is being deleted", nsEntry.Path)
}
```
Race window «DELETE из БД → ns.lock.Lock()» закрыт. `setNamespaceLocked` проверку не содержит — вызывается из легитимных путей через `taintNamespace` → `writeNamespace` напрямую.

### `Namespace.Clone(withUnlock)` — копирует Tainted
`helper/namespace/namespace.go:151` копирует ВСЕ поля включая `Tainted`/`Locked`/`CustomLocked`. `withUnlock` контролирует только `UnlockKey`/`CustomUnlockKey`. Клон из `getNamespaceByPathLocked` сохраняет taint-флаг → **проверять в вызывающем коде**.

### Panic при рестарте: taint-persist должен быть блокирующим (2026-04-19, фикс C)
Раньше `DeleteNamespace` (1134-1139) персистил Tainted=true **best-effort** — ошибка логировалась и проглатывалась. Goroutine всё равно стартовала, удаляла identity mount, а NS оставался без тента → рестарт → `i.views` пуст → nil-deref на `entityPacker.View()`. Фикс C: persist **блокирующий**, при провале — `rollbackTaint` + error клиенту. Никогда не запускать cleanup-goroutine с best-effort персистом.

### Fix B — view-check и nil-guard как defense-in-depth
В `loadIdentityStoreArtifacts` — view-check `i.views.Load(ns.UUID)` перед `loadEntities` (после tainted-check). Внутри `loadEntities`/`loadGroups` — nil-guard на `entityPacker`/`groupPacker`:
```go
entityPacker := i.entityPacker(ctx)
if entityPacker == nil {
    return goerrors.Errorf("entity packer not available for namespace %q", ns.Path)
}
```
Используем проверенный packer дальше в workerpool, не зовём `i.entityPacker(ctx)` повторно — та же поверхность для panic.

### Module-level логгер с захардкоженным уровнем — анти-паттерн в ESS (2026-04-24)
`var fooLogger = logging.NewVaultLogger(log.Trace)` (или `log.Debug`) на уровне пакета — **антипаттерн**. Такой логгер не слушает `log_level` конфига, не участвует в `SetLogLevelByName` / API `sys/loggers`. Typical-ошибка: оставлен со времён отладки. Правильно: поле `logger log.Logger` в структуре, инициализация в конструкторе прямо двумя строками: `logger := core.Logger().Named("subsystem-name"); core.AddLogger(logger)`. Без хелперов и nil-fallback — код для прода, не для тестов. Hex/base64-дампы чувствительных данных — **не нужен** guard `IsTrace()`, сам `Trace()` дропает сообщение на Info/Debug; аргументы вычисляются и отбрасываются, в sink не попадают. Guard имеет смысл только как perf-оптимизация на горячем пути, где аргумент дорогой (O(n) аллокация base64/sprintf на больших буферах). Примеры: исправлено — `vault/barrier_cryptopro.go` (было `Trace`). Осталось — `vault/barrier_openssl.go:124` (на `Debug`, чужая зона). Аналог правильного паттерна уже есть — `vault/core_cryptopro.go:149`: `cpWrapper.SetLogger(c.logger.Named("cryptopro-seal"))`.

### `disableCredentialInternal` использует `c.activeContext`, не incoming ctx
`vault/auth.go:293`: `revokeCtx := namespace.ContextWithNamespace(c.activeContext, ns)` — игнорирует переданный ctx (из deletion-pipeline). Значит при потере HA-lock (stepdown) → `activeCtxCancel()` → in-flight deletion-goroutines получают `context canceled`. В 4000-NS сценариях это ожидаемое поведение под нагрузкой, **не баг нашего фикса**. NS остаётся Tainted=true → `loadIdentityStoreArtifacts` пропускает → паники нет. Но это latent-issue: в бэклог пробросить `deletionJobContext` вместо `activeContext`.

### `writeTable` O(N) цикл по всем NS при unmount
`vault/auth.go:989-995`: при удалении mount цикл `for _, ns := range allNamespaces { view.Delete(ctx, path) }` — defensive cleanup. При 4000 NS это 4000 PG round-trips на одно unmount → расширяет окно context-cancel. Latent-issue, в бэклог оптимизации.

### `backend.Initialize()` НЕ вызывается при failover perf-standby → active
`ha.go:779` → `unsealMasterAfterBeingPerformance` (НЕ `postUnseal`) → `performance_unseal.go:15-100` делает только `postUnsealPhysical/startRollback/setupExpiration/setupSync/setupRaftActiveNode/startForwarding`. **`postUnsealFuncs` не реплеются, `backend.Initialize()` НЕ вызывается**. Следствие: если бэкенд прячет фоновые горутины на standby → при промоции нет хука чтобы их запустить.

### NotifyActive-хука для backends НЕТ
В SDK `logical.Backend` только `Initialize/HandleRequest/Cleanup/InvalidateKey/Setup`. Нет `OnPromote/NotifyActive/OnStandby`. Единственный механизм — опрос `b.System().ReplicationState()`. `c.replicationState.Store(ReplicationPerformancePrimary)` (`ha.go:778`) вызывается **ДО** `unsealMasterAfterBeingPerformance` → живая goroutine на следующем тике сразу видит Primary.

### ESS-1144: горутины на standby + per-tick gate
Стартуют всегда (чтобы не нужен был promotion-hook), но per-tick проверяют `ReplicationState()` и делают `continue` на standby:
```go
if sys := b.System(); sys != nil &&
    sys.ReplicationState().HasState(
        consts.ReplicationPerformanceStandby |
        consts.ReplicationPerformanceBootstrapping) {
    b.Logger().Debug("skipping ... on performance standby")
    continue
}
```
Nil-safety `b.System() != nil` — совместимость с тестом `TestStartCRLRotation_NilCRLClientAfterClean` (backend без Setup → `b.System() == nil`).

### В ESS-форке `disable_performance_standby` не распознаётся
Правильное имя — `enable_performance_standby = true` (`command/server/config.go:154`). Без блока `sync { conn_url = "..." }` `operator init` падает. Sync backend требует **отдельную БД** от storage.

### Health endpoint не заполняет `performance_standby`
Поле объявлено в `HealthResponse` но не сетится в `getSysHealth`. Чтобы понять что N2 реально perf-standby — смотреть лог `successful mount: ... essca/` (на обычном standby engine не монтируется, только perf-standby получает mount-sync).

### Unmount стирает config из storage
Config/attrs не в LocalStorage у essca. При перетестировании: **configure → restart N2** (НЕ unmount→config→remount).

### HA lock TTL 15s в postgres
После kill master ждать **~25-30s** до промоции standby.

### Java-тесты: `mvn clean` перед каждым прогоном
В `target/test-classes/` stale классы от предыдущих веток (`NoSuchFieldError` и т.п.).

### TCP-leak в doWithRetry (правильный паттерн)
```go
resp, err := c.httpClient.Do(req)
if err != nil { return nil, err }
if !isSuccess(resp.StatusCode) {
    io.Copy(io.Discard, resp.Body)  // читаем до EOF
    resp.Body.Close()               // возвращаем коннект в пул
    return nil, someError
}
return resp, nil  // caller делает defer resp.Body.Close()
```
Даже при non-2xx нужно `io.Copy(io.Discard)` + `Close()` — иначе соединение остаётся ESTABLISHED до TCP timeout ОС.

### IdleConnTimeout + CloseIdleConnections при замене клиентов
`IdleConnTimeout: 0` (дефолт) → idle соединения **никогда** не закрываются. `pathWriteAttrs` и `Clean()` заменяют клиентов — вызывать `Transport.CloseIdleConnections()` иначе старые ESTABLISHED висят.

### WorkerPool lifecycle
`backend.Clean()` должен вызывать `b.monitor.Stop()` (останавливает пул + timer-горутину). Stop должен быть идемпотентным через guard на `m.cancel == nil`. Nil-guard на `workerPool.Stopped()` если `wpool.NewPool` вернул ошибку.

### `time.Sleep(jitter)` в RoleMonitor — goroutine leak
Если не уважает `ctx.Done()` — при defaultInterval=24h и jitter до 2.4h горутина живёт часами после `Clean()`:
```go
select {
case <-time.After(jitter):
case <-rm.ctx.Done():
    return
}
```

### Security / Architecture review обязательны на shared API
Любое изменение exported API в `api/`, `sdk/` — shared contract, ломает других разработчиков. Security + Architect review перед мержем.

### Чужие зоны — не трогать, только фиксировать
- CryptoPro trace logging (`fix/ess-cryptopro-trace`)
- readinessProbe/mTLS (ESS-CI `feature/readinessProbe`)
- ESS-1152 ExpiresAt/Cleanup/retry_policy (`fix/ExpiresAt-fix`)
- ESS-2071 pgxpool label

Нашёл баг в чужой зоне → фиксировать в KNOWLEDGE.md + сообщить автору, не править самому.

### beginTransaction может вернуть nil, nil (InmemBackend)
`TestCluster` использует InmemBackend без TransactionalStorage. `beginTransaction` → `nil, nil`. Всегда nil-guard: `if txn != nil { req.Storage = txn; defer rollback }`. Graceful fallback без транзакции — работает на хранилищах без TransactionalStorage.

---

## Общие — оба проекта

### Коммит и push — только Игорь
Claude Code собирает изменения + аппрувы, Игорь делает `git commit` и `git push`. Не вызывать `git push` от своего имени без явной просьбы (ESS — жёсткое правило; SecAudit — после всех АППРУВ ролей).

### Проект — не писать код вручную
В SecAudit: «Не пишем код вручную (только Claude Code в GoLand)». Игорь редактирует через диалог, не через прямой ввод.

### Логи — никаких секретов
Секреты **никогда** в логах, **никогда** в error-messages. Для сравнения — `crypto/subtle`. Этот принцип общий для обоих проектов.

### TRACE уровень для отладки, не INFO/DEBUG
В ESS: предпочтительно `TRACE` для отладочной информации (паттерн ветки `fix/ess-cryptopro-trace`). В SecAudit: DEBUG через `APP_ENV=development` (zap.NewProduction скрывает DEBUG в prod).

### Пустые файлы в vault оставлять пустыми не стоит — заполнять на основе KNOWLEDGE.md источников
Vault `/home/igor/go/second-brain/` — источник для новых сессий. Если файл пустой и есть обновления KNOWLEDGE — обновлять, чтобы не было расхождений.

### Дата в памяти — обновлять относительные → абсолютные
При записи в vault/память «через неделю», «в четверг» → превращать в YYYY-MM-DD (`2026-04-28`, `2026-04-23`), иначе запись теряет смысл через неделю.

### Память стареет
Перед тем как рекомендовать что-то из памяти — проверять что файл/функция/флаг ещё существует (grep, Read). «Память говорит X существует» ≠ «X существует сейчас».
