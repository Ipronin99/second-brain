# SecAudit AI — Дизайн-система
Обновлено: 2026-04-21
Источник: `/home/igor/GolandProjects/secauditai-frontend/DESIGN.md` (source of truth)

## Направление
**Industrial / Utilitarian с редакционной дисциплиной.** Минимум декора, типографика и сетка делают всю работу. Mood — серьёзный, технический, авторитетный: «инструмент регуляторного compliance», не «стартап про wow-эффект». Пользователь за 3 секунды должен понять: «эти ребята знают ФСТЭК, это не игрушка».

### Референсы и чему учимся
- **BI.ZONE** — тёмный фон, белая типографика, сдержанные акценты. Авторитет через сдержанность.
- **Positive Technologies** — нумерованные секции (01/02/03), строгая иерархия, высокий контраст, минимум цвета.
- **Kaspersky** — геометрический sans, много воздуха, один выверенный акцент.

### Чего НИКОГДА не делаем
- Тёмно-фиолетовые фоны (#1a1a2e и родня).
- Розовые/коралловые CTA (#e94560, #FF2D78).
- Эмодзи в продуктовом UI (🛡️🤝📊🚨).
- Градиенты. Скруглённые углы >4px. Декоративные blob-фигуры.
- Центрирование всего подряд. Три колонки фич с иконками в цветных кружочках.
- «Built for X» / «Designed for Y» копирайт.

## Цветовая система

```css
/* Фоны и поверхности */
--bg:           #0B0F14; /* основной фон, глубокий сланец */
--surface:      #12171F; /* навбар, таблицы */
--surface-2:    #1A2029; /* карточки, инпуты */
--border:       #252C36;
--border-hi:    #3A4251;

/* Текст */
--text:         #E8ECF1;
--text-muted:   #8B95A7;
--text-dim:     #5A6576;

/* Brand / CTA — steel blue, trust-blue gosudarstvennyi */
--accent:       #1E6FEB; /* сдвинут в cyan, чтобы на #0B0F14 не читался как фиолет */
--accent-hover: #1558CC;
--accent-soft:  rgba(30, 111, 235, 0.12);

/* Семантика (скупо) */
--success:      #16A34A;
--danger:       #DC2626; /* регуляторное/критическое/destructive */
--danger-soft:  rgba(220, 38, 38, 0.10);
--warning:      #CA8A04; /* денежные суммы, частично */
```

### Правила применения
- Основной экран строится на 5 нейтральных (`bg/surface/surface-2/border/text`). Цвет — акцент, не фон.
- `--accent` — только primary CTA, лого «AI», активные табы/линки. **Не** использовать для бордеров карточек и раскраски мелких иконок.
- **Жёсткое разграничение danger vs warning** (2026-04-17):
  - **danger** — регуляторная опасность / необратимое действие / отказ: FSTEK-алерт, «предписание», «уголовная ответственность», form errors, destructive actions (удалить), статусы «просрочен/отсутствует», крестики в таблице.
  - **warning** — внимание без тревоги: денежные суммы (штрафы, риск в рублях), «частично», «перегенерировать». Деньги ≠ красный — это финансовое внимание, не регуляторное предписание.
  - Почему: на тёмном фоне красного много → он перестаёт читаться как сигнал. Банковский/fintech-язык нашего позиционирования («trust-blue gosudarstvennyi») требует, чтобы красный был редким.
- Контраст: `--danger` #DC2626 на `--bg` ≈ 4.5:1 (AA pass). Исправлено с #B91C1C (3.2:1, fail).

## Типографика

```css
--font-display: 'Manrope', system-ui, sans-serif;
--font-body:    'IBM Plex Sans', system-ui, sans-serif;
--font-mono:    'IBM Plex Mono', ui-monospace, monospace;
```

- **Display/Hero**: Manrope 600/700 — геометрический, сильная кириллица, для h1/h2.
- **Body/UI**: IBM Plex Sans 400/500/600 — технический, редакционный, «мы читаем ГОСТы, а не Medium».
- **Data/Tables/Mono**: IBM Plex Mono 400/500 + `font-variant-numeric: tabular-nums` для сумм (`2 990 ₽`), ID, дат.

### Modular scale (ratio 1.200 от 16px)
| Role | Size | Line height | Weight | Tracking |
|---|---|---|---|---|
| Display XL | 56 / 3.5rem | 1.05 | 700 | -0.02em |
| H1 | 40 / 2.5rem | 1.1 | 700 | -0.01em |
| H2 | 28 / 1.75rem | 1.2 | 600 | -0.01em |
| H3 | 20 / 1.25rem | 1.3 | 600 | 0 |
| Body | 16 / 1rem | 1.55 | 400 | 0 |
| Body small | 14 / 0.875rem | 1.5 | 400 | 0 |
| Caption | 12 / 0.75rem | 1.4 | 500 | 0.02em |
| Mono | 14 / 0.875rem | 1.5 | 400 | 0 (tabular-nums) |

### Правила
- `text-balance` на всех h1/h2.
- Максимум 2 веса на страницу: 400 body + 600/700 заголовки.
- Кириллический текст никогда не `uppercase` без `letter-spacing: 0.05em`.
- Никаких Inter / Roboto / Poppins. Plex + Manrope даёт отличительность.

## Layout & Spacing

- Max container: **1200px**. Gutters: 24px desktop / 16px mobile.
- 12-column implicit grid, большинство секций — 1/2/3 asymmetric.
- Левое выравнивание по дефолту, не центрирование.

Spacing scale (4px база):
```
--s-0:0; --s-1:4; --s-2:8; --s-3:12; --s-4:16;
--s-5:24; --s-6:32; --s-7:48; --s-8:64; --s-9:96;
```
- Секции разделены `--s-8` (64px).
- Hero padding: top 64, bottom 80.
- Карточки `padding: 24px` (`--s-5`).

Radius:
```
--r-sm: 2px;  /* инпуты, ячейки таблиц */
--r-md: 4px;  /* кнопки, карточки — дефолт */
--r-lg: 8px;  /* только для больших hero-панелей */
```
`rounded-xl` (12px) и `rounded-full` на кнопках — **убрано**.

## Компоненты

### Buttons
- **Primary**: `bg:--accent`, text:#FFF, без бордера, `--r-md`, padding 12×20. hover: `--accent-hover`. active: +5% darker. disabled: opacity 0.4.
- **Secondary**: прозрачный, `text:--text`, border `--border`, `--r-md`. hover: `border-color:--border-hi`, `bg:--surface-2`.
- **Ghost**: прозрачный, `text:--text-muted`. hover: `text:--text`, `bg:--surface-2`.
- **Destructive** (только delete): прозрачный, `text:--danger`, border `--danger`. hover: `bg:--danger-soft`.
- Высота **всегда 44px** (touch target). Никаких `rounded-full` chip на primary CTA. Никаких градиентов.

### Cards
- `bg:--surface-2`, border 1px `--border`, `--r-md`, padding 24px, **без shadow**.
- Если кликабельна — hover: `border-color:--border-hi`, `cursor:pointer`.
- Карточки фич — `lucide-react` `stroke-width:1.5`, 20px, цвет `--text-muted`. ShieldCheck, FileSignature, Network, AlertTriangle, ClipboardCheck, FileText.

### Tables (сравнение, dashboard)
- `font-family:var(--font-mono)` для числовых + `tabular-nums`.
- Хедер: `bg:--surface`, `text:--text-muted`, uppercase, `letter-spacing:0.05em`, 12px.
- Строки через `border-bottom:1px solid --border`.
- Галочки в таблице сравнения — SVG check/x из Lucide, цвет `--success`/`--danger`/`--warning`.

### Forms
- Инпуты: `bg:--surface-2`, border `--border`, `--r-sm`, padding 10×12, `font-body`.
- Focus: `border-color:--accent`, outline 2px `--accent-soft`.
- Ошибка: `border-color:--danger`, сообщение ниже `color:--danger`, 13px.
- Label: 13px, `color:--text-muted`, margin-bottom 6px. **Никогда** не placeholder-only labels.

### Navbar
- `bg:--surface` (не `gray-900`). Border-bottom `--border`.
- Лого: Manrope 700, «SecAudit» в `--text`, «AI» в `--accent`.
- Активная ссылка — `--accent`, не bold.
- Email пользователя в навбаре — truncate 20 симв, `text-gray-500` (приглушённый), `title=<полный>`.

### FSTEK-алерт (ключевой элемент доверия)
**Не «баннер продажи», а регуляторное уведомление — выглядит как юридический документ.**
```
bg:transparent
border-left:3px solid --danger
border-top/right/bottom:1px solid --border
padding:24px
radius:--r-md или 0 (не rounded-xl!)
Заголовок:color:--danger, weight:600
Текст:color:--text
CTA внутри:Secondary button (outlined), не Primary
```

## Motion
**Minimal-functional.** Только переходы, помогающие понять состояние.
- transitions: `150ms ease-out` на `background`, `border-color`, `color`.
- `transform` — только для disclosure (chevron на accordion).
- **Никаких** scroll-driven анимаций, parallax, animated numbers, fade-in-on-viewport.
- Loading: тонкий линейный индикатор сверху или `<Skeleton>` с shimmer 1.5s. Spinner — только на кнопках.

## Иконки
- **Lucide** (line, stroke 1.5-2px) — один источник на весь продукт.
- Размер: 20px в UI, 16px в плотных местах (таблицы, breadcrumbs).
- Цвет: `inherit`, обычно `--text-muted`; hover → `--text`.
- **Запрет эмодзи в продуктовом UI**. Эмодзи допустимы только в user-generated content (название документа, если юзер сам ввёл).

## Dark-only
Светлая тема не строится. Фокус — один выверенный тёмный режим. Когда понадобится light mode — тогда и проектируем, не «на всякий случай».

## SAFE vs RISK choices

**SAFE (категорийный baseline — ожидания пользователя)**:
- Тёмный фон — ИБ-продукты почти всегда тёмные.
- Сетка + sans — стандарт B2B-compliance.
- Галочки зелёным, крестики красным — табличный язык категории.

**RISK (где продукт берёт своё лицо)**:
- **Manrope + IBM Plex** вместо Inter/Roboto. Даёт «мы читаем ГОСТы, а не Medium». Цена: +20KB на первой загрузке.
- **Steel blue (#1E6FEB) как единственный акцент** вместо красного/оранжевого как у соседей. BI.ZONE/PT играют на тревожном красном, мы — на доверенном синем (банковский/финтех язык → правильная ассоциация «работаем с регулятором, а не против него»). Компенсация: красный остаётся, но **только в FSTEK-алертах** — тогда он читается как тревога, не декорация.
- **2px border-radius** почти везде вместо 12px bubbly. Индустриальный язык. Компенсация — воздух в layout и теплота в Plex Sans.

## Файлы-источники
- Tailwind config: `tailwind.config.ts`.
- Global CSS: `app/globals.css` (подключение Manrope + IBM Plex, CSS custom properties).
- Компоненты, затронутые системой: `app/layout.tsx`, `app/page.tsx`, `components/Navbar.tsx`, `AuditCalculator.tsx`, `CompanyForm.tsx`, `DocumentCard.tsx`, `ReadinessDashboard.tsx`, `Tooltip.tsx`, `TechInput.tsx`.
- Подробный план замены (что → на что, файл → строка): `DESIGN_PROPOSAL.md`.
