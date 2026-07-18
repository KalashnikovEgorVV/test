# Агентный продукт на вебе: чтиво в самолёт

**Стек в центре:** Mastra + Assistant UI primitives (+ AI SDK как мост)  
**Продукт в центре:** чат, который создаёт контент на сайте через MCP, с картинками, многими пользователями и личными тредами  
**Задача текста:** не туториал «как поставить пакет», а карта мышления — чтобы растить систему годами, а не переписывать через квартал

---

## 1. С чего начать голову

Большинство людей думают: «возьму LLM, прикручу чат, потом tools».  
В 2026 уже видно, что так получается демо, а не продукт.

Нормальная единица мышления — не сообщение и не модель, а **платформа агента**:

- человек что-то воспринимает и выражает (UI);
- система собирает контекст (что модель вообще видит);
- агент решает и действует (orchestration + tools);
- мир меняется (сайт/CMS);
- всё это наблюдаемо, ограничено политиками и готово расти.

Ваша связка хороша тем, что роли уже почти правильные:

| Роль | Технология |
|------|------------|
| Кастомный UI | Assistant UI primitives |
| Протокол стрима | AI SDK UI message stream |
| Мозг/оркестрация | Mastra |
| Руки в мир | MCP → сайт |
| Память диалога | Mastra `resourceId` / `threadId` |

Отдельного пакета `@assistant-ui/react-mastra` нет и не надо. Мост — AI SDK. Это не костыль, а де-факто стандарт.

---

## 2. Почему primitives, а не готовый Thread

Если нужен «максимально кастомный UI», есть спектр:

1. Полностью свой UI на `useChat` — максимум свободы, максимум велосипедов.
2. **Assistant UI primitives** — свой внешний вид, но готовый runtime чата.
3. Готовый `<Thread />` — быстро, но чужая композиция.

Primitives — правильный компромисс для продукта, который будет жить долго:

- вы контролируете layout, анимации, tool cards, preview сайта;
- не пишете заново streaming, message parts, composer state, thread switching;
- можете вырасти из «чата» в «пульт управления контентом», не сжигая фундамент.

Важно: кастомите **рендер**, не протокол. Протокол — святое.

---

## 3. Семь контуров системы

Когда всё начнёт расти, полезно видеть не фичи, а контуры.

### 1) Identity
Кто действует: user (и позже tenant/org).  
Модель никогда не выбирает идентичность. Её даёт auth.

### 2) Conversation
`user (resource) → threads → messages/parts → runs`  
Чат — журнал разговора, не хранилище сайта.

### 3) Perception
Текст + изображения/файлы.  
Upload → storage → reference в сообщении → (опционально) vision/описание → попадание в context.

### 4) Cognition
System policy, tool schemas, memory, retrieval, current entity.  
Это **context layer** — здесь чаще всего решается, умный агент или дорогой хаос.

### 5) Action
MCP tools: прочитать страницу, создать draft, обновить блок, опубликовать.  
Каждое действие — schema, side-effect class, authz, idempotency, UI-состояние.

### 6) Truth
- CMS/сайт — source of truth контента  
- Chat DB — source of truth разговора  
- Preview — проекция, не истина

### 7) Control
Approvals, budgets, cancel, audit, evals.  
Без этого пилот не становится продуктом.

Если помните только одну вещь: **развивайте контур, а не дыру между контурами**.

---

## 4. Главные объекты домена

Договоритесь о словаре. Иначе через месяц «чат», «задача», «агент» и «пост» будут означать всё сразу.

| Объект | Смысл |
|--------|--------|
| User | principal, владелец |
| Thread | личный разговор |
| Message / Part | единица отображения диалога |
| Attachment | картинка/файл на входе |
| Run | одно исполнение агента |
| ToolCall | попытка действия |
| Entity | страница/пост/блок на сайте |
| Draft / Revision | предлагаемое или применённое изменение |
| Approval | пауза перед опасной записью |
| Trace | наблюдаемость run |

**Messages — для людей. Runs — для системы. Entities — для сайта.**

---

## 5. Инварианты (конституция)

Их лучше нарушать осознанно, почти никогда.

1. UI не пишет в CMS напрямую — только через agent/MCP/policy.  
2. Agent не владеет UI state — только stream/events/tools.  
3. Message ≠ Content Entity.  
4. Run — единица side effects.  
5. Threads изолированы по user; внутри одного thread runs сериализуются.  
6. Binary не живёт вечно в history — живут references + descriptions.  
7. Unknown tool/part всегда имеет fallback.  
8. Authz проверяется на границе действия, не в prompt.  
9. Draft ≠ Publish.  
10. Изменение поведения без контракта или eval — долг.

Смелость оставляйте UX. Скуку — контрактам.  
Именно так система переносит рост.

---

## 6. Как устроен мост UI ↔ агент

```
React (Assistant UI primitives)
  → @assistant-ui/react-ai-sdk (useChatRuntime)
    → HTTP UI Message Stream (AI SDK)
      → Mastra agent (@mastra/ai-sdk)
        → tools / MCP
```

Два рабочих паттерна деплоя:

- **Full-stack:** Mastra внутри Next API route — проще на старте.  
- **Separate server:** Mastra отдельно (`chatRoute`), фронт бьёт в URL — чище при росте команд/нагрузки.

Клиент в обоих случаях один и тот же. Меняется только место мозга.

### Что считать протоколом

Не «текст ответа», а типизированные parts/events:

- lifecycle: start / finish / error / cancel  
- text streaming  
- tool: input → running → result/error  
- approval / interruption  
- image/file parts  
- (позже) state snapshot/delta  

Правило AI SDK, которое надо выжечь в память: рендерить **`message.parts`**, не `content`.

---

## 7. UI на primitives: дисциплина сборки

Кастомный интерфейс — это pipeline:

```
ThreadList
  → Thread
    → Messages
      → Message
        → Parts
          → text | image | tool | reasoning | data | fallback
```

### Правила, которые спасают через полгода

- Один renderer на tool name, не копипаста по экранам.  
- У каждого tool UI одна state machine:  
  `pending → args-streaming → running → success | error | needs-approval`.  
- Composer знает статус run: во время streaming политика send явная.  
- Thread list отделён от message canvas.  
- Превью сайта / entity panel — **рядом** с чатом, не внутри chat runtime.

Хорошая продуктовая композиция для вас:

```
[Список чатов]  [Чат на primitives]  [Превью страницы / сущности]
```

Чат управляет намерением.  
Preview показывает следствие.  
CMS хранит правду.

---

## 8. Много пользователей и много чатов

Вам **не нужны** shared-комнаты на несколько людей в одном треде. Нужны:

- много пользователей;
- у каждого много личных чатов;
- параллельная работа разных людей и разных тредов.

### Модель Mastra

- `resourceId` = пользователь  
- `threadId` = конкретный разговор  

Один user → много threads.  
Memory на resource может помнить предпочтения между чатами; история — per thread.

Важно: Mastra **не делает authz за вас**. В доках прямо сказано: перед recall/query проверяйте, что user имеет доступ к `resourceId`/thread.

### Модель Assistant UI

По умолчанию — один in-memory thread.  
Мультитред — через AssistantCloud / RemoteThreadList / ExternalStoreThreadList.  
Для кастомного продукта обычно своя БД + thread list primitives.

### Конкурентность

| Уровень | Параллелизм |
|---------|-------------|
| Разные users | да |
| Разные threads одного user | да |
| Один и тот же thread | нет без очереди/lock |

Особенно опасно параллелить два send в один thread, когда есть tools: ломается порядок `tool_calls → tool_results`.

Практическое правило:

**Изолируй `user → thread`, сериализуй runs внутри thread, всё остальное пускай параллельно.**

---

## 9. Очереди: где да, где нет

Интерактивный чат **не надо** класть в BullMQ.

- Живой stream токенов → прямой request/SSE.  
- Защита от double-send в thread → Redis lock / disable composer.  
- Rate limits → Redis counters.  
- Тяжёлый фон (reindex, webhooks, длинный publish pipeline) → тогда BullMQ.

У вас чат создаёт контент через MCP — очередь может понадобиться не «потому что чат», а потому что **side effects**. Но это отдельное решение, не фундамент диалога.

Формула:

**Chat stream всегда live.  
Write path — контролируемый.  
Queue — только для хрупкого/долгого фона.**

---

## 10. MCP и контент на сайте

Это делает продукт серьёзным: агент не только говорит, он меняет мир.

Минимальный продуктовый loop:

1. **Brief** — что сделать  
2. **Ground** — прочитать текущую entity  
3. **Propose** — draft/preview  
4. **Approve** — если write/publish  
5. **Apply** — MCP write  
6. **Verify** — read-back и показать результат  
7. **Close** — итог + ссылка на сущность  

Даже без сложных очередей этот loop отделяет «умный ассистент» от «бота, который иногда крушит прод».

### Dual catalog

Два каталога, одна правда:

| Backend Tool Catalog | Frontend Tool UI Catalog |
|----------------------|--------------------------|
| name, schema, effect | renderer, states |
| approval policy | approval card |
| errors | error UX |
| version | fallback |

PR на новый MCP tool без UI renderer = неполный.  
PR на красивую карточку без schema = тоже неполный.

---

## 11. Картинки: multimodal first-class

Раз в агента можно отправлять изображения, это не «бонус», а часть протокола.

Сообщение пользователя:

```
user message
  ├─ text?
  └─ image/attachment parts
```

### Где что живёт

- UI: picker, preview, upload states  
- Storage: object storage, не вечный base64 в history  
- Message history: reference + metadata (+ описание)  
- Model: vision на нужном turn  
- MCP: отдельно решить, это референс или asset для сайта  

Самая важная развилка:

1. image как **референс для рассуждения**;  
2. image как **asset для публикации**.

Это разные intent, разные tools, разные риски.

### Политика роста для images

- лимит на turn;  
- validate MIME/size на сервере;  
- signed URL с TTL;  
- в следующих turns не таскать все originals forever;  
- после анализа можно хранить description + `fileId`;  
- storage path с user/thread namespace.

State machine upload:

`selecting → uploading → ready → sent → failed|expired`

Пока не `ready` — send лучше не отпускать.

---

## 12. Context engineering важнее «красивого промпта»

Prompt engineering спрашивает: «как сформулировать?»  
Context engineering спрашивает: **«что модель видит на этом шаге, в каком виде и зачем?»**

Слоты типичного turn:

1. system / policy  
2. tool definitions  
3. memory  
4. current entity  
5. recent messages  
6. tool results  
7. images/attachments  
8. environment (user, locale, entity id)

Четыре стратегии:

- **Write** — важное во внешний draft/CMS, не только в чат;  
- **Select** — текущая страница, не весь сайт;  
- **Compress** — summary/observational memory вместо бесконечной истории;  
- **Isolate** — не сваливать research/write/publish в один грязный контекст без структуры.

Стоит завести короткий **Context Contract**: бюджеты слотов и правила сборки.  
Меняете состав контекста — считайте, что меняете поведение продукта.

Антипаттерны:

- весь сайт в prompt;  
- 40 tools сразу без фильтра;  
- сырые гигантские tool results;  
- картинки-мегабайты в каждом последующем turn.

---

## 13. Reliability: агент = distributed system

Как только есть MCP writes, вы в мире распределённых эффектов.

Для каждого write-tool полезно знать:

- effect: read / write / destructive / external  
- idempotent?  
- кто генерит idempotency key (**не LLM**)  
- timeout  
- retry policy  
- compensateWith?  
- approval policy  

Ключевые правила:

- ключ = детерминированный (`runId + stepId + tool + inputHash`);  
- retry write без ключа — путь к дублям статей;  
- nested budgets: per-tool, per-run, cost/steps;  
- no-progress breaker: если топчется — стоп;  
- typed errors вместо каши строк.

### Disconnect ≠ Stop

Если когда-нибудь сделаете resumable streams:

- refresh/offline = disconnect, generation может продолжаться;  
- кнопка Stop = отдельный cancel на сервере;  
- у thread появляется `activeStreamId`.

Это не день-один обязательность, но знать стоит заранее.

---

## 14. Human-in-the-loop

Канонический паттерн:

1. tool требует approval;  
2. run **pause**;  
3. UI показывает approve/reject;  
4. решение пишется в тот же run state;  
5. resume того же run, не новый user turn.

Approvals — это paused runs, не новые диалоги.  
Для publish/destructive — default осторожный.

---

## 15. Память

Mastra-модель:

- `resourceId` — пользователь / долгоживущая сущность;  
- `threadId` — разговор.

Практично:

- thread scope — default для истории;  
- resource scope — только когда реально нужна сквозная память;  
- working memory — маленькие structured facts;  
- observational memory — длинные диалоги со сжатием;  
- persist complete messages, не каждый chunk.

И снова: memory API ≠ authorization. Ownership проверяете вы.

---

## 16. Observability и evals

Без этого рост превращает продукт в фольклор («иногда норм, иногда нет»).

### Минимум observability

- `run_id` / `thread_id` / `user_id` сквозь UI → API → Mastra → MCP  
- spans на model/tool/memory  
- tool error rate, steps/run, cost/run, cancel rate  

### Evals как цикл, не как экзамен

Agent Development Lifecycle:

```
prod traces → failed cases → offline evals → ship → online monitors → repeat
```

- Offline: golden set критичных сценариев + CI gate.  
- Online: sample live traces, reference-free checks.  
- Failed prod → новый offline case.

Меняете prompt/tools/model — это изменение поведения, значит нужен eval smoke.

---

## 17. Стандарты вокруг, которые стоит знать

Не обязательно внедрять всё завтра. Стоит понимать карту.

| Слой | Стандарт | Зачем |
|------|----------|-------|
| Agent ↔ UI | AG-UI | события, state, HITL |
| Agent ↔ Tools | MCP | руки в системы |
| Agent ↔ Agent | A2A | позже, если распределённые агенты |
| Generative widgets | A2UI-like catalogs | безопасный UI от агента |
| Ваш текущий bridge | AI SDK stream | уже работает |

Практичная стратегия: **не мигрировать ради моды**, а заимствовать идеи контрактов.

Особенно полезны идеи A2UI даже без внедрения протокола:

- агент шлёт декларативные намерения UI;  
- клиент рендерит только trusted catalog;  
- никакого произвольного HTML от модели.

Ваш tool UI registry — уже зародыш этого мышления.

---

## 18. Multi-agent: не торопиться

Консенсус такой: topology важнее «ещё одной роли».

Сначала один хороший agent + tools/workflows.  
Multi-agent — когда упираетесь в context/роли/latency, а не потому что красиво на слайде.

Если дойдёте:

- orchestrator-worker как default;  
- передавать intent + constraints, не только сырой output;  
- verifier перед side effects;  
- общий budget на весь graph;  
- пользователю не показывать хаос из пяти «личностей», если продукт этого не требует.

---

## 19. Это всё будет расти

Значит проектируем не точку, а траекторию.

### Закон расширяемости

Новое добавляется как плагин в каталог, а не как перепись ядра.

Разрешённый рост:

- новый thread  
- новый part type  
- новый MCP tool + renderer  
- новый approval rule  
- новый eval case  
- новый context slot  

Дорогой рост:

- новый ad-hoc протокол UI↔agent  
- бизнес-логика в React  
- CMS write в обход MCP  
- «особый режим» для картинок/публикации без общего pipeline  

### Ядро медленно, поверхность быстро

Стабильное ядро:

- identity/thread/run  
- stream/parts contract  
- memory ids  
- authz envelope  
- draft≠publish  
- dual catalog  

Быстрая поверхность:

- красивые tool cards  
- preview UX  
- новые content tools  
- tone/brand prompts  
- шаблоны секций  

### Три горизонта

**H1 — Working**  
чат, images, MCP drafts, multi-user threads, custom primitives, basic traces

**H2 — Operable**  
approvals, context contract, catalog sync, eval smoke, budgets, attachment policy

**H3 — Platform**  
много entity types, много MCP tools, team rituals, online evals, degraded modes, фоновые pipelines по необходимости

Не кодируйте H3 в день H1.  
Но имена и швы сразу совместимы с H3.

---

## 20. Что заложить заранее (дёшево сейчас, дорого потом)

- `threadId` / `resourceId` / `runId` везде  
- tool registry + fallback  
- image as reference, не вечный blob в history  
- draft/publish в домене  
- ownership checks  
- typed errors  
- correlation/trace ids  

### Что можно later

- multi-agent mesh  
- полный AG-UI/A2UI  
- очередь на каждый чих  
- shared rooms  
- сверхсложная long-term memory на всё подряд  

---

## 21. Пакет документов, которого хватает команде

Не wiki на 200 страниц. Шесть living docs:

1. Identity & access  
2. Stream/Parts contract  
3. Tool/MCP catalog (+ UI mapping)  
4. Context contract  
5. Memory model  
6. Run/Governance & evals  

Плюс короткие ADR, когда меняете шов.

---

## 22. Definition of Done для новой возможности

Фича «готова», если есть:

- [ ] понятный объект домена (tool/part/entity action)  
- [ ] schema/contract  
- [ ] UI renderer или осознанный fallback  
- [ ] authz  
- [ ] fail/cancel path  
- [ ] trace fields  
- [ ] влияние на memory/context описано  
- [ ] хотя бы 1 happy + 2 failure checks  
- [ ] не сломан stream contract (или поднята версия)

---

## 23. Антипаттерны, которые встречаются у всех

- рендерить только `content`  
- новый tool без UI state machine  
- approval «просто спроси в тексте»  
- memory = весь raw chat навсегда  
- domain state внутри messages  
- tenant/user id из body «на доверии»  
- retry write без idempotency  
- LLM генерирует idempotency UUID  
- секреты и широкие credentials в контексте модели  
- обновлять `ai` / mastra / assistant-ui по одному «на глаз»  
- считать offline-демо доказательством production quality  
- плодить агентов вместо контрактов  

---

## 24. Модель зрелости

```
L1  Chat works
L2  Tools/MCP пишут drafts
L3  Threads + multi-user isolation
L4  Multimodal first-class
L5  Propose → Approve → Act → Verify
L6  Context contract + memory discipline
L7  Governance (authz, budgets, audit)
L8  Evals close the loop
L9  Reliable side effects / degraded modes
L10 Platformized catalogs + team rituals
```

Если чат, MCP-контент и images уже есть, вы не в начале нуля.  
Смысл следующего этапа — **сшить L2–L6 в конституцию**, а не бесконечно расширять поверхность.

---

## 25. Одна страница «севера»

Если останется в голове совсем мало, пусть останется это:

### Стек
Mastra = мозг  
MCP = руки  
Assistant UI primitives = лицо  
AI SDK = нервная система стрима  
CMS = мир  

### Правда
Чат хранит разговор.  
Сайт хранит контент.  
Run хранит факт действия.

### Рост
Каталоги и контракты.  
Не особые случаи и не магия в компонентах.

### UX loop
Perceive → Orient → Propose → Approve? → Act → Verify → Remember

### Формула
**События вместо голого текста.  
Catalog вместо произвольного UI.  
Pause/resume вместо нового turn.  
Evals вместо ощущений.  
State layers вместо каши.  
Стабильные швы вместо героизма.**

---

## 26. Карта чтения, если вернуться позже

- Хочешь понять продукт целиком → разделы 3–5, 25  
- Хочешь UI → 7, 11  
- Хочешь агента и MCP → 10, 12, 13  
- Хочешь масштаб → 8, 19–21  
- Хочешь качество → 16, 22–24  

---

## 27. Закрытие

Ваша ставка разумная: не прыгать на новый framework каждый месяц, а растить **Mastra + Assistant UI primitives** как тонкую платформу с богатой продуктной поверхностью.

Чат будет красивее.  
Tools станет больше.  
Картинки станут обыденностью.  
Контент-типов прибавится.  
Команда вырастет.

Переживёт это не самый хитроумный prompt, а система, в которой:

- объекты названы;  
- швы стабильны;  
- каталоги синхронны;  
- run наблюдаем;  
- draft не путают с publish;  
- неизвестное деградирует честно, а не молча ломается.

Если после посадки захочется продолжить — следующий полезный артефакт не «ещё библиотека», а один page **Growth-ready MVP boundary**: Must / Should / Later под ваш реальный продукт.

Хорошего полёта.
