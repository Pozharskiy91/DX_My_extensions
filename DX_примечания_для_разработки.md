# Примечания по разработке DX-расширений (DataExpress)

Файл для контекста в новом чате. Здесь — проверенные паттерны, синтаксис, типовые ошибки и ссылки.
Папки с расширениями: `D:\Github\DX_My_extensions` (мои), `D:\Github\DX_selection_extensions` (отбор, включая kok80-*).

---

## 1. Синтаксис фильтров DX (проверено на практике)

### Поиск «содержит» — ТОЛЬКО оператор `==`
```pascal
Fm.CustomFilter := '{[Наименование]==''текст''}';   // == = "содержит", без учёта регистра
Fm.open;
```
- `==` в CustomFilter = «содержит» (подтверждено тестом 2026-05-30, ImportDataInspection.epas).
- `=` в том же CustomFilter = **точное совпадение** значения (частичный ввод даёт пустой список).
- Несколько полей объединяются ИЛИ через ` | `: `{[Поле1]=='txt' | [Поле2]=='txt'}`.
- Числовые поля: `[Поле]=5` (без кавычек), только если введено число.

### Filter-объект (официальный API) — ТОЧНОЕ совпадение
```pascal
var FF: TFilterField;
Fm.Filter.Clear;
FF := Fm.Filter.AddField('Имя поля');   // имя поля из конструктора
FF.Values.Add('значение');
Fm.Refresh;
```
Wiki: https://wiki.mydataexpress.ru/programmirovanie_v_dx:filtr_formy
Подходит для точного поиска/диапазонов (`'2 .. 5'`, `'10 .. '`); для «содержит» НЕ подходит.
Для объекта (lookup) значение = id записи (число). Цепочки вызовов `Fm.Filter.AddField(...).Values.Add(...)` могут не парситься — присваивать промежуточную переменную.

### КРИТИЧНО: имена полей в фильтрах
- В фильтрах (CustomFilter, Filter.AddField, OpenRecords) поля указываются **по имени поля из конструктора** — то, что возвращает `GetComponentFieldName(component)`.
- Системное имя датасета (`Field.FieldName` = `F12`, `F12A`) в фильтре **не работает** — фильтр молча не применяется (без исключения!).
- Получение имени поля из колонки Grid:
```pascal
// Grid column Field.FieldName = 'F12' или 'F12A' (колонка табличной части)
FN := Grid.Columns.Items[x].Field.FieldName;
FN := StringReplace(FN, 'F', '', [rfReplaceAll]);       // '12A'
LastChar := Copy(FN, Length(FN), 1);                     // убрать последнюю букву A-Z
if (LastChar >= 'A') and (LastChar <= 'Z') then
  FN := Copy(FN, 1, Length(FN) - 1);                     // '12'
id := StrToIntDef(FN, 0);
// найти компонент по ID и взять имя поля из конструктора:
for y := 0 to Fm.ComponentCount - 1 do
  if GetComponentId(Fm.components[y]) = id then Cmp := Fm.components[y];
FN := GetComponentFieldName(Cmp);
```
- Этот же паттерн использует kok80-poisk (см. `GetFormVisibleColumnFieldIDList`, `GetOpenRecordsFilter` в kok80-poisk3.00.epas).

### Запасной механизм фильтрации (если фильтры не сработали)
Клиентский скан записей + фильтр по списку RecId:
```pascal
Fm.DisableControls;
Fm.MoveFirst;
while (not Fm.Eof) do
begin
  if Pos(AnsiUpperCase(txt), AnsiUpperCase(VarToStr(Fm['Имя поля']))) > 0 then
    ids := ids + ' | [RecId]=' + IntToStr(Fm.RecId);
  Fm.MoveNext;
end;
Fm.EnableControls;
Fm.CustomFilter := '{' + ids + '}';   // или '{[RecId]=-1}' для пустого результата
Fm.open;
```
Методы перехода: MoveFirst/MoveLast/MoveNext/MovePrior/MoveBy/MoveTo/GotoRecord/Locate.
Wiki: https://wiki.mydataexpress.ru/programmirovanie_v_dx:perexod_na_zapis
`Locate(FieldNames; Values: array of Variant; [loCaseInsensitive, loPartialKey])` — поиск фрагмента с переходом на запись (не фильтр).

---

## 2. Особенности Pascal-скрипта DX (частые ошибки компиляции)

| Ошибка | Причина / решение |
|---|---|
| `Type mismatch` на `and` | Приоритет: `and` выше `<>`. Писать скобки: `(a <> '') and (StrToIntDef(s, -1) <> -1)` |
| Вызовы по цепочке `a.b().c()` | Может не парситься — присваивать промежуточную переменную |
| `''` внутри строкового литерала | Кавычка удваивается: `'[' + FN + ']==''' + txt + ''''` — проверять каждый литерал (лишний `"` = ошибка компиляции) |
| Опечатки в строковых литералах | Легко внести лишний символ при правке `'='''` → `'==''"'` — компилятор укажет строку |
| `Self` недоступен в процедуре расширения | Только в контексте формы/метода; в процедурах модуля использовать `GetActiveForm` или параметры |
| debug / Debug | Регистр важен; в kok80-poisk — `debug` (маленькая), встроенная `Debug` тоже есть |

Встроенные функции/методы, доступные в скрипте (проверено):
`GetComponentId(comp)`, `GetComponentFieldName(comp)`, `Debug(s)`, `ExceptionToString(ExceptionType, ExceptionParam)`, `EvalExpr(expr, fm)`, `SQLSelect(query)`, `MsgBox`, `MessageDlg`, `StrToIntDef`, `TryStrToFloat/TryStrToInt/TryStrToDate`, `AnsiUpperCase`, `StringReplace`, `Pos`.
У `TdxForm`: `RecordCount`, `RecId`, `RecNo`, `DisableControls/EnableControls`, `MoveFirst/...`, `Filter: TFilterObject`, `CustomFilter`, `Open`, `OpenRecords(filter, nil, true)`, `OpenRecord(0)`, `Grid.Columns`, `ComponentCount/components[]`, `FindComponentByFieldName(name)`, `Refresh`, `Params`, `Eof`.
Классы: `TdxEdit, TdxComboBox, TdxMemo, TdxFile` (текст), `TdxCalcEdit` (float), `TdxCounter, TdxRecordId` (int), `TdxDateEdit, TdxTimeEdit`, `TdxCheckBox` (bool), `TdxLookupComboBox` (объект).
Проверка типа через строку: `Cmp.ClassName in ['TdxEdit','TdxComboBox',...]` (как в kok80-poisk `IsTextClass`).

---

## 3. Паттерн: защита расширений от падений (по опыту kok80-poisk)

Цепочка падений при переключении вкладок: `MainWindow_pages_OnChange → RestoreSearchHistoryAndValue → TEdit_Filter_onChange → OnTimer → ...`.
Правила:
1. Каждая точка, где объект может быть nil (`qg`, `DOMsource`, `DOMField`, `fm.Queries[name]`, `GetActiveForm`, `Grid.Rows[x]`), защищать: `if obj = nil then Exit/Continue;`
2. `fm.queries[...]` и `fm.Queries[...]` брать в `try/except` — исключение, если запроса нет.
3. Внешние вызовы (`SetVarsForQueryGrid`, `qg.refresh`) оборачивать в `try/except` с `debug(...)` — чтобы ошибка не ломала переключение вкладок.
4. `StrToInt('%' + строка)` падает при пустой строке → проверять `Length(...) < N` перед `StrToInt`, в `except` — fallback `'%100000'`.
5. `GetQueryFilter(...)` для FilterType=2 должен **всегда** возвращать минимум `'%100000'`, никогда `''`.
6. «Шумные» `debug(...)` (предупреждения о ненастроенных объектах) прятать за `if DebugModeKOK80Poisk then debug(...)`.

---

## 4. Паттерн: фильтр «Показать только ошибки» (ImportDataInspection)

- Состояние фильтра — глобальный флаг `OnlyErrorsMode: Boolean`.
- Единственная точка перерисовки — `RefreshGridView`: вызывает `ShowErrorsOnly` или `ShowAllRows` по флагу.
- Все действия, меняющие данные (удаление строки, «Удалить записи с ошибками», перепроверка), вызывают `RefreshGridView`, а не `ShowAllRows` — иначе фильтр слетает.
- Снимать фильтр — только кнопкой («Показать все»), при создании окна проверки флаг сбрасывается.

## Паттерн: поиск в TListWindow (окно «Найти и заменить»)

- Панель с `TComboBox` + кнопка «X» на `LW.ToolBar` (позиция `LW.ToolBar.Width + 4`).
- Ввод → `TTimer` (300 мс) → применить фильтр (не на каждый символ).
- Фильтр: `CustomFilter` с `==` по всем видимым колонкам Grid формы + поле сравнения; fallback — скан + RecId.
- Ссылки на компоненты окна (форму, edit, timer) хранить в глобальных переменных и обнулять в `finally` перед `LW.Free`.
- Двойной клик по строке: `LW.FormView.Grid.OnDblClick := @ListWindowSelectClick;` → `LW.ModalResult := mrOk`, значение берётся из `Fm[FieldName]`.

---

## Паттерн: таблица настроек в ui действия (grid + component)

DX поддерживает в `<ui>` действия таблицу `<grid>` с колонками-компонентами формы — не нужен отдельный вызов действия на каждую пару:
```xml
<grid name="pairs" caption="Надпись -> группа" height="250">
  <component name="lbl" caption="Надпись / Кнопка" filter="TdxLabel;TdxButton" required="1"/>
  <component name="grp" caption="Группа настроек" filter="TdxGroupBox;TdxPanel" required="1"/>
</grid>
```
- Значения приходят в функцию как `TVariantArray2d`: строка i — `Pairs[i][0]` (имя компонента), `Pairs[i][1]`; обход `for i := 0 to Length(Pairs) - 1`.
- Примеры: «Индивидуальная видимость_доступность компонентов.epas» (grid + component + ifgrid-валидация), «Закладки CustomizationBookmark.epas», ImportDataInspection (`<field source="table">`, `<form>` + `<field source="...">`).
- Внутри grid доступны также `<list>`, `<checkbox>`, `<expr>`, `<form>`, `<childform>`, `<divider>`, `<ifgrid>` c `<if expr=... msg=... focus=...>`.
- Универсальность TdxGroupBox/TdxPanel: оба — наследники TWinControl, кастовать в `TWinControl` (Parent/Left/Top/Width/Height/Visible доступны), а не в TdxGroupBox.

---

## 5. Известные файлы и их состояние

| Файл | Состояние |
|---|---|
| `DX_selection_extensions/kok80-poisk3.1.epas` | **Рабочая** версия 3.1: EConvertError в SetVarsForQueryGrid исправлен, защиты от nil в GetQueryFilter/GetDomSources/GetQueryGridCode, try/except в OnTimer/ClearFilterField/MainWindow_pages_OnChange, debug «не настроен в конфигураторе» только при DebugModeKOK80Poisk |
| `DX_selection_extensions/kok80-poisk3.1 (есть ошибка компиляции).epas` | Битая заготовка (~410 строк, нет большинства процедур) — не использовать |
| `DX_selection_extensions/kok80-poisk3.00.epas` | Полный оригинал 3.00 (3999 строк), база для правок |
| `DX_My_extensions/Работа с файлами/ImportDataInspection.epas` | 3.1: флаг `OnlyErrorsMode` для фильтра ошибок + поиск в окне «Найти и заменить» (CustomFilter `==`, fallback скан по RecId), отладка убрана |

| `DX_selection_extensions/Настройки CustomizationLbl.epas` | 1.5.0: настройки через таблицу `<grid>` (Надпись/Кнопка -> TdxGroupBox/TdxPanel), ShowSettings/SetupPair через TWinControl |
| `DX_My_extensions/Настройки ограничений/Индивидуальная видимость_доступность компонентов.epas` | Образец ui-паттерна `<grid>` + `<component>` + `<ifgrid>` |
| `DX_My_extensions/Оформление и темы/Закладки CustomizationBookmark.epas` | Образец grid-настроек (TabsGrid) |

Версия расширения указывается в шапке `{@module ... Version=3.1 ...@}`.

---

## 6. Ссылки на документацию

- Фильтр формы (TFilterObject): https://wiki.mydataexpress.ru/programmirovanie_v_dx:filtr_formy
- Переход на запись (Move*/Locate/BoF/Eof/DisableControls): https://wiki.mydataexpress.ru/programmirovanie_v_dx:perexod_na_zapis
- Корень wiki: https://wiki.mydataexpress.ru/
- Описание функций скрипта — в справке DX (например, `StrToIntDef(S, def): LongInt` — число по глобальным настройкам форматирования).

## 7. Процесс работы (для нового чата)

1. Перед правками прочитать нужные участки grep'ом/read'ом (файлы 2000–4000 строк).
2. Кодировка файлов — UTF-8 без BOM; русские комментарии сохранять как есть.
3. Компиляция проверяется **только пользователем в DataExpress** — нельзя проверить локально; писать код аккуратно, литералы с кавычками перепроверять посимвольно.
4. После правок давать пользователю сценарий проверки (что кликнуть, что ожидать, что смотреть в Debug).
5. Отладочный вывод добавлять через `Debug('Префикс: ...')` и убирать после стабилизации.
