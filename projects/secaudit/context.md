# SecAudit AI — Полный контекст
Обновлено: 2026-04-21
Источники: KNOWLEDGE.md обоих репо + PRODUCT.secauditai.md

## Что за продукт
SaaS генерация ИБ-документации за 5 минут. 6 типов документов: политика ИБ, NDA, матрица рисков, план реагирования на инциденты, чеклист аудита, письмо директору о необходимости ИБ. Домен: **secauditai.ru**. Состояние на апрель 2026 — MVP задеплоен, архитектура стабильна, все P0/P1 security-замечания закрыты, ждёт ЮКассу и soft launch.

## Целевая аудитория и позиционирование
ИБ-специалисты и сисадмины в компаниях 50–500 человек. Главный сегмент — **«одинокий безопасник»** (~50-80 тыс. в РФ, ЗП ~230к/мес ≈ 1400 руб/час; ручная политика ИБ = 16-24ч = 22-34к руб → SecAudit 2990 → ROI 7-10x).

Позиционирование — «проверьте готовность к проверке ФСТЭК». Говорить: «политика ИБ за 5 минут вместо 3 дней», НЕ «как Vanta». Ценность — готовый документ для регулятора, не «wow-эффект стартапа».

## Тарифы и финансовая модель
- **Free** — 0 руб, 1 документ без карты
- **Старт** 1490/мес — 5 документов, все типы
- **Про** 2990/мес — безлимит + история + DOCX/PDF (флагман, «Популярный»)
- **Команда** 6990/мес — до 3 компаний
- Годовая — скидка 20-25%

Roadmap MRR: мес2=5к → мес3=12к → мес6=80к → мес9=200к → мес12=350к. Точка независимости — 175к MRR (~60 Про). **Kill criteria**: 0 платящих через 60 дней → пивот. **Scale**: MRR 30к и растёт + retention >60%.

## Архитектура: Backend (Go)

### Стек
- **Go 1.25** + **chi v5** HTTP router
- **Supabase** (PostgreSQL + Auth, Frankfurt eu-central-1)
- **Claude API** (основной) + **DeepSeek** (резервный, используется в тестах — экономия)
- Логирование: **zap** structured JSON

### Ключевые архитектурные решения
- **Регистрация проксируется через Go-бэк**: `POST /api/v1/auth/register` → Supabase `/auth/v1/signup`. Добавляет слой валидации (disposable email + блок-лист паролей + IP rate limit). Прямой `supabase.auth.signUp()` на фронте **запрещён** — обходит защиту.
- **profiles.id ≠ auth.users.id**: `profiles.id = gen_random_uuid()`, связь через колонку `auth_user_id`. Auth middleware на каждом запросе резолвит auth UUID → profiles.id. Все бизнес-операции (documents, companies) идут с `profiles.id`.
- **Генерация асинхронна**: AI возвращает результат клиенту немедленно; запись в БД — в goroutine с таймаутом 120s. AI HTTP timeout — 300s (5 мин) для длинных документов.
- **WriteTimeout HTTP сервера**: 360s — увеличен под AI.

### Эндпоинты
Публичные: `GET /health`, `POST /api/v1/auth/register`, `POST /api/v1/audit/check`.
Защищённые (Bearer JWT): `POST /companies`, `GET /companies/my`, `GET /companies/{id}`, `POST /documents/generate`, `GET /documents`, `GET /documents/{id}`, `DELETE /documents/{id}`, `GET /dashboard`.

### База данных (Supabase)
- `auth.users` — Supabase auth
- `profiles` — id=uuid gen_random_uuid(), auth_user_id FK→auth.users
- `companies` — user_id FK → **profiles.id**
- `documents` — doc_type (security_policy, nda, risk_matrix, incident_response_plan, audit_checklist, **director_letter**), generation_status (pending/generating/done/error)
- `document_expiry` — сроки действия
- `legal_knowledge` — Legal RAG, см. ниже

**Check constraints**: doc_type массив фиксирован (см. миграцию 001). `documents.content` **обязательно TEXT, не varchar** — иначе длинные документы молча обрезаются Supabase (вернёт 200 без ошибки).

### Legal RAG + RSS-шедулер
- `public.legal_knowledge` — содержит 11 законов (+Q1 2026 дополнения: РП №360-Р «типовые КИИ», ФСТЭК №117 по ИИ). Колонки: `law_id`, `title`, `content`, `source_url`, `doc_types[]`, `effective_date`, `last_updated`, `last_checked`, `is_active`.
- `BuildLegalContext(docType)` подбирает релевантные законы → инжектится в промпт через `ai.GenerateRequest.LegalContext`.
- **RSS-шедулер** (`internal/scheduler/legal_update.go`) крутится по воскресеньям 03:00 UTC. Источники: `publication.pravo.gov.ru/RSS/` + `fstec.ru/RSS` (парсер — gofeed). Window: последние 7 дней. Keywords: 152-ФЗ, 187-ФЗ, КИИ, ФСТЭК, 420-ФЗ, 421-ФЗ, персональные данные, КИИ, ИБ, защита информации.
- **Экономия AI**: если после фильтра релевантных 0 — Anthropic не зовётся (≈90% недель тишина). Иначе один запрос с `{title+link}` + `{law_id: title}`.
- **Sanitize RSS**: `validation.SanitizeLine` на Title/Description/Link — закрывает prompt-injection через RSS (FINDING-6).
- `last_checked=now()` обновляется всем активным законам **даже если изменений нет** (иначе не отличить «стабильно» от «шедулер упал»).

### Disposable email + Password blocklist + Rate limit
- **Disposable**: `internal/validation/email.go`, 72k доменов через `go:embed`. Родительские домены матчатся рекурсивно.
- **Пароль**: мин 8 символов, только печатаемые ASCII (кириллица/эмодзи запрещены), минимум цифра ИЛИ спецсимвол. Блок-лист: топ-20 + 16 числовых (111111..999999, 123456, и т.д.). Единое сообщение об ошибке (не подсказывать атакующему).
- **Rate limit**: `internal/handler/auth.go::RegisterIPLimiter`. Fixed-window 3 попытки / 24h / IP. Не token bucket (он плавный, не подходит для «N per 24h»).
- Синхронизация правил с фронтом через `lib/passwordValidation.ts` (якорь `SYNC-ANCHOR` в обоих файлах). Бэк — авторитетный, фронт — UX-подсказка.

### AdditionalParams (двухэтапная форма)
Поля: `has_remote`, `has_contractors`, `pd_count` (`none/under_1k/1k_100k/over_100k`), `ib_responsible` (`dedicated/combined_it/none`), `regulators[]` (`FSTEK/RKN/FSB/internal`), `has_siem`, `working_hours` (`247/business_hours`), `risk_priorities[]` (`pd/finance/continuity/reputation/trade_secrets`), `nda_duration`, `nda_party`.

Инжектятся в промпты:
- **security_policy** — персональный раздел про ответственного за ИБ (3 варианта по `IBResponsible`), усиленная парольная политика для 187-ФЗ/КИИ (12 симв / 60 дней / блок после 3), сроки уведомления регуляторов (РКН 24ч/72ч, НКЦКИ 3ч для КИИ), журналы (≥3 лет для ПД по 152-ФЗ ст.22, ≥1 года техн.), раздел 6а «Модель нарушителя» (ФСТЭК №117 п.8).
- **risk_matrix** — `buildTechSpecificThreats()` подставляет угрозы по keyword-матчингу на Technologies[] (Windows/AD → ransomware/PtH, 1С → хищение финансов, СКЗИ → компрометация ключей, Docker/K8s → container escape, БД → SQLi, cloud → IAM-утечка). Всегда добавляет 2025-2026 угрозы (supply chain, RaaS, дипфейки, инсайдеры 85%).
- **incident_response_plan** — блок НКЦКИ/187-ФЗ появляется при `187-fz` в compliance или `industry=kii`. С SIEM → Telegram/SMS алерты, без — еженедельный разбор логов.
- **audit_checklist** — формат `| Норма | Штраф/риск |`, раздел «⚠️ Частые замечания регуляторов».
- **director_letter** — ⚠️ LLM запрещено выдумывать цифры/проценты и атрибутировать их реальным источникам (Positive Technologies, BI.ZONE, РКН). Разрешены hedge-формулировки + whitelisted числа (ФЗ-420, 4,2 млн руб как средняя стоимость инцидента для МСБ в РФ).

### Тестирование — ВАЖНО
- **В тестах использовать только DeepSeek** (`DEEPSEEK_API_KEY`). Anthropic API в тестах не использовать — дорого и медленно.
- Unit-тесты провайдеров работают через `httptest.NewServer` без реальных ключей.
- Integration-тесты DeepSeek — build-tag `integration`, skip если нет ключа.
- Integration-тесты Anthropic — build-tag `anthropic_integration` (не захватывается `integration`), только ручная команда владельца. В CI не запускаются.
- Если в любом новом тесте появится прямой вызов Anthropic API → это баг.

### CI/CD
`.github/workflows/ci.yml`: `actions/setup-go@v5 (1.25)` → `go mod download/verify` → `go vet ./...` → `go build ./...` → `go test -race -count=1 -timeout 300s ./...`. Integration-тесты **не запускаются** в CI.

## Архитектура: Frontend (Next.js)

### Стек
- **Next.js 14** App Router, `'use client'` где нужен state/effects
- **Supabase JS v2**
- **Tailwind CSS** — дизайн-система в `DESIGN.md` (см. `projects/secaudit/design.md`)
- **ReactMarkdown** — рендер сгенерированных документов
- Backend API через `NEXT_PUBLIC_API_URL`

### Ключевые паттерны auth
- `getSession()` — localStorage, мгновенно, без lock
- `getUser()` — HTTP, авторитетно, но медленно
- `getAuthToken()` (`lib/auth.ts`) — `getSession()` + retry 100ms
- На polling `/generate` токен получается **ОДИН РАЗ до loop** (`const pollToken`)

### Страницы
| Путь | Описание |
|---|---|
| `/` | Landing + hero + калькулятор + **pricing секция** (#pricing якорь) |
| `/audit` | Калькулятор рисков (lead magnet, без регистрации) |
| `/register` | Регистрация (email confirm required) |
| `/login` | Вход + инлайн «Забыли пароль?» |
| `/reset-password` | Приёмник recovery-ссылки Supabase |
| `/documents` | Дашборд документов (auth) |
| `/generate` | Генерация: 3 флагмана (security_policy/nda/risk_matrix) + «Ещё 3 типа ▾» — снижение выбора → выше конверсия на первый документ |
| `/privacy` | Политика конфиденциальности |

### Ключевые потоки
**Lead Magnet**: `/audit` → sessionStorage `audit_data` → `/register` → email confirm → `/login` → ветвление `sessionStorage ? /generate : /documents` → на `/generate` prefill из sessionStorage, потом удалить, fallback `GET /companies/my`.

**Document Polling**: 500ms interval, `MAX_POLLS=1200` (10 мин), `pollToken` один раз до loop. Прогресс: 5 этапов 0-50s → "big document" mode. Fallback: если polling без `done` → контент из POST response.

**Password Reset**: `/login?forgot` → `resetPasswordForEmail(redirectTo=…/reset-password)` → anti-enumeration сообщение → письмо → `/reset-password` с `onAuthStateChange('PASSWORD_RECOVERY')` + fallback `getSession()` delay 400ms → validate → `updateUser({ password })`. **В Supabase Redirect URLs ОБЯЗАТЕЛЬНО `https://secauditai.ru/**` + `<vercel>.vercel.app/**`** — иначе ссылки не работают.

### TechInput (автокомплит технологий)
11 категорий: ОС, БД, СКЗИ, Антивирус, Контейнеры, ERP/CRM, Разработка, Инфраструктура, Виртуализация, Сеть/Firewall, SIEM/Мониторинг. Свободный ввод через «Добавить <...>». Backspace на пустом — удаляет последний тег. Escape закрывает.

### Аналитика и UTM
- Яндекс.Метрика через `NEXT_PUBLIC_YM_COUNTER_ID` — без ID блок не рендерится (прод не ломается до получения номера).
- Главный CTA на лендинге: `/audit?utm_source=landing&utm_medium=cta&utm_campaign=v1launch`. Вторичный `utm_medium=fstek-alert`. Расширять под каналы через `utm_medium=habr-article-fstek117`, `utm_medium=telegram-sec_it`.

### Юридика на UI
**ФСТЭК НЕ дисквалифицирует директора напрямую** — это суд через КоАП. Все 4 места (CompanyForm, AuditCalculator ×2, page.tsx) используют «**предписание ФСТЭК и повторная проверка**». Если добавляешь новое упоминание санкций ФСТЭК — заменять на эту формулировку.

## Деплой

### Инфраструктура
- **Backend** → Railway (us-west2, Docker)
- **Frontend** → Vercel Hobby
- **БД/Auth** → Supabase Frankfurt (план FREE)

### Порядок деплоя
1. Push в main → Railway собирает Docker автоматически.
2. Скопировать Railway URL.
3. Vercel: задеплоить фронт, `NEXT_PUBLIC_API_URL=<Railway URL>`.
4. Railway env: `ALLOWED_ORIGINS=https://secauditai.ru,https://<vercel>.vercel.app`.
5. Supabase → Redirect URLs: добавить Vercel + secauditai.ru `/**`.
6. Vercel → Domain: `secauditai.ru`, DNS на REG.RU (A @ → `76.76.21.21`, CNAME www → `cname.vercel-dns.com`).
7. Supabase → Site URL: `https://secauditai.ru`.

### Env Railway (backend)
`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` (secret), `ANTHROPIC_API_KEY` (secret), `APP_ENV=production`, `ALLOWED_ORIGINS`, `PORT` **не задавать** (Railway сам).

### Env Vercel (frontend)
`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_YM_COUNTER_ID` (опционально).

## Git автор — НИКОГДА НЕ МЕНЯТЬ
```
git config user.name  "Ipronin99"
git config user.email "goldenpython99@gmail.com"
```
Причина: Vercel привязан к GitHub `ipronin99`. Рабочий email `IgorPr@clearwayintegration.com` → Vercel блокирует деплои. Локальный config имеет приоритет над глобальным.

## Правила ревью перед push
Обязательно пройти цепочку ролей в `/home/igor/GolandProjects/.reviewers/secauditai/`:
- **Senior Dev** — точка входа и выхода (читает весь изменённый код, не diff).
- Может советоваться с любой ролью / gstack-скиллом, возвращать на правки.
- После цикла правок: tester → techlead → security → analyst → designer → owner → techwriter → devops → architect.
- После итогового аппрува: `git push origin main && git push origin develop` (ВСЕГДА в обе ветки).
- **Платёжные функции** (`internal/payment/`, `/pricing`): Security + Architect + Tester обязательны; webhook spoofing/replay/idempotency; quota race conditions; plan check только из БД, не из клиента.

## Юридика / финансы
- **Самозанятый (НПД)**: 4% B2C / 6% B2B, лимит 2.4 млн/год. ИП — при MRR 150-170к.
- ИП оформляется на Дашу через Госуслуги (Игорь не резидент) → доступ к Реестру Минцифры и ЮКассе B2B.
- **Реестр отечественного ПО Минцифры** → v1.5 (бесплатно, льготы, доверие).
- **ГОСТ Р 56939-2024** (процессы разработки) → v2.0.
- **ФСТЭК сертификация СЗИ** → v3.0 только под M&A (2+ млн руб, 16+ мес — нецелесообразно сейчас; SecAudit не СЗИ, а генератор документов).
- **152-ФЗ не нарушается** — форма собирает только корпоративные данные юрлиц.
- **Маркетинг**: «интеллектуальная генерация документов», НЕ упоминать Claude/иностранный ИИ клиентам. Конкурентов НЕ называть по имени (ФЗ-38 «О рекламе»).

## Roadmap коротко
- **v1.0** (сейчас) — MVP на secauditai.ru, ждёт ЮКассу + soft launch.
- **v1.5** — Pricing ✅, Метрика ✅, CI ✅, RSS sanitize ✅, ЮКасса redirect, DOCX/PDF экспорт (pandoc), пакетная генерация 3 доков, ИП Даши, Реестр Минцифры, soft launch 5-10 ИБ-спецов.
- **v2.0** — тариф под аттестацию (инструкции юзера/админа, регламент доступа, акт классификации, журналы), 5-8к/мес, ЦА — интеграторы. Старт ГОСТ Р 56939-2024.
- **v3.0** — M&A-готовность (ГК Солар, Positive Technologies, BI.ZONE). 100+ платящих, MRR 300к+.

## Что НЕ делаем никогда
— Не продаём лично (холодные звонки/встречи запрещены)
— Не нанимаем людей на старте
— Не идём в госсектор напрямую
— Не пишем код вручную (только Claude Code в GoLand)
— Не называем конкурентов по имени в маркетинге
— Не выпускаем сырой продукт — ИБ-сообщество самая требовательная аудитория.
