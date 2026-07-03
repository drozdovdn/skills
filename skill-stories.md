# Skill: генерация Storybook-документации компонента (А.Дизайн-система)

Рецепт для сборки страницы документации компонента в стиле **Storybook autodocs**,
как страница `Input.dc.html`. Формат вывода — один файл `ComponentName.dc.html` (Design Component).

Пиши всё инлайн-стилями. Никаких внешних CSS-классов, кроме `@font-face`/`@keyframes`/reset в `<helmet>`.

---

## 1. Токены бренда

Цвета (использовать как литералы в inline-стилях):

| Роль | HEX |
|---|---|
| Акцент (бренд) | `#EF3124` |
| Акцент-фон (tint) | `#FCEBEA` |
| Успех | `#1E9E6A` / фон `#E7F5EE` / рамка `#C9E9D8` |
| Предупреждение | `#B54708` / фон `#FDF0E6` |
| Текст основной | `#1A1A18` |
| Текст вторичный | `#57574F` |
| Текст приглушённый | `#8A8A82` / `#9A9A93` |
| Фон страницы | `#FFFFFF` |
| Фон-подложка (панели) | `#FAFAF8` |
| Рамка | `#E6E6E1` / hairline `#ECECE7` / `#F1F1EC` |
| Код (тёмный блок) | фон `#111114`, текст `#C4C4CC` |

Синтаксис-подсветка (на тёмном): keyword `#BB9AF7`, tag/component `#F7768E`, attr `#7AA2F7`,
string `#9ECE6A`, punctuation `#89DDFF`, comment/muted `#565F89`.

Шрифты: UI — **Onest** (400–800), код/технический — **JetBrains Mono**.
Подключать в `<helmet>` через Google Fonts.

Радиусы: канвас/панели `12px`, контролы `8px`, поля превью `10–13px`. Отступы секций: `margin-top:52–56px`.

---

## 2. Анатомия страницы (строго в этом порядке)

1. **Storybook top toolbar** — сегмент `Canvas / Docs` (активна Docs) + иконки zoom/reset/share. `position:sticky;top:0`.
2. **Центр-колонка** `max-width:1000px;margin:0 auto;padding:44px 24px 120px`.
3. **Breadcrumb** `Категория / Component` (моно, приглушённый).
4. **Заголовок**: `<h1>` (40px, 800) + бейдж статуса (`Stable`) + версия (`v2.4.0`).
5. **Описание** — 1 абзац, `max-width:680px`, 17px.
6. **Import-строка** — плашка `import { Component } from '@astrahovanie/ui'` + кнопка «Копировать».
7. **Primary story** — канвас с точечной сеткой + тулбар зума (top-right), компонент в дефолтном состоянии по центру.
8. **Show code** — раскрывающийся блок под канвасом: вкладки React/HTML + копирование, тёмный код с подсветкой.
9. **Addon panel** — вкладки `Controls / Actions / Interactions / Accessibility` (переключаются по клику).
10. **Stories** — `<h2>` + список именованных стори (`<h3>` PascalCase + описание + канвас + свой Show code).
11. **Гайдлайны (Do / Don't)** — две карточки: зелёная «Так делать», красная «Так не надо».
12. **Связанные компоненты** — сетка карточек-ссылок.
13. **Footer** — подпись «Storybook autodocs».

---

## 3. Панель аддонов — что кладём в каждую вкладку

- **Controls** = ArgsTable. Колонки `Name · Description · Default · Control`.
  В Name: имя (моно) + тип строкой ниже (`#C0392B`) + бейдж `Required` для обязательных.
  В Control — живые виджеты: text input, сегмент (sm/md/lg), select-look, toggle.
- **Actions** — лог событий: `onChange` / `onFocus` / `onBlur` с аргументами и временем. Кнопка Clear.
- **Interactions** — тулбар play (⏮◀▶⏭) + шаги play-функции с зелёными галочками, статус `Pass`.
  Шаги пишем как `canvas.getByLabelText(...)`, `userEvent.type(...)`, `expect(...).toBeVisible()`.
- **Accessibility** — сводка `Violations · 0` / `Passes · N` / `Incomplete · 1` + список axe-правил
  (`label`, `color-contrast`, `aria-valid-attr`, `tabindex`) с pass/warn иконками.
  **Это и есть способ отслеживать доступность — отдельного текстового раздела не делаем.**

---

## 4. Логика (класс Component)

Состояние держит: активную вкладку кода (`codeTab`), раскрытость primary-кода (`codeOpen`),
активную вкладку аддон-панели (`panel`), и map раскрытости кода по каждой story (`open`).

```js
class Component extends DCLogic {
  state = { codeTab:'react', copied:'', codeOpen:true, panel:'controls',
    open:{ /* по одному ключу на story */ } };
  copy(text,id){ try{navigator.clipboard&&navigator.clipboard.writeText(text)}catch(e){}
    this.setState({copied:id}); setTimeout(()=>this.setState({copied:''}),1500); }
  togStory(k){ return ()=>this.setState({open:Object.assign({},this.state.open,{[k]:!this.state.open[k]})}); }
  renderVals(){
    // вернуть: display/arrow/label/toggle для primary и каждой story,
    // стили+сеттеры+display для 4 вкладок панели, copy-хэндлеры.
  }
}
```

Правила DC, которые нельзя нарушать:
- Переключение видимости (`display`, `rotate`) — через runtime-холы `{{ }}`, значения считаем в `renderVals()`.
- Статичный текст и цвета — литералами в разметке, НЕ через холы (иначе не отрисуется при стриминге).
- Никакого `React.createElement` для layout — только шаблон.

---

## 5. Готовые сниппеты

**Канвас story (сетка + Show code бар):**
```html
<div style="border:1px solid #E6E6E1;border-radius:11px;overflow:hidden">
  <div style="padding:40px;display:flex;align-items:center;justify-content:center;
       background-image:radial-gradient(#EDEDE7 1px,transparent 1px);background-size:20px 20px">
    <!-- компонент -->
  </div>
  <div onClick="{{ xxToggle }}" style="display:flex;align-items:center;justify-content:space-between;
       padding:8px 14px;background:#FAFAF8;border-top:1px solid #EDEDE8;cursor:pointer">
    <span style="...">▸ {{ xxLabel }}</span>
    <span style="font-family:'JetBrains Mono';font-size:11.5px;color:#B4B4AD">StoryName</span>
  </div>
  <div style="display:{{ xxDisplay }};background:#111114;padding:16px 20px">
    <pre style="...color:#C4C4CC">&lt;Component ... /&gt;</pre>
  </div>
</div>
```

**Строка ArgsTable:**
```html
<div style="display:grid;grid-template-columns:1.3fr 1.6fr .8fr 1.5fr;padding:15px 20px;
     border-top:1px solid #F1F1EC;align-items:center">
  <span><span mono bold>propName</span><br><span mono #C0392B>type</span> <badge Required?></span>
  <span>Описание</span>
  <span mono muted>default</span>
  <контрол>
</div>
```

**Do / Don't карточка** — заголовок с иконкой (✓ зелёный / ✕ красный), пример компонента, `<ul>` из 3 пунктов.

---

## 6. Чек-лист для нового компонента

- [ ] Заменил токены? (бренд — красный, фон белый — не трогаем)
- [ ] Primary story = самое типичное состояние
- [ ] ArgsTable покрывает все публичные props, обязательные помечены `Required`
- [ ] 4–6 именованных stories, PascalCase, у каждой свой Show code
- [ ] Accessibility-вкладка заполнена реальными axe-правилами для этого компонента
- [ ] Interactions отражают ключевой сценарий (ввод/клик/выбор)
- [ ] Do / Don't — по 3 конкретных пункта
- [ ] Связанные компоненты ведут на соседей по группе
- [ ] Проверил рендер, нет ошибок в консоли

---

## 7. Как использовать

1. Скопируй `Input.dc.html` → `NewComponent.dc.html` как основу.
2. Пройди по разделам 2–3 и заменяй контент под новый компонент.
3. Обнови ключи `state.open` и хэндлеры в `renderVals()` под свои stories.
4. Пройди чек-лист (раздел 6).
