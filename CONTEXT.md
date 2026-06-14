# Hidebar - контекст и устройство аддона

Hidebar - это компактный центр управления видимостью для сложных Blender-проектов. Вместо постоянного поиска коллекций, объектов и Mask-модификаторов в большом Outliner пользователь создает собственные Panels с понятными названиями и оставляет в них только часто используемые элементы. Из одной панели можно быстро показать или скрыть отдельный элемент, временно изолировать нужную часть сцены через Solo или скрыть и затем точно восстановить целый рабочий набор. Аддон особенно полезен для персонажей, окружения, света, вариантов модели и эффектов в насыщенных сценах.

## Главная идея

Пользователь создает Panels. Каждая Panel может содержать:

- Collections;
- Objects;
- Mask modifiers.

Panel имеет два разных режима работы:

- `Panels` - быстрые рабочие кнопки видимости и Solo;
- `Settings` - создание Panels, добавление элементов, сортировка, переименование, импорт и экспорт.

Panel в интерфейсе и `panel` в коде означают одну и ту же сущность.

## Файлы

- `__init__.py` - весь рабочий код Hidebar.
- `blender_manifest.toml` - метаданные Blender Extension.
- `AI_CONTEXT.md` - это описание продукта и архитектуры.
- `HyperShot.py` - локальный архитектурный ориентир; Hidebar его не импортирует и не регистрирует.

Аддон намеренно остается однофайловым. Его размер пока не оправдывает разделение на Python-модули.

## Как читать `__init__.py`

Файл устроен сверху вниз:

1. `CONFIG & RUNTIME` - конфигурация и временное состояние сессии.
2. `LOGGER` - консольное логирование.
3. `UTILITIES: SYSTEM` - общие helpers для Panels, Masks и Blender context.
4. `OBJECTS & COLLECTIONS` - snapshot-структуры, visibility helpers и managers.
5. `DATA STRUCTURES` - данные, которые сохраняются в `.blend`.
6. `PANEL OPERATIONS` - создание, удаление, импорт, экспорт и скрытие Panels.
7. `ITEM OPERATIONS` - добавление, сортировка, visibility и Solo элементов.
8. `SYSTEM` - служебные операторы.
9. `UI INFRASTRUCTURE` - динамическая регистрация Sidebar Panel.
10. `UI PANELS` - интерфейс, UI lists, menus и Preferences.
11. `REGISTER` - классы, handlers и свойства Scene.

Названия и порядок разделов повторяют подход Hyper Shot. Внутри managers и групп operators сначала расположены основные пользовательские операции, затем их helpers.

## Архитектурный стиль

Hidebar построен как младший родственник Hyper Shot:

- `HB_Config` соответствует роли `HS_Config`;
- `HB_Runtime` и `hb_runtime` соответствуют `HS_Runtime` и `hs_runtime`;
- `HB_Logger` и `hb_logger` соответствуют централизованному logger Hyper Shot;
- collection/object visibility находится в managers;
- Blender operators остаются тонкими;
- UI разделен на самостоятельные компоненты;
- данные сцены доступны через один корень `scene.hidebar`;
- callbacks начинаются с `on_`;
- зарегистрированные классы собраны в `CLASSES` по слоям.

При этом Hidebar не копирует тяжелые системы Hyper Shot, которые ему не нужны.

## HB_Config

`HB_Config` хранит все общие константы и симметричную UI-конфигурацию:

- `ADDON_ID` - ID аддона в Blender Preferences.
- `ADDON_VERSION` - версия Python-кода и `bl_info`.
- `EXPORT_VERSION` - версия JSON-формата Panel.
- `TAB_PANELS`, `TAB_SETTINGS` - внутренние ID главного меню.
- `DEFAULT_UI_TAB` - стандартное имя вкладки Sidebar.
- `DEFAULT_PANEL_NAME_PREFIX` - основа автоматических имен `Panel N`.
- `ITEM_COLLECTION`, `ITEM_OBJECT`, `ITEM_MASK` - внутренние типы элементов.
- `DIRECTION_UP`, `DIRECTION_DOWN` - направления сортировки.
- `MASK_OWNER_TYPES` - типы объектов, на которых ищутся Mask modifiers.
- `EMPTY_MASK_ITEM`, `EMPTY_MASK_NAME` - безопасный вариант пустого списка масок.
- `PANEL_ENTRY_CONFIG` - связывает тип элемента с его collection, active index и foldout.
- `SETTINGS_SECTION_CONFIG` - единое описание трех симметричных секций Settings.

Конфигурационные словари нужны, чтобы Collections, Objects и Masks использовали один общий UI-код.

## Данные сцены

Главная точка входа:

```python
scene.hidebar
```

`HB_SceneData` содержит:

```python
scene.hidebar.panels
scene.hidebar.active_panel_index
scene.hidebar.active_tab
scene.hidebar.available_masks
```

Одна `HB_PanelItem` содержит:

```python
panel.panel_id
panel.name
panel.collections
panel.objects
panel.masks
```

Также Panel хранит active indices и foldout-состояния Settings.

`panel_id` - стабильная внутренняя идентичность. Имя Panel предназначено для человека и может повторяться. Новая Panel получает первое свободное имя `Panel N`.

Элементы используют Blender pointers, поэтому обычное переименование Collection или Object не разрушает ссылку. Поле `custom_name` хранит только пользовательский alias.

`HB_CollectionItem.get_safe_collection()` и `HB_ObjectItem.get_safe_object()` централизованно проверяют сохраненные Blender pointers. Они возвращают живой datablock или `None`, если Collection/Object удален; Object также должен оставаться связан с текущей Scene. UI, operators, managers и экспорт используют эти методы вместо прямой проверки pointer.

## Runtime

`HB_Runtime` имеет один module-level экземпляр:

```python
hb_runtime
```

Он хранит только временные данные:

- cache найденных Mask modifiers;
- Panel hide snapshots;
- collection/object Solo snapshots;
- состояние регистрации динамической UI Panel.

Runtime не сохраняется в `.blend`. Он очищается:

- при регистрации аддона;
- при отключении аддона;
- после загрузки другого `.blend` через `on_load_post`.

Если сохранить файл во время Solo и перезапустить Blender, Solo-индикатор исчезнет, а записанные в `.blend` visibility flags останутся такими, какими были в момент сохранения. Это осознанно не исправляется автоматически.

## Snapshot-структуры

Вместо неочевидных tuples используются dataclasses:

- `HB_CollectionVisibilityState` - Collection и ее `exclude`;
- `HB_ObjectVisibilityState` - Object, `hide_viewport` и `hide_render`;
- `HB_SoloSnapshot` - цель Solo и список сохраненных состояний;
- `HB_PanelVisibilitySnapshot` - состояния Collections и Objects одной скрытой Panel.

Эти структуры существуют только в runtime и расположены в начале раздела `OBJECTS & COLLECTIONS`, как `HS_VisStateData` в Hyper Shot. Сразу после них идут helpers и managers, которые создают и восстанавливают эти snapshots.

## Visibility managers

`HB_VisibilityUtils` отвечает за общие операции:

- обход дерева LayerCollection от родителей к детям;
- временный `Collection -> LayerCollection` map;
- поиск пути к Collection;
- безопасное сравнение Blender data blocks.

`HB_CollectionManager` отвечает за:

- скрытие Collections одной Panel;
- collection Solo;
- восстановление snapshot через `restore_snapshot()`;
- diff-присваивание одного `LayerCollection.exclude` через `apply_item()`.

`HB_ObjectManager` отвечает за:

- скрытие Objects одной Panel;
- object Solo;
- восстановление snapshot через `restore_snapshot()`;
- diff-присваивание `hide_viewport` и `hide_render` одного Object через `apply_item()`.

Оба manager устроены зеркально: сначала основные операции `hide_panel()` и `toggle_solo()`, затем helpers `restore_snapshot()` и `apply_item()`.

Managers содержат бизнес-логику. Operators проверяют входные данные, вызывают manager и возвращают `FINISHED` или `CANCELLED`.

## Обычные кнопки

Collection button переключает:

```python
LayerCollection.exclude
```

Object button синхронно переключает:

```python
Object.hide_viewport
Object.hide_render
```

Mask button синхронно переключает:

```python
Modifier.show_viewport
Modifier.show_render
```

Присваивание выполняется только тогда, когда текущее состояние отличается от требуемого.

## Hide / Restore Panel

При скрытии Panel:

1. Активные Solo этой Panel завершаются и восстанавливаются.
2. Для каждой Collection Panel определяется ее полное поддерево.
3. Сохраняются исходные `exclude` всех затрагиваемых потомков, даже если их нет в Panel.
4. Скрываются только Collections, непосредственно добавленные в Panel.
5. Сохраняются и скрываются Objects Panel.
6. `HB_PanelVisibilitySnapshot` помещается в `hb_runtime.panel_visibility_snapshots`.

Collections скрываются от детей к родителям. При повторном нажатии snapshot восстанавливается от родителей к детям. Поэтому порядок элементов Panel не влияет на результат, а состояния вложенных Collections возвращаются точно.

Panel hide сейчас не управляет Masks.

## Collection Solo

Collection Solo работает по всему активному View Layer, а не только по содержимому Panel:

1. Сохраняется состояние всех LayerCollections.
2. Все Collections временно исключаются.
3. Целевая Collection включается.
4. Ее потомкам возвращаются собственные исходные состояния.
5. Повторное нажатие восстанавливает полный snapshot сверху вниз.

Если пользователь переключает Solo с одной Collection на другую, сначала восстанавливается предыдущий snapshot, затем создается новый.

## Object Solo

Object Solo проверяет все Objects активного View Layer:

1. Целевой Object получает `hide_viewport=False` и `hide_render=False`.
2. Остальные Objects получают `True` для обоих флагов.
3. Уже подходящие состояния не присваиваются повторно.
4. Snapshot содержит только реально измененные Objects.
5. Выход из Solo восстанавливает только этот diff.

Подход рассчитан и на сцены с тысячами объектов: обход остается линейным, а snapshot и RNA-записи ограничены изменениями.

Collection Solo и Object Solo одной Panel независимы друг от друга.

## Mask cache

Dropdown масок не сканирует сцену при каждом `draw()`.

```python
hb_runtime.mask_items
hb_runtime.mask_lookup
```

Cache строится лениво при первом запросе или вручную кнопкой Refresh. Mask modifier идентифицируется через Object pointer и `persistent_uid` модификатора.

## Import / Export

Panel экспортируется в JSON со структурой:

```text
version
panel_id
panel_name
collections
objects
masks
```

Collections и Objects сохраняют реальное Blender-имя и alias. Masks сохраняют Object, `modifier_uid`, имя модификатора и alias.

При импорте:

- создается новая Panel;
- `panel_id` переиспользуется только без конфликта;
- при конфликте создается новый UUID;
- одинаковое пользовательское имя Panel разрешено;
- отсутствующие Blender data blocks пропускаются.

## UI

`HB_PT_MainPanel` только маршрутизирует отрисовку:

```python
HB_UI_MainMenu
HB_UI_Panels
HB_UI_Settings
```

`HB_UI_MainMenu` рисует кнопки `Panels` и `Settings`.

`HB_UI_Panels` рисует рабочие кнопки, используя один layer map на весь UI draw. Кнопки размещаются в `grid_flow` с равными колонками, поэтому ширина не зависит от имени элемента.

`HB_UI_Settings` использует `HB_Config.PANEL_ENTRY_CONFIG` и `SETTINGS_SECTION_CONFIG`, чтобы Collections, Objects и Masks имели зеркальную структуру без трех копий одного кода.

Методы `draw()` только читают состояние и строят UI. Они не выполняют тяжелые операции и не пишут логи.

## Динамическая вкладка Sidebar

Preference `ui_tab` задает вкладку 3D View Sidebar. При изменении вызывается `on_ui_tab_update()`, после чего `HB_PT_MainPanel` безопасно перерегистрируется с новой `bl_category`.

`panel_button_columns` задает количество равных колонок рабочих кнопок.

## Developer logging

`HB_Logger` пишет технические логи только в консоль Blender. В UI и файл они не выводятся.

Формат:

```text
[Hidebar] LEVEL EVENT | key=value duration_ms=0.0
```

Семантические цвета в интерактивном Terminal:

- `DEBUG` - серый;
- `INFO` - зеленый;
- `WARNING` - желтый;
- `ERROR` и traceback - красный;
- Collection events - фиолетовые;
- Object events - синие;
- Mask events - бирюзовые;
- Panel events - зеленые;
- `duration_ms` - желтый.

Если stream не является Terminal, ANSI-цвета автоматически отключаются.

Каждая пользовательская операция пишет одну агрегированную строку, а не лог на каждый Object или Collection.

## Регистрация

Главная `HB_PT_MainPanel` регистрируется отдельно, потому что ее `bl_category` зависит от Preferences.

`HANDLERS_MAP` централизует `load_post`. Функция `manage_handlers()` симметрично добавляет и удаляет handlers без дублей.

`on_load_post()` очищает runtime caches и snapshots после загрузки другого `.blend`. Постоянные данные Panels остаются в `scene.hidebar`.

Статические проверки:

```bash
python3 -m py_compile __init__.py
python3 -m compileall -q .
git diff --check
```
