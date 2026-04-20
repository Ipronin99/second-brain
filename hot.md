# hot.md — текущий контекст
Обновлять: после каждой сессии GoLand + раз в неделю вручную
Последнее обновление: 2026-04-21

## Статус (апрель 2026)

### Второй мозг
- Vault: `/home/igor/go/second-brain/` ✅
- Заполнены: `projects/secaudit/context.md` + `projects/secaudit/design.md` + `projects/ess/context.md` + `knowledge/patterns.md` ✅
- Правило обновления vault добавлено в `CLAUDE.md` трёх проектов (backend, frontend, ess) ✅
- MCP Obsidian для GoLand: подключён ✅

### SecAudit
- MVP задеплоен на secauditai.ru ✅
- ЮКасса интегрирована в код, нужны тестовые ключи в Railway ⏳
- Миграция 006 НЕ выполнена в Supabase ⏳
- 73 теста зелёных ✅
- Следующее: добавить ключи → миграция 006 → smoke test → тестовый платёж

### ESS
- Форк OpenBao, домен PKI/криптография
- Путь: `/home/igor/go/ess`
- Ветка `release-1.4.0-pilot`: бинарь `bin/ess_openssl` собран на Go 1.25.7 с тегом `openssl` (2026-04-20)
- Ветка `feature/ESS-2099-master-stop`: fix B+C применён (коммит `861a601a`), QA проверка 4000 NS прошла без паники (2026-04-20)
- Новый лог `"failed to remove credential entry" / context canceled` при mass-delete — расследован: НАБЛЮДЕНИЕ, не проблема (HA-lock churn под нагрузкой, к нашему фиксу не относится)

## Приоритеты
1. 🔴 Эндокринолог (пролактин 440, норма 407)
2. 🔴 Миграция 006 Supabase
3. 🔴 ЮКасса тестовые ключи → Railway
4. 🟡 Smoke test + тестовый платёж
5. 🟡 Анонс в Telegram
