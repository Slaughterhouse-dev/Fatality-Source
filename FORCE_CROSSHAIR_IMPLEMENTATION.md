# Force Crosshair - Полная Реализация

## Описание
Force Crosshair - это функция, которая принудительно отображает прицел в CS:GO даже когда игрок находится в прицеле снайперской винтовки. Обычно при использовании прицела (scope) стандартный прицел скрывается, но эта функция позволяет его оставить видимым.

---

## Архитектура Реализации

### 1. Конфигурационная Переменная
**Файл:** `misc/config.h` (строка 513)

```cpp
VAR( force_crosshair, "misc.force_crosshair" );
```

**Назначение:**
- Создает конфигурационную переменную `vars::misc.force_crosshair`
- Хранит состояние включения/выключения функции (bool)
- Сохраняется в конфигурационных файлах через систему `config_init`

**Использование в коде:**
```cpp
vars::misc.force_crosshair->get<bool>()  // Получить значение
vars::misc.force_crosshair->set(true)     // Установить значение
```

---

### 2. Интерфейс Меню
**Файл:** `menu/tabs/tab_visuals.cpp` (строка 588)

```cpp
ADD("Force crosshair", "visuals>misc>local>force crosshair", checkbox, cfg.misc.force_crosshair);
```

**Расположение в меню:**
- Вкладка: Visuals → Misc → Local
- Тип элемента: Checkbox (галочка)
- Рядом с опцией "Penetration crosshair"

**Альтернативная реализация (menu/menu.cpp, строка 1321):**
```cpp
std::make_shared<checkbox>( _w( "Force crosshair" ), vars::misc.force_crosshair )
```

---

### 3. Хук ConVar (Основная Логика)
**Файл:** `hooks/con_vars.cpp` (строки 18-27)

```cpp
int __fastcall hook::convar_get_int_client( ConVar* convar, void* edx )
{
    // Проверка адреса возврата из функции отрисовки прицела
    if ( reinterpret_cast< uintptr_t >( _ReturnAddress() ) == make_offset( "client.dll", sig_ret_to_draw_crosshair ) 
        && !local_player->get_scoped()      // Игрок НЕ в прицеле
        && local_player->get_alive()        // Игрок жив
        && vars::misc.force_crosshair->get<bool>() )  // Функция включена
    {
        return 3;  // Возвращаем значение, которое заставляет прицел отображаться
    }

    if ( vars::misc.birthday_mode->get<bool>() && fnv1a_rt( convar->name ) == FNV1A( "sv_party_mode" ) )
        return true;

    // Вызов оригинальной функции
    return original_vfunc( convar, convar_get_int_client )( convar, edx );
}
```

**Детальное объяснение:**

#### Условия активации:
1. **Проверка адреса возврата:** `_ReturnAddress() == make_offset("client.dll", sig_ret_to_draw_crosshair)`
   - Проверяет, что функция вызвана из кода отрисовки прицела
   - Сигнатура: `0x6a1ff8` (из `misc/offsets.h`)
   - Это гарантирует, что мы перехватываем именно нужный вызов

2. **Проверка состояния игрока:**
   - `!local_player->get_scoped()` - игрок НЕ использует прицел снайперки
   - `local_player->get_alive()` - игрок жив
   - Эти проверки предотвращают конфликты и баги

3. **Проверка настройки:**
   - `vars::misc.force_crosshair->get<bool>()` - функция включена пользователем

#### Возвращаемое значение:
- **3** - магическое число, которое заставляет игру отрисовывать прицел
- В CS:GO это значение соответствует режиму отображения прицела

---

### 4. Регистрация Хука
**Файл:** `misc/hooks.cpp` (строка 126)

```cpp
make_hook_virt(convar_get_int_client, 13, proxy::weapon_debug_spread_show());
```

**Параметры:**
- `convar_get_int_client` - функция-обработчик хука
- `13` - индекс виртуальной функции в vtable
- `proxy::weapon_debug_spread_show()` - ConVar, который хукается

**Proxy функция (misc/hooks.cpp, строки 20-23):**
```cpp
__declspec(noinline) void* weapon_debug_spread_show() {
    var(weapon_debug_spread_show);
    return weapon_debug_spread_show;
}
```

---

### 5. Сигнатуры и Офсеты
**Файл:** `misc/offsets.h`

```cpp
// Сигнатура адреса возврата из функции отрисовки прицела
constexpr uint64_t sig_ret_to_draw_crosshair = 0x6a1ff8;

// Офсет хука для ConVar::GetInt в client.dll
constexpr uint64_t hook_convar_get_int_client = 0x198060;
```

**Использование в массиве офсетов (строка 1831):**
```cpp
sig_ret_to_draw_crosshair,  // Добавлен в список для валидации
```

---

## Технические Детали

### Принцип Работы

1. **Перехват вызова:**
   - Игра вызывает `ConVar::GetInt()` для проверки настроек прицела
   - Наш хук перехватывает этот вызов

2. **Проверка контекста:**
   - Анализируем адрес возврата (`_ReturnAddress()`)
   - Убеждаемся, что вызов идет из функции отрисовки прицела

3. **Подмена значения:**
   - Если все условия выполнены, возвращаем `3` вместо реального значения
   - Игра интерпретирует это как команду отрисовать прицел

4. **Обход оригинальной логики:**
   - Игра думает, что прицел должен быть отображен
   - Стандартная логика скрытия прицела в scope обходится

### Связанные ConVar'ы

**weapon_debug_spread_show:**
- Оригинальная консольная переменная CS:GO
- Используется для отладки разброса оружия
- Мы хукаем её `GetInt()` метод

**cl_crosshair* (семейство переменных):**
- Стандартные настройки прицела в CS:GO
- Не хукаются напрямую, но влияют на отображение

---

## Интеграция в Проект

### Шаг 1: Добавить конфигурационную переменную
В файл с конфигурацией (например, `config.h`):
```cpp
namespace vars {
    struct {
        // ... другие переменные ...
        VAR( force_crosshair, "misc.force_crosshair" );
    } inline misc{};
}
```

### Шаг 2: Создать хук функцию
В файл с хуками (например, `hooks/convars.cpp`):
```cpp
int __fastcall hook::convar_get_int_client( ConVar* convar, void* edx )
{
    // Проверка для Force Crosshair
    if ( reinterpret_cast< uintptr_t >( _ReturnAddress() ) == 
         make_offset( "client.dll", sig_ret_to_draw_crosshair ) 
         && !local_player->get_scoped() 
         && local_player->get_alive() 
         && vars::misc.force_crosshair->get<bool>() )
    {
        return 3;
    }

    // Вызов оригинальной функции
    return original_vfunc( convar, convar_get_int_client )( convar, edx );
}
```

### Шаг 3: Объявить хук в заголовке
В файл `hooks/hooks.h`:
```cpp
namespace hook {
    int __fastcall convar_get_int_client(ConVar*, void*);
}
```

### Шаг 4: Зарегистрировать хук
В функции инициализации хуков:
```cpp
void init_hooks() {
    // Получаем ConVar
    ConVar* weapon_debug_spread_show = interfaces::cvar->find_var("weapon_debug_spread_show");
    
    // Регистрируем хук на vtable индекс 13 (GetInt)
    make_hook_virt(convar_get_int_client, 13, weapon_debug_spread_show);
}
```

### Шаг 5: Добавить в меню
В файл меню (например, `menu/visuals.cpp`):
```cpp
void create_visuals_menu() {
    // ... другие элементы ...
    
    // Добавляем чекбокс для Force Crosshair
    menu->add_checkbox("Force crosshair", vars::misc.force_crosshair);
}
```

### Шаг 6: Определить сигнатуры
В файл с офсетами (например, `offsets.h`):
```cpp
namespace offsets {
    // Адрес возврата из функции отрисовки прицела
    constexpr uint64_t sig_ret_to_draw_crosshair = 0x6a1ff8;
    
    // Офсет ConVar::GetInt в client.dll
    constexpr uint64_t hook_convar_get_int_client = 0x198060;
}
```

---

## Зависимости

### Необходимые Компоненты:

1. **Система хуков:**
   - `c_hook` - базовый класс хуков
   - `c_vtable_hook` - хук виртуальных таблиц
   - Макросы: `make_hook_virt`, `original_vfunc`

2. **Система конфигурации:**
   - `value` - класс для хранения настроек
   - `config_init` - инициализация конфигов
   - Макрос `VAR` для создания переменных

3. **Игровые интерфейсы:**
   - `ConVar` - класс консольных переменных
   - `local_player` - указатель на локального игрока
   - Методы: `get_scoped()`, `get_alive()`

4. **Утилиты:**
   - `_ReturnAddress()` - получение адреса возврата (intrinsic)
   - `make_offset()` - вычисление офсетов в модулях
   - `reinterpret_cast` - приведение типов

---

## Возможные Проблемы и Решения

### Проблема 1: Прицел не отображается
**Причины:**
- Неверная сигнатура `sig_ret_to_draw_crosshair`
- Игра обновилась, офсеты изменились

**Решение:**
- Найти новую сигнатуру через IDA/Ghidra
- Обновить значение в `offsets.h`

### Проблема 2: Прицел отображается всегда
**Причины:**
- Условие `!local_player->get_scoped()` не работает
- Неправильная логика проверки

**Решение:**
- Проверить реализацию `get_scoped()`
- Добавить дополнительные проверки состояния

### Проблема 3: Краш при использовании
**Причины:**
- `local_player` равен nullptr
- Хук вызывается в неправильном контексте

**Решение:**
```cpp
if ( !local_player || !local_player.is_valid() )
    return original_vfunc( convar, convar_get_int_client )( convar, edx );
```

### Проблема 4: Конфликт с другими читами
**Причины:**
- Несколько хуков на одну функцию
- Порядок вызова хуков

**Решение:**
- Использовать цепочку хуков (hook chain)
- Проверять, не захукана ли функция уже

---

## Оптимизации

### 1. Кэширование офсета
```cpp
static const uintptr_t cached_offset = make_offset("client.dll", sig_ret_to_draw_crosshair);
if ( reinterpret_cast< uintptr_t >( _ReturnAddress() ) == cached_offset )
```

### 2. Быстрая проверка настройки
```cpp
// Проверяем настройку первой (самая быстрая проверка)
if ( !vars::misc.force_crosshair->get<bool>() )
    return original_vfunc( convar, convar_get_int_client )( convar, edx );
```

### 3. Inline функции
```cpp
__forceinline bool should_force_crosshair() {
    return local_player 
        && !local_player->get_scoped() 
        && local_player->get_alive() 
        && vars::misc.force_crosshair->get<bool>();
}
```

---

## Альтернативные Реализации

### Вариант 1: Прямая модификация памяти
```cpp
// Патчим инструкцию, которая скрывает прицел
void patch_crosshair_hide() {
    uintptr_t addr = make_offset("client.dll", sig_crosshair_hide_instruction);
    DWORD old_protect;
    VirtualProtect((void*)addr, 6, PAGE_EXECUTE_READWRITE, &old_protect);
    memset((void*)addr, 0x90, 6); // NOP инструкции
    VirtualProtect((void*)addr, 6, old_protect, &old_protect);
}
```

**Плюсы:** Быстрее, нет overhead хука  
**Минусы:** Сложнее поддерживать, может сломаться при обновлении

### Вариант 2: Хук функции отрисовки
```cpp
void __fastcall hook::draw_crosshair(void* ecx, void* edx, bool scoped) {
    // Игнорируем параметр scoped, если включен force crosshair
    bool force_draw = vars::misc.force_crosshair->get<bool>();
    return original(draw_crosshair)(ecx, edx, force_draw ? false : scoped);
}
```

**Плюсы:** Более прямой подход  
**Минусы:** Нужно найти функцию отрисовки прицела

### Вариант 3: Модификация ConVar напрямую
```cpp
void force_crosshair_update() {
    static ConVar* cl_crosshair_draw = interfaces::cvar->find_var("cl_crosshair_draw");
    if (vars::misc.force_crosshair->get<bool>()) {
        cl_crosshair_draw->set_value(1);
    }
}
```

**Плюсы:** Простота  
**Минусы:** Может не работать в scope, игра может переопределить

---

## Тестирование

### Тест-кейсы:

1. **Базовая функциональность:**
   - ✓ Включить Force Crosshair
   - ✓ Взять AWP/Scout
   - ✓ Нажать ПКМ (прицелиться)
   - ✓ Проверить, что прицел виден

2. **Проверка условий:**
   - ✓ Прицел скрывается, когда функция выключена
   - ✓ Прицел не отображается, когда игрок мертв
   - ✓ Работает со всеми снайперскими винтовками

3. **Совместимость:**
   - ✓ Не конфликтует с другими визуальными функциями
   - ✓ Работает с кастомными прицелами
   - ✓ Сохраняется в конфиге

4. **Производительность:**
   - ✓ Нет падения FPS
   - ✓ Нет задержек при прицеливании

---

## Заключение

Force Crosshair - это относительно простая, но эффективная функция, которая демонстрирует:
- Работу с хуками виртуальных функций
- Перехват вызовов через анализ адреса возврата
- Интеграцию с системой конфигурации
- Условную логику на основе состояния игры

Реализация занимает всего ~10 строк кода в хуке, но требует правильной настройки инфраструктуры (хуки, конфиги, меню, офсеты).

**Ключевые файлы для переноса:**
1. `hooks/con_vars.cpp` - основная логика
2. `misc/config.h` - конфигурационная переменная
3. `menu/tabs/tab_visuals.cpp` - элемент меню
4. `misc/offsets.h` - сигнатуры и офсеты
5. `misc/hooks.cpp` - регистрация хука

**Время реализации:** ~30 минут для опытного разработчика  
**Сложность:** Средняя (требуется понимание хуков и игровых механик)
