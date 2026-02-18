# Оптимизация работы с 1С веб-клиентом

## Содержание

- Таймаут Playwright MCP
- Сравнение подходов к заполнению
- Типы полей выбора (ВАЖНО!)
- Рекомендуемый паттерн
- JS-верификация
- Антипаттерны
- Когда делать browser_snapshot

## Таймаут Playwright MCP

По умолчанию таймаут `@playwright/mcp` — **5000ms**. Настраивается в `.mcp.json`:

```json
{
  "playwright": {
    "command": "npx",
    "args": ["@playwright/mcp@latest", "--timeout", "1000"]
  }
}
```

**Таймаут — это НЕ пауза.** Это максимальное время ожидания элемента. Если элемент найден за 30ms — действие выполнится за 30ms. Уменьшение таймаута ускоряет только обнаружение ошибок.

| Ситуация | Таймаут | Пример |
|----------|---------|--------|
| Лёгкая база, быстрый сервер | 100–200ms | Простые справочники, мало данных |
| Средняя нагрузка | 300–500ms | Документы со сложными формами |
| Тяжёлая база, медленный сервер | 700–1500ms | ERP с большими регистрами |

Оценить скорость отклика на первых действиях и предложить пользователю оптимальное значение.

## Стратегия: максимальная цепочка

**Цель — один `browser_run_code` на всё задание**, а не на один элемент. Несколько объектов разных типов, навигация между разделами, цикл — всё в одном вызове.

**При сбое цепочки** (`browser_run_code` упал или вернул ошибку) — **не повторять тот же подход**:
1. Переключиться на пошаговые инструменты: `browser_snapshot` → `browser_click` / `browser_press_key`
2. `browser_snapshot` при сбое обязателен — даёт картину текущего состояния DOM
3. Найти причину: неверный селектор, неожиданный диалог, изменившийся DOM
4. Исправить паттерн и вернуться к длинной цепочке

## Сравнение подходов к заполнению (реальные данные)

**Подход 1: Пошаговый** — каждое действие отдельным MCP-вызовом
```
browser_click → browser_run_code → browser_press_key → browser_snapshot → ...
```
Результат: **11 вызовов MCP** на элемент. Использовать только при сбое цепочки для диагностики.

**Подход 2: Объединённый** — всё задание в одном `browser_run_code`
```
browser_run_code (клиент + навигация + документ + ещё документ + JS-верификация)
```
Результат: **1 вызов MCP** на всё задание. Рекомендуемый подход.

**Подход 3: Цикл** — несколько однотипных элементов в одном вызове
```
browser_run_code (for ... { создать + заполнить + сохранить })
```
**Работает** при использовании суффиксных селекторов и `getByRole` — они находят актуальную открытую форму независимо от инкрементального ID.
**Не работает** если хардкодить номер формы (`form40_...`) — после первой итерации эта форма закрыта.

## Типы полей выбора (ВАЖНО!)

**Все кнопки выбора в 1С имеют суффикс `_DLB`** — и для перечислений, и для справочников. Разница не в кнопке, а в том, что открывается после клика.

### Универсальный паттерн (работает для обоих типов)

```javascript
await page.locator('[id$="_ВидАттракциона_DLB"]').click();
await page.waitForTimeout(100);

if (await page.locator('#editDropDown').count() > 0) {
  // Dropdown — перечисление или небольшой справочник
  await page.locator('#editDropDown').getByText('Экстремальный').click();
} else {
  // Пикер — большой справочник, нужно подтвердить
  await page.getByTitle('Выбрать значение (Ctrl+Enter)').waitFor();
  await page.getByText('Экстремальный').click();
  await page.getByTitle('Выбрать значение (Ctrl+Enter)').click();
}
```

### Почему не `getByTitle('Выбрать из списка')`

При нескольких открытых формах (например, рабочее место кассира + форма создания аттракциона) этот селектор находит несколько элементов → `strict mode violation`. Суффиксный `[id$="_ИмяПоля_DLB"]` всегда однозначно идентифицирует нужное поле.

### Типичная ошибка

```javascript
// ОШИБКА: getByTitle без скоупинга при нескольких открытых формах
await page.getByTitle('Выбрать из списка').click();
// → strict mode violation: resolved to 2+ elements

// ПРАВИЛЬНО: суффиксный DLB-селектор
await page.locator('[id$="_ВидАттракциона_DLB"]').click();
```

## Рекомендуемый паттерн

Один `browser_run_code` на один элемент с JS-верификацией:

```javascript
async (page) => {
  // 1. Создать элемент
  await page.locator('[id$="_ФормаКоманднаяПанель"]')
    .getByTitle('Создать новый элемент списка (Ins)').click();

  // 2. Ждать форму
  const nameInput = page.getByRole('textbox', { name: 'Наименование:' });
  await nameInput.waitFor({ timeout: 2000 });

  // 3. Заполнить наименование (Ctrl+A для очистки!)
  await page.evaluate(() => navigator.clipboard.writeText('Название'));
  await nameInput.click();
  await page.keyboard.press('Control+A');
  await page.keyboard.press('Control+V');

  // 4. Tab снимает фокус с поля — ОБЯЗАТЕЛЕН перед кликом по DLB
  await page.keyboard.press('Tab');
  await page.waitForTimeout(50);

  // 5. Клик по DLB — dropdown появляется мгновенно, waitFor не нужен
  await page.locator('[id$="_ВидАттракциона_DLB"]').click();
  await page.waitForTimeout(50);

  // 6. Выбрать значение из dropdown
  await page.locator('#editDropDown').getByText('Экстремальный').click();
  await page.waitForTimeout(50);

  // 7. Сохранить
  await page.keyboard.press('Control+Enter');
  await page.waitForTimeout(800);

  // 8. JS-верификация
  const hasModal = await page.locator('.modalSurface').count() > 0;
  const formOpen = await page.getByRole('textbox', { name: 'Наименование:' }).count() > 0;
  // Панель валидации — блокирует клики на DLB, закрывается через Ctrl+Shift+Z
  // ВАЖНО: этот элемент всегда есть в DOM (невидимый), проверять через filter visible
  const hasValidationPanel = await page.locator('[title="Закрыть (Ctrl+Shift+Z)"]').filter({ visible: true }).count() > 0;
  if (hasModal) return { status: 'error', reason: 'модальная ошибка' };
  if (formOpen && hasValidationPanel) return { status: 'error', reason: 'ошибка валидации — Ctrl+Shift+Z' };
  if (formOpen) return { status: 'error', reason: 'форма не закрылась' };
  return { status: 'ok', element: 'Название' };
}
```

Принципы:
- **`Tab` перед DLB-кликом** — снимает фокус с поля ввода, иначе клик может не сработать
- **`waitForTimeout(50)`** — dropdown появляется мгновенно, `waitFor({ state: 'visible' })` избыточен и может мешать
- **`Control+A` перед вставкой** — очищает поле от предыдущего значения
- **Суффиксные селекторы** `[id$="..."]` — не зависят от номера формы
- **JS-верификация внутри** — проверка без отдельного вызова

## Навигация и strict mode

ID разделов (`#themesCell_theme_N`) и пунктов меню (`#cmd_0_3_txt`) специфичны для каждой базы. Брать из кеша `.claude/1c-nav.json`, не хардкодить:

```javascript
// ПРАВИЛЬНО: ID из кеша
const sectionId = '#themesCell_theme_1'; // из 1c-nav.json
const itemId    = '#cmd_0_3_txt';        // из 1c-nav.json
await page.locator(sectionId).click();
await page.locator(itemId).waitFor({ state: 'visible', timeout: 2000 });
await page.locator(itemId).click();

// ОШИБКА: getByText даёт strict mode violation (меню + заголовок страницы)
await page.getByText('Справочники').click(); // → 2+ элемента
```

Кеш создаётся автоматически при первом скане (`references/scan-nav.js`). Если панель уже открыта — навигация не нужна, сразу создавайте элемент.

## JS-верификация

После `Ctrl+Enter` проверить результат прямо в `browser_run_code`:

```javascript
await page.waitForTimeout(100);

const hasModal = await page.locator('.modalSurface').count() > 0;
const formOpen = await page.locator('[id$="_Наименование_i0"]').count() > 0;
// Панель валидации — блокирует клики, закрывается через Ctrl+Shift+Z
// ВАЖНО: этот элемент всегда есть в DOM (невидимый), проверять через filter visible
const hasValidationPanel = await page.locator('[title="Закрыть (Ctrl+Shift+Z)"]').filter({ visible: true }).count() > 0;

if (hasModal) return { status: 'error', reason: 'модальная ошибка' };
if (formOpen && hasValidationPanel) return { status: 'error', reason: 'ошибка валидации — Ctrl+Shift+Z' };
if (formOpen) return { status: 'error', reason: 'форма не закрылась' };
return { status: 'ok' };
```

Результат — ~20 байт: `{ status: 'ok' }`. Ни картинок, ни дерева элементов.

## Антипаттерны

```javascript
// ПЛОХО: waitForTimeout вместо ожидания конкретного элемента
await page.waitForTimeout(300);
await page.locator('[id$="_DLB"]').click(); // элемент уже есть — можно было waitFor

// ХОРОШО: ждать конкретный элемент-индикатор готовности
await page.getByRole('textbox', { name: 'Наименование:' }).waitFor();

// OK: waitForTimeout когда нет элемента-индикатора (форма закрылась, UI-переход)
await page.keyboard.press('Control+Enter');
await page.waitForTimeout(800); // форма закрыта, ждём сервер — waitFor некуда применить

// ПЛОХО: хардкод номера формы
await page.locator('[id="form40_ВидНоменклатуры_DLB"]').click();
// ХОРОШО: суффиксный селектор
await page.locator('[id$="_ВидНоменклатуры_DLB"]').click();

// ПЛОХО: page.screenshot() в run_code — бесполезен без Read (+100K токенов vision)
await page.screenshot({ path: 'verify.png' });
// ХОРОШО: JS-верификация — 0 токенов
const hasError = await page.locator('.modalSurface').count() > 0;
```

## Когда делать `browser_snapshot`

`browser_snapshot` возвращает accessibility tree (~2-10 KB) с `ref` для кликов:

- **Обязательно:** при первом создании элемента нового типа (изучить форму)
- **Обязательно:** при ошибке или неожиданном поведении
- **Не нужно:** при массовом создании однотипных элементов (достаточно JS-верификации)
- **Не нужно:** между заполнениями полей внутри `browser_run_code`
