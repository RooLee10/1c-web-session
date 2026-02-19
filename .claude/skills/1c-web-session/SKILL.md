---
name: 1c-web-session
description: >
  Взаимодействие с 1С:Предприятие в веб-клиенте через Playwright.
  Использовать для ЛЮБОЙ работы в браузере с 1С-базой: запуск/перезапуск/
  завершение сеанса; навигация по разделам интерфейса; создание и
  редактирование справочников и документов; заполнение форм и табличных
  частей; генерация тестовых данных через UI. Содержит паттерны: clipboard
  paste, DLB-кнопки, пикеры, навигационный кеш, обработка диалогов 1С,
  оптимизация browser_run_code. Триггеры: открой/запусти/перезапусти/
  заверши базу; создай/добавь/измени элемент справочника или документа;
  проведи документ; сгенерируй тестовые данные; любое browser_* действие
  в 1С-базе.
---

# Управление сеансом 1С в веб-клиенте

## Запуск базы

Прочитать `CLAUDE.md` проекта (Read tool) и найти URL базы. Формат обычно: `http://localhost/<имя-базы>/ru_RU/`

Использовать `browser_navigate` для перехода по URL. Если сеанс уже запущен — сначала завершить (см. ниже).

## Запуск JS-сценариев по пути

Команда пользователя вида `запусти сценарий <путь-к-файлу>.js` означает: выполнить код этого файла в текущем сеансе 1С через `browser_run_code`.

Формат сценария: JS-выражение функции `async (page) => { ... }` (пример: `.claude/scenarios/*.js`).

Не искать отдельный npm/ps1-раннер как первый шаг. Не запускать файл как обычный Node.js-скрипт.

Алгоритм:
1. Проверить существование файла сценария по указанному пути.
2. Запустить или восстановить сеанс 1С (URL из `CLAUDE.md`, обработать лицензию и форму входа по правилам этого скилла).
3. Выполнить сценарий способом A (по умолчанию). При недоступном clipboard переключиться на способ B.

### Способ A (рекомендуемый по скорости): clipboard loader

Использовать по умолчанию. Это самый быстрый способ для длинных сценариев: в `browser_run_code` передаётся короткий загрузчик, а не весь файл.

1. Скопировать файл сценария в буфер обмена:
```powershell
Get-Content -LiteralPath <путь-к-сценарию> -Raw | Set-Clipboard
```
2. Проверить доступность clipboard из браузера:
```javascript
async (page) => {
  try {
    await page.evaluate(() => navigator.clipboard.readText());
    return { clipboard: 'ok' };
  } catch (e) {
    return { clipboard: 'denied', error: String(e) };
  }
}
```
3. Запустить улучшенный загрузчик сценария (рекомендуется):
```javascript
async (page) => {
  const fail = (stage, e, extra = {}) => {
    const err = e instanceof Error ? e : new Error(String(e));
    return {
      ok: false,
      stage,
      name: err.name || 'Error',
      message: err.message || String(e),
      stack: err.stack || '',
      ...extra
    };
  };

  let src = '';
  try {
    src = await page.evaluate(() => navigator.clipboard.readText());
  } catch (e) {
    return fail('clipboard', e);
  }

  if (!src || !src.includes('async (page)')) {
    return {
      ok: false,
      stage: 'clipboard',
      message: 'Буфер обмена не содержит сценарий формата async (page) => { ... }'
    };
  }

  let scenario;
  try {
    scenario = (0, eval)(`(\n${src}\n)\n//# sourceURL=1c-web-scenario.from-clipboard.js`);
  } catch (e) {
    return fail('compile', e, { sourcePreview: src.slice(0, 300) });
  }

  if (typeof scenario !== 'function') {
    return {
      ok: false,
      stage: 'compile',
      message: 'Сценарий не является функцией',
      valueType: typeof scenario
    };
  }

  try {
    const result = await scenario(page);
    return { ok: true, stage: 'runtime', result };
  } catch (e) {
    return fail('runtime', e);
  }
}
```

Этот лоадер возвращает структурированную обратную связь: `stage` (`clipboard`/`compile`/`runtime`), `message`, `stack`.

### Способ B (fallback): прямой запуск без clipboard

Использовать, если браузер не дал доступ к clipboard (`NotAllowedError`/`clipboard: denied`).

1. Прочитать файл сценария через shell.
2. Передать содержимое сценария напрямую в `browser_run_code` и выполнить его как функцию `async (page) => { ... }`.
3. Если запуск длинного блока падает по лимиту/таймауту — перейти на пошаговое выполнение через `browser_snapshot`, `browser_click`, `browser_press_key`, `browser_type`.

## Завершение и перезапуск сеанса

**Сервис и настройки** → **Файл** → **Выход** → **Завершить работу**

Для перезапуска: завершить сеанс, затем `browser_navigate` на URL базы.

## Диалог лицензии

Если появляется "Не обнаружено свободной лицензии!" — установить все checkbox через JS (обычный клик не работает) и нажать "Выполнить запуск":

```javascript
async (page) => {
  const rows = await page.locator('text=сеанс:').all();
  for (const row of rows) {
    const parent = row.locator('xpath=..');
    await parent.locator('input, [class*="check"], [class*="box"]').first().click();
  }
  return { clicked: rows.length };
}
```

## Форма входа

Если появляется форма входа — сразу выполнить `browser_navigate` с тем же URL. Не нажимать "Войти".

## Модальные диалоги

При навигации может появиться диалог `beforeunload` — обработать через `browser_handle_dialog` с `accept: true`.

Диалог "Данные были изменены" после `Ctrl+Enter` — нажать **"Нет"**:
```javascript
await page.locator('a').filter({ hasText: 'Нет' }).click();
```

**Диалоги подтверждения 1С** (например "Пометить на удаление?") — `locator('text=Да')` и `Ctrl+Enter` не работают. Использовать **`Enter`**:
```javascript
await page.keyboard.press('Delete');
await page.waitForTimeout(200);
await page.keyboard.press('Enter'); // подтверждает диалог "Да"
```

Пометка удаления в 1С не удаляет запись физически — она остаётся в списке. Физическое удаление — через "Администрирование → Удаление помеченных объектов".

## Заполнение полей ввода

**Стандартные методы Playwright (`fill()`, `type()`, `pressSequentially()`) НЕ работают в 1С.** Всегда использовать clipboard paste:

```javascript
await page.evaluate(() => navigator.clipboard.writeText('Значение'));
await page.keyboard.press('Control+V');
```

Для полей с начальным значением (например "0,00") — сначала `Control+A`, потом `Control+V`.

**Поле "Наименование" при создании элемента справочника** — когда форма нового элемента открывается, поле "Наименование" уже в фокусе автоматически. Не нужно его искать по ID (`[id$="_Description"]` ненадёжен). Вводить сразу:
```javascript
// Форма открылась → Наименование уже активно
await page.keyboard.press('Control+A');
await page.evaluate((v) => navigator.clipboard.writeText(v), name);
await page.keyboard.press('Control+V');
```

## Клавиатурные сочетания

| Сочетание | Действие |
|-----------|----------|
| `Ctrl+Enter` | Записать и закрыть / Провести и закрыть |
| `Ctrl+S` | Записать (без закрытия, без проведения) |
| `Escape` | Закрыть диалог ошибки |
| `Tab` | Перейти к следующему полю |
| `F4` | Показать весь список для выбора |
| `Ins` | Добавить строку в табличную часть |
| `Ctrl+Shift+Z` | Закрыть панель ошибок валидации |

**Проведение:** `Ctrl+Enter` проводит документ и формирует движения по регистрам. `Ctrl+S` только записывает — движений не будет.

## Выпадающие списки

**Все кнопки выбора в 1С имеют суффикс `_DLB`** — как для перечислений, так и для справочников. Всегда использовать суффиксный селектор по имени поля:

```javascript
await page.locator('[id$="_ВидАттракциона_DLB"]').click();
await page.waitForTimeout(100);

// После клика открывается dropdown ИЛИ пикер
if (await page.locator('#editDropDown').count() > 0) {
  // Dropdown (перечисление или небольшой справочник)
  await page.locator('#editDropDown').getByText('Экстремальный').click();
} else {
  // Пикер (большой справочник) — нужно подтвердить выбор
  await page.getByTitle('Выбрать значение (Ctrl+Enter)').waitFor();
  await page.getByText('Экстремальный').click();
  await page.getByTitle('Выбрать значение (Ctrl+Enter)').click();
}
```

**Почему не `getByTitle('Выбрать из списка')`:** при нескольких открытых формах несколько элементов с одним title дадут `strict mode violation`.

**Справочники в ячейке табличной части** — когда ячейка активна, `F4` открывает пикер. Искать через поле поиска (без `browser_snapshot`):
```javascript
await page.keyboard.press('F4');
await page.waitForTimeout(300);
await page.getByTitle('Выбрать значение (Ctrl+Enter)').waitFor();
const pickerSearch = page.getByRole('textbox', { name: 'Поиск (Ctrl+F)' }).last();
await pickerSearch.click();
await page.evaluate(() => navigator.clipboard.writeText('Текст поиска'));
await page.keyboard.press('Control+V');
await page.waitForTimeout(600);
await page.locator('.gridBoxText').filter({ hasText: /Текст поиска/ }).first().click({ force: true });
await page.getByTitle('Выбрать значение (Ctrl+Enter)').click();
```

**Элементы пикера** — тип `generic`, не `gridcell`. `getByText()` может вернуть "not visible" — использовать `.gridBoxText` с `force: true` (см. выше). Прерывать на `browser_snapshot` только при неожиданных ошибках.

**Создание объекта прямо из dropdown** — если нужного элемента нет в справочнике:
```javascript
await page.locator('[id$="_Филиал_DLB"]').click();
await page.waitForTimeout(200);
await page.getByTitle('Создать (F8)').click(); // кнопка в dropdown
await page.waitForTimeout(500);
// Форма нового объекта открывается поверх текущей
// После Ctrl+Enter объект автоматически подставляется в поле
```

## Группы в иерархических справочниках

Развернуть/свернуть группу — двойной клик. При создании элемента внутри развёрнутой группы поле "Группа" заполняется автоматически.

## Табличные части

Добавить строку (`Ins`), заполнить через clipboard paste + `Tab` для перехода между полями.

**Заполнение справочного поля в строке (например, Номенклатура):**
```javascript
// Если несколько таблиц на форме — скоупить к нужной командной панели
await page.locator('[id$="_ПозицииПродажиКоманднаяПанель"]').last()
  .getByTitle('Добавить новый элемент (Ins)').click();
await page.keyboard.press('F4');       // открыть пикер
await page.waitForTimeout(500);

// Поиск по тексту в поле пикера (без browser_snapshot!)
const nomSearch = page.getByRole('textbox', { name: 'Поиск (Ctrl+F)' }).last();
await nomSearch.click();
await page.evaluate(() => navigator.clipboard.writeText('Семейный'));
await page.keyboard.press('Control+V');
await page.waitForTimeout(700);
// Кликнуть по первому результату
await page.locator('.gridBoxText').filter({ hasText: /Семейный/ }).first().click({ force: true });
await page.waitForTimeout(100);
await page.getByTitle('Выбрать значение (Ctrl+Enter)').click();

// Tab → пропустить автозаполняемые поля (Цена) → перейти к редактируемым
await page.keyboard.press('Tab');      // → Цена (автозаполняется)
await page.keyboard.press('Tab');      // → Количество
await page.keyboard.press('Control+A');
await page.evaluate(() => navigator.clipboard.writeText('2'));
await page.keyboard.press('Control+V');
```

**Поиск по тексту работает и в пикере справочника Клиента** (вместо browser_snapshot):
```javascript
await page.locator('[id$="_Клиент_DLB"]').last().click();
await page.waitForTimeout(200);
await page.getByTitle('Показать весь список для выбора (F4)').click();
await page.waitForTimeout(400);
const pickerSearch = page.getByRole('textbox', { name: 'Поиск (Ctrl+F)' }).last();
await pickerSearch.click();
await page.evaluate(() => navigator.clipboard.writeText('Петрова'));
await page.keyboard.press('Control+V');
await page.waitForTimeout(600);
await page.locator('.gridBoxText').filter({ hasText: 'Петрова Мария' }).first().click({ force: true });
await page.getByTitle('Выбрать значение (Ctrl+Enter)').click();
```

## Нумерация форм

Каждая форма получает инкрементальный ID (`form26`, `form34`...). **Не хардкодить номер формы.** Использовать суффиксные селекторы:
```javascript
// Плохо:  page.locator('[id="form40_ВидНоменклатуры_DLB"]')
// Хорошо: page.locator('[id$="_ВидНоменклатуры_DLB"]')
```

## Несколько открытых панелей

**Начальная страница (форма рабочего стола) всегда присутствует в DOM** (form0) и создаёт конфликты для общих селекторов — её поля перекрываются с полями в открытых формах. Например: поле `Клиент` на начальной странице конфликтует с `_Клиент_DLB` любого документа.

**Навигация между разделами накапливает вкладки.** Каждый переход в новый раздел меню открывает новую вкладку списка — старые остаются в DOM. Итого несколько `_ФормаКоманднаяПанель` → strict mode violation. **Перед каждым переходом в новый раздел закрывать все лишние вкладки** (см. `.openedClose` ниже), либо использовать `.last()`.

Правила:
```javascript
// Если только одна открытая форма — однозначно
await page.locator('[id$="_ФормаКоманднаяПанель"]')
  .getByTitle('Создать новый элемент списка (Ins)').click();

// Если возможны несколько открытых списков — использовать .last()
await page.locator('[id$="_ФормаКоманднаяПанель"]').last()
  .getByTitle('Создать новый элемент списка (Ins)').click();

// Поля с конфликтом (Клиент, ПозицииПродажи) — использовать .last()
await page.locator('[id$="_Клиент_DLB"]').last().click();
await page.locator('[id$="_ПозицииПродажиКоманднаяПанель"]').last()
  .getByTitle('Добавить новый элемент (Ins)').click();
```

**Закрытие лишних вкладок перед работой** — класс `.openedClose` — кнопки закрытия горизонтальной панели вкладок (начальная страница не закрывается):
```javascript
while (await page.locator('.openedClose').count() > 0) {
  await page.locator('.openedClose').first().click();
  await page.waitForTimeout(300);
  // Диалог "Данные изменены?" — нажать Нет
  if (await page.locator('a').filter({ hasText: 'Нет' }).count() > 0)
    await page.locator('a').filter({ hasText: 'Нет' }).click();
}
```

## Навигация

Кеш навигации — `.claude/1c-nav.json` в корне проекта. ID разделов и пунктов меню специфичны для каждой базы — не хардкодить, брать из кеша.

**Алгоритм перед навигацией:**
1. Прочитать `.claude/1c-nav.json` (Read tool) — найти раздел и пункт по имени, взять их `id`
2. Если файл не существует или пункт не найден — прочитать `references/scan-nav.js`, запустить через `browser_run_code`, записать результат в `.claude/1c-nav.json` (Write tool), затем навигировать

**Навигация по ID из кеша:**
```javascript
// sectionId, itemId — из .claude/1c-nav.json
await page.locator(sectionId).click();
await page.locator(itemId).waitFor({ state: 'visible', timeout: 2000 });
await page.locator(itemId).click();
```

Пункты меню видны только пока меню открыто — `waitFor({ state: 'visible' })` обязателен. `getByText()` с текстами разделов даёт `strict mode violation` при дублях.

## Известные проблемы

**Скрытые обязательные поля БСП** — справочники с контактной информацией могут иметь скрытые обязательные поля (например "Телефон"). Ошибка блокирует форму через `modalSurface` — ни Escape, ни клик не работают. Единственный выход — перезагрузить страницу (URL базы берётся из `CLAUDE.md`):
```javascript
await page.goto('<URL базы>');
```

**Панель ошибок валидации** — при проведении документа с незаполненными обязательными полями открывается панель ошибок (текст вида "Поле "Филиал" не заполнено"). Панель блокирует клики на DLB-кнопки и другие элементы через `<div class="surface">`.

Обнаружить: `await page.locator('[title="Закрыть (Ctrl+Shift+Z)"]').filter({ visible: true }).count() > 0`

Закрыть: `await page.keyboard.press('Control+Shift+Z');`

## Оптимизация

**Цель: максимально длинная цепочка в одном `browser_run_code`.** Несколько объектов разных типов, навигация между ними — всё в одном вызове. Чем длиннее цепочка, тем быстрее выполнение.

**Для сценариев из файла (`.claude/scenarios/*.js`) по умолчанию использовать clipboard loader** (см. раздел "Запуск JS-сценариев по пути") — это обычно быстрее, чем передавать весь файл в параметре `browser_run_code`.

**При сбое** (`browser_run_code` упал или вернул ошибку) — не продолжать тот же подход. Переключиться на пошаговые инструменты (`browser_snapshot` → `browser_click` и т.д.), найти причину, затем вернуться к длинной цепочке.

Подробности, паттерны, антипаттерны: см. [references/optimization.md](references/optimization.md).
