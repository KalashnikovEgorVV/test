# Agentic Content OS: чтиво в самолёт

**Издание:** финальное для чтения (Mastra + Assistant UI primitives)  
**Стек:** Mastra · AI SDK stream · Assistant UI primitives · MCP → CMS  
**Продукт:** чат, который создаёт контент на сайте; images; много users и личных threads  
**Метафора:** Agentic Content OS / Assistant OS — не виджет чата, а OS-слой координации агента  
**Задача текста:** карта мышления и дисциплина роста, не туториал «как поставить пакет»

### Как читать
- Мало времени → **§3–16** (OS), потом **§39** (север) и **§59** (эталонный run)  
- Хочется с начала → §1 → дальше по блокам из **§60**  
- После посадки → **§40** (Must/Should/Later) и **§54** (RFC-шаблоны) как рабочие артефакты  

### Оглавление крупных блоков
1. §1–2 — ставка и primitives  
2. **§3–16 — Agent / Content OS**  
3. §17–21 — контуры, объекты, bridge, UI  
4. §22–32 — users, MCP, images, context, reliability, standards  
5. §33–40 — рост, зрелость, MVP boundary  
6. §41–48 — CMS, content UX, security, incidents, handoff  
7. §49–61 — tests, ops, templates, эталон, карта чтения, закрытие  

---

## 1. С чего начать голову

Большинство людей думают: «возьму LLM, прикручу чат, потом tools».  
В 2026 уже видно, что так получается демо, а не продукт.

Нормальная единица мышления — не сообщение и не модель, а **Agentic Content OS** (операционная система агента для контента).

Это не бренд и не отдельный продукт «скачать OS».  
Это способ собрать слой координации, без которого чат остаётся игрушкой:

- человек что-то воспринимает и выражает (UI);
- система собирает контекст (что модель вообще видит);
- агент решает и действует (orchestration + tools);
- мир меняется (сайт/CMS);
- всё это наблюдаемо, ограничено политиками и готово расти.

В индустрии это же называют agent platform / agentic OS: память, tools, orchestration, governance, observability.  
У вас узкая и сильная специализация — **OS для создания и изменения контента на сайте** через чат + MCP.

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

## 3. Это Assistant / Agent OS, а не «чат-виджет»

Вы строите не фичу чата.  
Вы собираете **OS-слой для агента** внутри продукта.

В индустрии это называют по-разному: agentic OS, agent platform, harness-as-OS, AIOS-kernel.  
Суть одна: между «моделью» и «реальным миром» нужен coordination layer — иначе получается демо.

### Чем OS отличается от чатбота

| Чатбот | Agent / Assistant OS |
|--------|----------------------|
| Отвечает текстом | Управляет работой end-to-end |
| История = всё состояние | Разговор / run / entity разделены |
| Tools как плагины «на потом» | Tools/MCP — syscalls в мир |
| UI = пузырьки | UI = cockpit: preview, diff, approval, status |
| Упал — «попробуй ещё» | Cancel, budgets, audit, rollback |
| Растёт промптами | Растёт контрактами и каталогами |
| Безопасность «в промпте» | Политика на границе действия |

### Два оттенка одного слова

- **Assistant OS** — человек в центре: чат, approvals, handoff, понятный пульт.  
- **Agent OS** — исполнение: runs, tools, memory, isolation, governance.  

Вам нужны оба. Практичный ярлык продукта:

**Agentic Content OS** — OS-слой, где ассистент создаёт и меняет контент сайта.

Это не desktop OS и не «ещё один framework».  
Это тонкий coordination layer, который вы собираете из Mastra + Assistant UI + MCP + CMS.

---

## 4. Карта OS: kernel, cockpit, world

Полезнее думать не «у нас Linux для агентов», а так:

```
┌────────────────────────────────────────────┐
│ COCKPIT (Assistant UI primitives)          │
│ chat · preview · diff · approvals · status │
├────────────────────────────────────────────┤
│ SUPERVISOR / KERNEL-ISH LAYER              │
│ identity · scheduler-ish · memory · tools  │
│ policy · budgets · audit · stream bridge   │
├────────────────────────────────────────────┤
│ WORLD                                      │
│ CMS / site · object storage · LLM APIs     │
└────────────────────────────────────────────┘
```

### Cockpit
То, что видит человек.  
У вас — Assistant UI primitives + preview/entity panel.  
Если cockpit умеет только пузырьки — это ещё не OS для content.

### Supervisor / kernel-ish
Не буквальное ядро Linux. Ближе к **process supervisor + policy middleware**:

- кто запросил;  
- какой run стартовал;  
- какие tools можно вызвать;  
- сколько шагов/денег осталось;  
- нужен ли approval;  
- что записать в audit.

У вас это склейка: auth + Mastra + `@mastra/ai-sdk` + policy на MCP tools.

### World
CMS, файлы, модели.  
Агент не «живёт» в CMS — он ходит туда через syscalls (MCP).

### Почему скептики правы — и всё равно метафора полезна

Метафора ломается, если обещать «настоящий kernel с планировщиком как у Linux».  
Метафора работает, если использовать её как **чеклист обязанностей**:

есть ли identity? память? I/O стандарт? изоляция? governance? audit? cockpit?

Если да — вы строите OS-слой.  
Если нет — у вас обёртка над `chat.completions`.

---

## 5. Модули OS и ваш стек

Классический agent-OS словарь (AIOS и наследники) хорошо мапится на вас:

| OS-модуль | Вопрос | У вас |
|-----------|--------|-------|
| Access / Identity | кто действует? | auth → userId/tenant |
| Scheduler | что исполняется сейчас? | run lifecycle; per-thread serialize |
| Context manager | что в окне модели? | context contract |
| Memory manager | что помним между turns? | Mastra resource/thread memory |
| Storage manager | где бинарное и долгие артефакты? | attachments store, CMS revisions |
| Tool manager | как вызываем мир? | MCP catalog + idempotency + UI registry |
| Stream/IPC | как cockpit слышит kernel? | AI SDK UI message stream |
| Governance | что нельзя без спроса? | approvals, budgets, validate/publish gates |
| Telemetry | как понять, что случилось? | traces, evals, analytics |

Правило сборки OS: **модуль можно заменить, контракт модуля — нет**.  
Можно сменить CMS. Нельзя позволить каждому tool изобретать свой auth и свой audit.

---

## 6. Process model: Run как процесс

В обычной OS единица исполнения — process/thread.  
В Agent OS единица исполнения — **run**.

```
User
 └─ Thread (session/conversation)
     └─ Run (process)
         ├─ model steps
         ├─ tool/MCP syscalls
         ├─ approvals (signals)
         └─ terminal status
```

### Что даёт process-мышление

- cancel = kill/signal, а не «закрыть вкладку и забыть»;  
- budgets = quotas на процесс;  
- audit = process accounting;  
- idempotency keys привязаны к run/step;  
- UI показывает process state, не только текст.

### Состояния процесса (run)

`created → running → waiting_approval → running → succeeding/failing → terminal`  
плюс `cancelled`, `budget_exceeded`, `degraded`.

Cockpit обязан уметь эти состояния показать.  
Иначе OS есть «внутри», но для человека снова магический чат.

### Изоляция

- разные users — разные address spaces (tenancy);  
- разные threads — параллельные процессы ок;  
- один thread — не два writer-run’а сразу (lock), как файл, открытый на запись.

---

## 7. Syscalls: MCP tools как системные вызовы

LLM не должен «просто дергать CMS».  
Он делает **syscall** в OS-слой, а OS решает, исполнять ли.

### Аналогия

| OS | У вас |
|----|-------|
| `open/read/write` | `getEntity` / `updateDraft` |
| `ioctl` спец. операции | `publishRevision`, `validateEntity` |
| driver | MCP server + CMS adapter |
| syscall table | Tool Catalog |
| errno | typed error taxonomy |
| strace | tool trace в run |

### Свойства нормального syscall

1. стабильное имя и schema;  
2. проверка прав;  
3. валидация аргументов;  
4. side-effect class;  
5. idempotency для writes;  
6. timeout;  
7. структурированный результат/ошибка;  
8. audit record;  
9. UI-проекция (tool card).

Если tool = «функция в промпте без этого» — это не syscall, а дырка в ядре.

### Dual catalog = syscall table + man pages для UI

- backend catalog — что можно вызвать;  
- UI registry — как человек это видит;  
- рассинхрон = неизвестный syscall → fallback, не silent fail.

### Новые возможности = новые syscalls, не новый kernel

Хотите поддерживать товары, лендинги, локали — добавляете entity types + tools.  
Не переписываете chat runtime.

---

## 8. Память OS: RAM, disk, page faults

Самая сильная часть OS-метафоры для агентов — память.

| OS | Agent OS | У вас |
|----|----------|-------|
| RAM | context window | то, что собрали в turn |
| Disk | long-term memory / CMS / files | Mastra memory, entity revisions, attachments |
| Page in | retrieval / recall / getEntity / analyze image | select в context contract |
| Page out | summarize / drop raw / keep reference | compression policy |
| Pinning | always-on policy/tools/system | stable prefix для cache |
| Memory leak | тащить весь сайт и все images каждый turn | cost + quality death |

### Практические следствия

- не хранить binary в «RAM-истории» навсегда — только references;  
- после vision — description + fileId на disk, original не в каждом prompt;  
- entity HTML не обязан быть всегда в окне — `getEntity` = page fault;  
- working memory = маленькие pinned facts о user/entity;  
- observational/summary = compaction, когда thread вырос.

Context engineering в этой рамке — не «красивый промпт», а **memory manager OS**.

---

## 9. Права и кольца защиты

Промпт «пожалуйста, не публикуй без спроса» — не security.  
В OS права проверяет ядро.

### Кольца (упрощённо)

| Ring | Кто | Можно |
|------|-----|-------|
| 0 Policy | authz/approval/budget code | разрешить/запретить syscall |
| 1 Supervisor | Mastra orchestration | планировать steps, звать tools |
| 2 Tools | MCP adapters | менять мир в рамках grant |
| 3 Model | LLM | предлагать actions, не исполнять напрямую |
| User | человек | intent, approval, handoff |

Модель всегда выше/снаружи опасных операций: она *просит*, OS *исполняет*.

### Grant на действие

На publish/write полезен envelope:

```
principal + tenant
action
entity
revision
risk level
approval state
budget remaining
```

Нет envelope — нет write.

### Почему это критично для Content OS

Агент с широким CMS token = root.  
Root-агент в проде без publish gate — инцидент, который только ждёт prompt injection из содержимого страницы.

---

## 10. Scheduler, fairness, noisy neighbor

Даже без BullMQ у OS есть планирование:

- кто сейчас занимает model/tool capacity;  
- не дать одному user съесть всех;  
- не запустить второй write-run в тот же thread;  
- остановить no-progress loops.

### Минимальный scheduler для вас

1. per-thread mutex на active run;  
2. per-user concurrency cap;  
3. per-run maxSteps/cost;  
4. global soft limit на LLM/MCP;  
5. явные статусы queue/wait только если реально нужна очередь.

Очередь jobs — опция для фона.  
Scheduler-дисциплина — must для интерактива.

---

## 11. Cockpit: Assistant UI как панель оператора

В зрелых agent OS спорят не только про kernel, но про **cockpit**.  
Без кабины оператора OS бесполезна людям.

### Что cockpit обязан уметь в Content OS

- видеть process/run state;  
- видеть syscalls (tool cards);  
- approve/reject опасные;  
- открыть world state (preview/entity);  
- понять diff before/after;  
- остановить run;  
- найти trace id, когда «сломалось».

### Почему primitives здесь идеальны

Готовый generic Thread оптимизирован под chat SaaS.  
Вам нужен **operator cockpit for content** — значит композиция своя, runtime чужой не мешает.

Cockpit ≠ CMS admin.  
Cockpit = место, где человек и агент вместе ведут изменение мира.

---

## 12. IPC: stream как шина сообщений

В OS процессы общаются через IPC.  
У вас cockpit ↔ supervisor общаются через **UI message stream**.

Это значит:

- parts = message types шины;  
- unknown part = forward-compatible IPC;  
- versioning stream contract = ABI;  
- ломать ABI без версии = segfault продукту.

Поэтому «кастомим UI, не протокол» — не эстетика, а kernel hygiene.

---

## 13. Drivers и world model

MCP server + CMS adapter ≈ device driver.  
Entity types ≈ device classes.

### Хороший driver

- прячет внутренности CMS;  
- говорит на доменном языке (`createDraft`, не «POST /api/v3/...`);  
- отдаёт typed errors;  
- поддерживает conflict/revision;  
- не требует от модели знать SQL/API мусор.

### World model

OS должна знать объекты мира:

`EntityType · EntityId · Revision · Locale · Asset`

Без world model syscalls превращаются в кашу «создай что-нибудь где-нибудь».

---

## 14. Где метафора OS помогает — и где вредит

### Помогает

- разделить обязанности модулей;  
- не путать chat history и content truth;  
- требовать policy на границе write;  
- растить через catalog/syscalls;  
- объяснить команде, почему нужны runId/audit/budgets.

### Вредит

- если начать писать «настоящий kernel» год вместо продукта;  
- если искать 1-в-1 аналоги Linux везде;  
- если забыть, что LLM недетерминирован (это не CPU);  
- если cockpit отложить «на потом».

Честная формула 2026:

**Берём словарь OS, реализуем supervisor + cockpit + governed syscalls, не претендуем на микроядро.**

---

## 15. Как растить Agentic Content OS без переписки

| Хочу | Добавляю в OS | Не трогаю |
|------|----------------|-----------|
| Новый тип контента | entity type + tools + cards | stream runtime |
| Новый шаг publish | gate + approval policy | chat primitives core |
| Картинки как assets | storage + asset syscall | message protocol shape |
| Лучше UX | cockpit composition | MCP contracts |
| Другая CMS | новый driver | tool names/domain verbs |
| Другая модель | model router | syscall/policy layer |

Kernel-ish layer толстеет редко.  
Syscall table и cockpit — часто.

Это и есть OS-strategy vs chatbot-strategy.

---

## 16. Мини-манифест Content OS

1. Человек задаёт intent в cockpit.  
2. Supervisor создаёт run (process).  
3. Memory/context manager собирает working set.  
4. Model предлагает syscalls, не исполняет сама.  
5. Policy решают: allow / deny / ask human.  
6. MCP drivers меняют CMS world.  
7. Verify читает world обратно.  
8. Audit/telemetry записывают историю.  
9. Evals учат OS не повторять ошибки.  
10. Новые возможности = новые syscalls + cards, не новый чат.

Если манифест выполняется — у вас Assistant/Agent OS.  
Если нет — вернитесь к модулю, который проседает.

---

## 17. Семь контуров системы

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

## 18. Главные объекты домена

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

## 19. Инварианты (конституция)

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

## 20. Как устроен мост UI ↔ агент

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

## 21. UI на primitives: дисциплина сборки

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

## 22. Много пользователей и много чатов

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

## 23. Очереди: где да, где нет

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

## 24. MCP и контент на сайте

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

## 25. Картинки: multimodal first-class

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

## 26. Context engineering важнее «красивого промпта»

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

## 27. Reliability: агент = distributed system

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

## 28. Human-in-the-loop

Канонический паттерн:

1. tool требует approval;  
2. run **pause**;  
3. UI показывает approve/reject;  
4. решение пишется в тот же run state;  
5. resume того же run, не новый user turn.

Approvals — это paused runs, не новые диалоги.  
Для publish/destructive — default осторожный.

---

## 29. Память

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

## 30. Observability и evals

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

## 31. Стандарты вокруг, которые стоит знать

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

## 32. Multi-agent: не торопиться

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

## 33. Это всё будет расти

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

## 34. Что заложить заранее (дёшево сейчас, дорого потом)

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

## 35. Пакет документов, которого хватает команде

Не wiki на 200 страниц. Шесть living docs:

1. Identity & access  
2. Stream/Parts contract  
3. Tool/MCP catalog (+ UI mapping)  
4. Context contract  
5. Memory model  
6. Run/Governance & evals  

Плюс короткие ADR, когда меняете шов.

---

## 36. Definition of Done для новой возможности

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

## 37. Антипаттерны, которые встречаются у всех

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

## 38. Модель зрелости

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

## 39. Одна страница «севера»

Если останется в голове совсем мало, пусть останется это:

### Что строим
Не чат-виджет.  
**Agentic Content OS** = cockpit + supervisor + governed syscalls + world (CMS).

### OS одной строкой
Model предлагает → policy решает → MCP меняет мир → cockpit показывает → audit помнит.

### Стек как модули OS
Mastra = supervisor/brain  
MCP tools = syscalls  
Assistant UI primitives = cockpit  
AI SDK stream = IPC bus  
CMS = world  
Auth/approvals/budgets = access manager  

### Правда
Чат хранит разговор.  
Сайт хранит контент.  
Run = process, хранит факт действия.

### Рост
Новые syscalls + cards, не новый kernel.  
Каталоги и контракты вместо особых случаев.

### UX loop
Perceive → Orient → Propose → Approve? → Act → Verify → Remember

### Формула
**Cockpit вместо пузырьков.  
Syscalls вместо «просто tools».  
Policy на границе write, не в промпте.  
Context = RAM, CMS/files = disk.  
Run = process.  
События / catalogs / evals — ABI вашей OS.**

---

## 40. Growth-ready MVP boundary: Must / Should / Later

Это практическая граница: что должно быть в фундаменте **до** волны роста, что можно чуть позже, что сознательно later.

### Must (без этого рост станет переписью)

| Область | Must |
|---------|------|
| Identity | auth → trusted `userId`; никогда из body/model |
| Threads | внешнее хранение, list/create/switch, ownership check |
| Stream | parts-first UI; text + tool + error parts |
| Tools | catalog + UI registry + fallback |
| MCP content | read entity + create/update **draft**; verify/read-back |
| Images | upload → object storage → reference в message |
| Runs | `runId` в логах/traces; cancel хотя бы на клиентском stop |
| Isolation | serialize sends per thread |
| Truth | CMS = content truth; chat = conversation truth |
| Docs | stream contract + tool catalog хотя бы в черновике |

### Should (очень скоро, иначе операционный ад)

| Область | Should |
|---------|--------|
| Publish | отдельный tool + approval |
| Context contract | что видит модель на write-turn |
| Idempotency | на все write MCP tools |
| Budgets | maxSteps / cost cap / image limits |
| Attachments policy | retention, MIME, size, signed URLs |
| Evals | 10–20 golden cases на critical flows |
| Preview panel | рядом с чатом, не только текст «готово» |
| Typed errors | единая taxonomy UI ↔ agent ↔ MCP |
| Trace correlation | UI request → run → tool calls |

### Later (когда появится боль, не раньше)

- BullMQ на тяжёлые pipelines  
- multi-agent / handoffs  
- AG-UI / A2UI как протоколы  
- resource-wide observational memory everywhere  
- shared rooms / multi-user threads  
- full resumable streams с Redis buffer  
- сложный generative UI constructor  
- online evals на 100% трафика  

### Правило границы

Если фича требует сломать Must — она не «feature», а **platform change**.  
Отдельный ADR, отдельный eval pack, отдельное окно миграции.

---

## 41. Entity / CMS domain model

Пока entity не названа, MCP — это «магические руки».  
Назовите мир сайта.

### Базовые объекты контента

| Объект | Пример | Зачем |
|--------|--------|-------|
| **EntityType** | `page`, `post`, `block`, `product` | разные схемы и tools |
| **EntityId** | стабильный id в CMS | на что ссылается чат и preview |
| **Revision** | draft / published / archived | draft≠publish материально |
| **Locale** | `ru`, `en` | не смешивать языки в одном write без intent |
| **Slug / Path** | URL на сайте | verify и deep link из чата |
| **Field/Block** | title, hero, body sections | granular updates вместо rewrite all |
| **Asset** | image/file в медиатеке | отделён от attachment в чате |

### Связь с чатом

В thread metadata / working memory полезно держать:

```
currentEntity: { type, id, locale, revisionId? }
lastProposedRevision?
lastAppliedRevision?
```

Чат не хранит HTML страницы целиком как source of truth.  
Чат хранит **указатели** и разговор о них.

### Команды к миру (через MCP)

Минимальный набор глаголов:

- `getEntity`  
- `listEntities` (осторожно с объёмом)  
- `createDraft`  
- `updateDraft`  
- `getRevisionDiff`  
- `publishRevision`  
- `attachAsset`  
- `validateEntity` (quality gates)

Лучше узкие глаголы, чем один `doEverythingWithSite`.

### Preview sync

Preview читает **CMS/draft revision**, не invent из последнего assistant message.  
После apply → invalidate preview.  
Если apply failed → preview не врёт, что «уже на сайте».

### Конфликты

Когда пользователь (или другой процесс) изменил entity между read и write:

- optimistic locking / revision token;  
- MCP возвращает `conflict`;  
- UI показывает «сущность изменилась, перечитай»;  
- агент не затирает молча.

Это must для растущего продукта, даже если сначала редкость.

---

## 42. Content-agent UX patterns

Кастомные primitives имеют смысл, только если UX отражает content loop, а не generic chat.

### Рекомендуемый layout

```
┌──────────────┬────────────────────┬────────────────────┐
│ Thread list  │ Chat (primitives)  │ Preview / Entity   │
│              │ + tool cards       │ + diff / status    │
└──────────────┴────────────────────┴────────────────────┘
```

На мобиле: chat основной, preview — sheet/drawer по статусу apply/publish.

### Обязательные UX-состояния контент-run

1. **Idle** — можно писать  
2. **Grounding** — читает entity («смотрю текущую страницу»)  
3. **Proposing** — стримит план/draft  
4. **Awaiting approval** — карточка publish/update  
5. **Applying** — MCP write in progress  
6. **Verifying** — read-back  
7. **Succeeded** — ссылка на entity + что изменилось  
8. **Failed** — typed error + retry/safe next step  
9. **Cancelled**

### Diff как first-class

Не «я всё обновил», а:

- какие fields/blocks changed;  
- before/after или summary diff;  
- link open preview;  
- явно draft vs published.

### Tool cards, которые стоит иметь

- `EntityCard` — type/id/title/status  
- `DiffCard` — изменения  
- `ApprovalCard` — publish/destructive  
- `AssetCard` — uploaded/attached image  
- `ValidationCard` — SEO/a11y/brand issues  
- `ErrorCard` — taxonomy + trace id  

### Composer rules для content UX

- while applying/publishing → block conflicting sends  
- allow cancel when safe  
- if awaiting approval → primary actions Approve/Reject, не «просто ещё текст»  
- after success → suggested follow-ups: «улучшить hero», «добавить FAQ», «опубликовать»

### Пустые и краёвые экраны

- нет current entity → предложить выбрать/создать  
- нет permission → честно сказать  
- preview unavailable → не блокировать чат полностью  
- unknown tool → fallback card, не пустота  

---

## 43. Edit / regenerate / branch при side effects

Обычный chatbot предполагает: regenerate = просто другой текст.  
У вас regenerate может означать **ещё один write в CMS**. Это другая физика.

### Операции UI и их опасность

| UI action | Безопасно, если | Опасно, если |
|-----------|-----------------|--------------|
| Edit user message + resend | только propose, без auto-write | автоматически повторит MCP writes |
| Regenerate assistant | пересоберёт текст proposal | заново createDraft/publish |
| Branch / fork thread | новый thread от snapshot | непонятно, какая revision «главная» |
| Retry tool | idempotent write | создаст дубли entity |
| Undo | есть reverse/compensate | publish уже ушёл в prod |

### Политика по умолчанию

1. **Regenerate** перегенерирует proposal/diff, не publish.  
2. Повторный apply — только явный intent пользователя или явная approval.  
3. Retry write использует тот же idempotency key / same revision target.  
4. Edit earlier user message после apply → предупреждение: «в CMS уже есть изменения».  
5. Branching conversation не ветвит CMS автоматически; CMS остаётся linear revisions.

### Модель «proposal vs committed»

В run/result различайте:

- `proposedRevision` — ещё не применено / или применено только в draft;  
- `committedRevision` — записано;  
- `publishedRevision` — живо на сайте.

UI и агент говорят этими словами.  
Тогда regenerate не путает людей и систему.

### Практическое правило

Любая кнопка, которая переигрывает историю сообщений, должна отвечать на вопрос:

**«Будут ли side effects?»**  
Если да — нужен confirm и idempotency strategy.  
Если нет — можно быть лёгкой.

---

## 44. Security для пишущего агента

Пишущий агент — это привилегированный пользователь с LLM в контуре. Угрозы другие.

### Главные угрозы

1. **Prompt injection** через содержимое страницы, комментарии, OCR/картинки, tool results.  
2. **Confused deputy** — агент с широким service account обходит user RBAC.  
3. **Data exfiltration** — «прочитай всё и вставь в публичный пост».  
4. **Destructive overwrite** — затёрли prod.  
5. **Attachment attacks** — SVG/HTML/polyglot files.  
6. **Tool poisoning** — плохие descriptions/schemas у MCP tools.

### Базовые контроли

- least privilege MCP credentials per user/tenant;  
- authz внутри каждого write tool;  
- draft default, publish через approval;  
- allowlist entity types/paths, которые агент может трогать;  
- sanitize/quarantine untrusted content before it becomes «instructions»;  
- treat tool output as data, не как system voice;  
- schema validate MCP args и results;  
- cap response sizes from tools;  
- никогда не класть secrets в context;  
- audit: who/when/tool/entity/before-after hash.

### Injection-aware content loop

Когда агент читает entity:

1. помечать retrieved content как untrusted;  
2. не выполнять найденные в контенте «инструкции»;  
3. policy: менять можно только в рамках user intent текущего run;  
4. publish gate может включать safety checks.

### Images

- MIME sniff server-side;  
- no scriptable formats in public path без политики;  
- если image = reference, не публиковать автоматически;  
- если image = asset, отдельный upload tool с правами.

### Threat model one-pager (стоит иметь)

Короткий doc:

- assets to protect (prod content, credentials, PII);  
- actors (user, attacker via content, malicious file);  
- controls;  
- residual risks.

---

## 45. Cost & model routing

Рост usage убивает продукт тише, чем баги.

### Разные работы — разные модели

| Работа | Модель |
|--------|--------|
| classify intent / route | дешёвая быстрая |
| extract structured brief | средняя |
| long-form write | сильная |
| vision по референсу | vision-capable, точечно |
| judge/eval | отдельная, не всегда top-tier |
| summarize memory | дешёвая |

Не гоняйте frontier model на каждый «да/нет» и каждый tool router.

### Бюджеты

- per run: max steps, max tool calls, max writes, max tokens, max images;  
- per user/day: soft/hard caps;  
- per tenant: noisy-neighbor protection;  
- per tool: timeout + rate limit.

UI должен уметь сказать: «остановлено по лимиту», не просто «ошибка».

### Context cost

Дороже всего часто не output, а:

- огромные tool results;  
- повторная отправка images;  
- раздутый tool catalog;  
- сырой HTML entity в каждом turn.

Лечится context contract’ом и compression policy.

### Product levers

- для cheap users — сильнее лимиты / меньше auto-tools;  
- для pro — выше priority и бюджет;  
- cache стабильного system+tools prefix;  
- не ретраить дорогие writes автоматически без ключа и лимита attempts.

---

## 46. Quality gates before publish

Publish — не кнопка, а ворота.

### Типы gates

| Gate | Примеры |
|------|---------|
| Structural | required fields, locale, slug unique |
| Editorial | tone, banned claims, placeholder text left |
| SEO | title length, meta, headings |
| A11y | alt у images, heading order |
| Safety | policy/PII/secrets leaked into content |
| Brand | voice, forbidden phrases |
| Link health | broken internal links (по возможности) |

### Как встроить

Лучше как tools/checks:

- `validateEntity(revisionId)` → structured issues;  
- агент чинит или просит пользователя;  
- `publishRevision` отказывается при hard-fail;  
- soft-fail можно override с approval и reason.

### UX

ValidationCard:

- errors vs warnings;  
- «fix automatically» vs «publish anyway»;  
- не сыпать 50 замечаний без группировки.

### Правило

Чем легче создать контент, тем жёстче publish gate.  
Иначе AI ускоряет производство мусора.

---

## 47. Incident & rollback playbook

Если агент пишет на сайт, инциденты будут. Готовьтесь текстом, не импровизацией.

### Классы инцидентов

1. Wrong draft content  
2. Wrong publish to prod  
3. Duplicate entities  
4. Partial write / corrupted revision  
5. Permission bug (user увидел чужое)  
6. Cost runaway loop  
7. MCP/CMS outage mid-apply  

### Для каждого класса заранее

- как детектить (alert / user report / audit);  
- blast radius;  
- immediate mitigation (unpublish / revert revision / disable tool / kill switch);  
- user messaging;  
- forensics (runId, tool args, revision ids);  
- follow-up eval case.

### Kill switches

Иметь тумблеры:

- disable publish globally;  
- disable writes for tenant;  
- disable specific MCP tool;  
- force read-only mode for agent;  
- reduce maxSteps globally.

Лучше тупой kill switch, чем изящный труп.

### Rollback model

Идеально: CMS revisions позволяют revert.  
Тогда agent publish = создать revision, rollback = publish previous.  
Если CMS без revisions — сначала ограничить агента draft-only, пока это не появится.

### Postmortem минимум

- timeline;  
- which invariant failed;  
- which catalog/contract gap;  
- new eval + new gate;  
- owner.

---

## 48. Human editor handoff

Не shared chat на много людей, а **передача работы человеку**.

### Когда нужно

- approval слишком сложный;  
- юридический/брендовый review;  
- агент не уверен;  
- publish высокого риска;  
- пользователь просит «отдай редактору».

### Модель

Агент создаёт пакет передачи:

- entity + revision;  
- summary of changes;  
- open questions;  
- risk level;  
- links to preview/diff;  
- conversation pointers (threadId/runId), не обязательно весь dump.

Человек работает в CMS/admin.  
Статус возвращается в thread: `approved | rejected | edited_manually`.

### UX

- статус «на ревью»;  
- кто assignee (если есть);  
- агент в это время не публикует сам;  
- после возврата — verify факта в CMS, не верь словам.

Это дешевле и реалистичнее, чем строить multi-user collaborative agent thread.

---

## 49. Testing strategy: не только «пощупать глазами»

### Пирамида для агентного продукта

1. **Contract tests** — schemas tools/parts, fixture streams.  
2. **MCP mocks** — fake CMS с revision/conflict/errors.  
3. **Agent evals** — golden trajectories (offline).  
4. **UI part tests** — renderers для tool states/fallback.  
5. **Smoke e2e** — один critical path на staging.  
6. **Online monitors** — sample prod traces.

### Что мокать обязательно

- createDraft success/fail/conflict  
- publish approval required  
- validateEntity hard/soft fail  
- timeout/dependency down  
- image upload fail  

### Golden scenarios (минимум)

1. Создать draft страницы из брифа  
2. Обновить существующую entity  
3. Референс-image → proposal без publish  
4. Publish с approval accept/reject  
5. Conflict на update  
6. Unauthorized entity  
7. Cancel mid-run  
8. Unknown tool fallback  
9. Regenerate после draft apply (не должен плодить дубли)  
10. Validation blocks publish  

### Правило CI

PR меняет prompt/tool/schema/renderer → гоняет relevant suite.  
Не «все evals мира», а затронутый контур + критичный smoke.

---

## 50. Retention, удаление, частные данные

Рост = накопление мусора и рисков.

### Что хранится

- threads/messages  
- attachments  
- traces/logs  
- drafts/revisions в CMS  
- memory summaries  

У каждого — своя политика TTL и delete path.

### Практичные defaults

| Данные | Идея |
|--------|------|
| Attachments | TTL или delete with thread |
| Raw traces | короче, чем business records |
| Drafts | по политике продукта |
| Published content | lifecycle CMS, не чата |
| Memory | expire/summarize, не infinite raw |

### User deletion

«Удалить аккаунт/данные» должно понимать:

- удалить threads + attachments;  
- решить, что с generated content на сайте (оставить/анонимизировать/удалить — продуктовый выбор);  
- подчистить memory ids;  
- сохранить то, что нельзя стереть по закону (с минимизацией).

### Правило

Chat delete ≠ automatic site unpublish, пока это не явное продуктовое поведение.

---

## 51. Upgrade playbook: ai / mastra / assistant-ui

Самый хрупкий контур — версии моста.

### Принципы

- обновлять связкой, не по одному пакету в пятницу вечером;  
- читать changelog AI SDK parts/stream первым;  
- иметь smoke matrix до/после;  
- pin в lockfile осознанно;  
- staging с реальными tool cards и одним MCP write path.

### Smoke matrix

1. text stream  
2. tool call + result render  
3. image upload + send  
4. draft apply + preview refresh  
5. approval flow  
6. cancel  
7. reload thread history  
8. unknown tool fallback  
9. error part  
10. multi-thread switch mid-work  

### Если сломалось

Сначала предполагайте **протокол/адаптер**, не «React баг».  
Откат версии моста часто дешевле локальных хаков в UI.

### Календарь

Раз в регулярный интервал — window обновлений.  
Не обновлять runtime mid-feature-freeze без причины.

---

## 52. Product analytics: что считать успехом

Технические метрики нужны. Но продукт измеряется иначе.

### North-star кандидаты

- % сессий, где создан/обновлён draft  
- % drafts, дошедших до publish  
- time-to-first-useful-draft  
- publish success rate after approval  
- revert/unpublish rate (качество)  
- user retry rate (трение)  
- approval reject rate (калибровка осторожности)

### Health metrics

- tool error rate by tool  
- fallback hits (catalog drift)  
- cost per successful publish  
- abandoned runs  
- attachment fail rate  
- conflict rate on writes  

### Qualitative loop

Раз в неделю смотреть 20 traces:

- где человек злится;  
- где агент лишний раз пишет;  
- где preview врёт;  
- где approval бессмысленный или наоборот нужен.

Каждый finding → eval или UX fix.

---

## 53. Когда покидать стек (exit criteria)

Дисциплина включает знание, когда *не* фанатеть.

### Оставайтесь на Mastra + Assistant UI primitives, пока

- хватает AI SDK bridge;  
- custom UI закрывается parts/tool registry;  
- один/несколько agents + MCP ок;  
- команда успевает держать catalogs.

### Пересматривайте шов, если

| Боль | Куда смотреть |
|------|----------------|
| Нужен глубокий shared app state agent↔UI | AG-UI state / CopilotKit-like patterns |
| Сильный multi-agent graph с durable checkpoints | более graph-native orchestration |
| UI целиком generative surfaces | A2UI-like catalog protocol |
| Bridge ломается каждый upgrade | thinner anti-corruption layer / freeze versions |
| Chat runtime мешает продукту | ExternalStore и свой state жёстче |

### Правило выхода

Меняют **один шов**, не всё сразу.  
Сначала protocol/state, потом UI, потом orchestration — по фактической боли.

---

## 54. RFC-шаблоны

Ниже — заготовки. Копируйте в repo как `docs/rfc/`.

### A. Tool Catalog Card

```text
Name:
Version:
Description (for model):
When to use / when not:
Input schema:
Output schema:
Side effect class: read | write | destructive | external
Idempotent: yes/no
Idempotency key strategy:
Approval: auto | always | conditional(rules)
Timeout:
Retry:
Compensate:
Authz requirements:
Entity types affected:
UI renderer:
Fallback:
Errors:
Eval cases:
Owner:
Status: draft | active | deprecated
```

### B. Stream/Part Contract note

```text
Part type:
Version:
Direction: client→server | server→client | both
Payload schema:
Required UI states:
Unknown-part fallback:
Breaking change policy:
Examples:
```

### C. Context Contract slot

```text
Slot name:
Purpose:
Source:
Trust level: trusted | user | untrusted_retrieved
Budget tokens:
Inclusion rules:
Exclusion rules:
Compression strategy:
Cacheability:
Owner:
```

### D. Entity Action

```text
Action: createDraft | updateDraft | publish | ...
EntityType:
Preconditions:
Conflict strategy:
Verify method:
Approval required:
Analytics event:
Incident class if fails:
```

### E. Eval case

```text
ID:
Intent:
Setup (entity/fixtures):
User input:
Attachments?:
Expected tools (optional):
Forbidden tools/actions:
Expected end state (CMS + UI):
Grader: deterministic | llm-judge | human
Severity if fails: block | warn
```

---

## 55. Failure UX catalog

Сделайте библиотеку состояний — иначе каждый экран импровизирует.

| Failure | User-facing | System behavior |
|---------|-------------|-----------------|
| validation | что исправить | no retry loop |
| authz | нет доступа | no leak existence if needed |
| not_found | entity missing | offer create/select |
| conflict | кто-то изменил | re-ground |
| rate_limited | подождите | backoff |
| dependency_down | сайт/MCP недоступен | read-only degrade |
| budget_exceeded | лимит | stop run |
| safety_blocked | нельзя | escalate/handoff |
| upload_failed | повторить файл | keep composer text |
| unknown_tool | частичный результат | fallback card + trace |
| cancelled | остановлено | persist partial safely |

Каждый тип — один ErrorCard pattern + один agent-facing error code.

---

## 56. Degraded modes

Нормальные системы умеют работать «хуже», а не только «никак».

| Mode | Когда | Поведение |
|------|-------|-----------|
| read-only agent | CMS write outage | только анализ/proposal |
| text-only | vision provider down | просить описание картинки словами |
| no-publish | risk event | drafts only |
| limited-tools | overload | subset tools |
| cheap-model | budget pressure | simpler responses, fewer steps |
| human-only publish | safety | agent готовит, человек публикует |

UI явно показывает mode badge.  
Скрытая деградация хуже честной.

---

## 57. Team rituals at scale

Когда вырастете хотя бы до нескольких людей:

### Weekly

- 20–50 trace review  
- catalog drift check (tool without UI / UI without tool)  
- cost anomalies  

### Per PR

- contract touched?  
- evals updated?  
- smoke relevant?  

### Monthly

- dependency upgrade window  
- threat model skim  
- retention policy check  
- kill switch drill (хотя бы tabletop)  

### Ownership

- Agent owner  
- UI owner  
- Contract owner  
- Eval/ops owner  

Можно совмещать роли, нельзя оставлять «ничейным» publish path.

---

## 58. Roadmap narrative (как рассказывать рост)

Вместо хаотичного backlog — сюжет:

1. **Stabilize conversation platform** (threads, parts, images, ownership)  
2. **Make writes safe** (draft/verify/idempotency/approvals)  
3. **Make UX product-native** (preview, diff, content states)  
4. **Make quality measurable** (gates, evals, analytics)  
5. **Make ops boring** (incidents, kill switches, upgrades)  
6. **Then expand surface** (больше entity types/tools/locales)

Каждая новая контент-фича проверяется: «на каком акте этого сюжета мы?»  
Если ещё не закончили акт 2, не открывайте акт 6.

---

## 59. Сквозной сценарий «идеального» run

Соберите в голове один эталон — и сверяйте с ним дизайн.

1. User открывает thread, выбирает/создаёт entity context.  
2. Прикладывает референс-image + текст брифа.  
3. Agent grounding: `getEntity` + vision reference (не publish asset).  
4. Proposal + DiffCard + Preview draft plan.  
5. User уточняет.  
6. Agent `updateDraft` / `createDraft` с idempotency.  
7. `validateEntity` → warnings fixed.  
8. User просит publish → ApprovalCard.  
9. `publishRevision` → verify → Preview shows published.  
10. Trace хранит весь путь; analytics отмечает success; eval покрывает сценарий.

Если ваш текущий путь сильно короче — ок для H1.  
Но эталон показывает, куда сходятся швы.

---

## 60. Карта чтения (обновлённая)

### Быстрые маршруты

- **Agent / Content OS целиком → 3–16**, потом 39, 59  
- Понять контуры/инварианты → 17–19  
- Figma/UI / cockpit → 11, 21, 25, 42, 55  
- Backend/syscalls/MCP → 7, 13, 24, 27, 41  
- Память/context как RAM/disk → 8, 26, 29  
- Рост OS без переписки → 15, 33–35, 40, 58  
- Операционка → 47, 51, 56, 57  
- Качество → 30, 46, 49, 52  
- Шаблоны → 54  

### Если читать в самолёте блоками по 20–30 минут

1. **3–16 (вся тема Agent/Content OS)** ← начните отсюда, если летите ради этой идеи  
2. 1–2 + 17–21 (фундамент стека и UI)  
3. 22–26 (users, MCP, images, context)  
4. 27–32 (reliability, HITL, standards)  
5. 33–40 (рост и MVP boundary)  
6. 41–48 (CMS, UX, security, incidents)  
7. 49–61 (tests, ops, templates, эталон, закрытие)

---

## 61. Закрытие

На этом издание можно считать завершённым: дальше не «ещё глава», а практика в продукте.

Ваша ставка разумная: не прыгать на новый framework каждый месяц, а растить **Mastra + Assistant UI primitives** как тонкий **Agentic Content OS** с богатой продуктной поверхностью.

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
- entity в CMS — правда контента;  
- неизвестное деградирует честно;  
- инциденты заранее имеют ручки;  
- рост идёт через Must/Should/Later, а не через хаос.

Север короткий:

**Это Content OS, не чат.  
Cockpit + supervisor + syscalls + world.  
Model предлагает, policy исполняет.  
CMS — disk мира, context — RAM.  
Run — process.  
Растём каталогом syscalls, не переписью ядра.**

Файл готов к офлайн-чтению. Изменения по этому артефакту закончены.

Хорошего полёта.
