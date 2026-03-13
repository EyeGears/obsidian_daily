# Промпт для разработки демо Thunder Age JRPG на Godot 4

> Этот промпт предназначен для передачи AI-помощнику для разработки прототипа игры.
> Используйте как основу для Claude, ChatGPT или другого AI кодировщика.

---

## ЗАДАЧА: Разработать демо Thunder Age JRPG на Godot 4

Ты — сеньор-разработчик Godot 4, специализирующийся на тактических JRPG. Твоя задача — создать рабочий прототип боевой системы Thunder Age.

---

## ТРЕБОВАНИЯ К ДЕМО

### Минимальный MVP (первая неделя)

**Что ДОЛЖНО работать:**

1. **Сцена боя (3D)**
   - Изометрическая камера (45°, над полем)
   - Сетка 10x10 клеток (каждая клетка = 1 метр)
   - 3 юнита на поле: Эллиот + 2 врага (простые демоны)
   - Визуальные модели: кубы/капсулы (placeholder) с цветами

2. **Базовая боевая система**
   - Turn-based система: таймлайн инициативы внизу экрана
   - Ход персонажа: выбрать клетку → переместиться (1 ОВ)
   - Атака: выбрать врага из 5 случайных карт → нанести урон
   - HP бары над персонажами
   - Логирование всех действий в консоли

3. **Управление (Godot Input System)**
   - WASD: навигация по клеткам (подсветить доступные)
   - ЛКМ: выбрать карту / цель
   - ПКМ: отмена
   - Space: пропустить ход

4. **UI (Canvas)**
   - Таймлайн инициативы (прямоугольники с именами юнитов)
   - Рука карт (5 прямоугольников с названиями приёмов)
   - HP бары (Canvas Rect с цветами)
   - Лог боя (Label с историей последних 10 действий)

5. **Логика**
   - Расчёт урона: base_damage + crit_roll
   - Инициатива: INIT + d6
   - Система действий: 1 движение + 1 атака за ход (или 2 движения)

### Расширенный прототип (недели 2–3)

**Дополнительно:**
- Демон-Пёс (быстрый, стайный)
- Демон-Громила (танк, медленный)
- Статус-эффекты (Горение, Кровотечение, Оглушение)
- Элементальные комбинации (огонь + масло = усиление)
- Интерактивные объекты (взрывная бочка на сетке)
- Система камеры (follow юнита, зум на атаку)
- Audio (звуки удара, щелчка интерфейса)

---

## АРХИТЕКТУРА GODOT

### Структура папок

```
project/
├── scenes/
│   ├── battle/
│   │   ├── BattleManager.gd           # Главный контроллер боя
│   │   ├── BattleUI.gd                # Управление интерфейсом
│   │   ├── BattleGrid.gd              # Сетка и позиции
│   │   ├── TurnTimeline.gd            # Таймлайн инициативы
│   │   ├── CardHand.gd                # Рука карт
│   │   └── battle_scene.tscn           # Главная сцена боя
│   │
│   ├── units/
│   │   ├── Unit.gd                    # Базовый класс персонажа
│   │   ├── Player.gd                  # Эллиот (расширение Unit)
│   │   ├── Enemy.gd                   # Враги (AI)
│   │   ├── unit_3d.tscn               # 3D модель юнита
│   │   └── elliot_unit.tscn           # Эллиот
│   │
│   ├── abilities/
│   │   ├── Ability.gd                 # Базовый класс способности
│   │   ├── abilities.json             # База данных всех приёмов
│   │   └── AbilityResolver.gd         # Применение способности
│   │
│   └── environment/
│       ├── BattleMap.gd               # Парсинг уровня
│       ├── GridCell.gd                # Отдельная клетка
│       └── InteractiveObject.gd       # Бочка, взрывчатка и т.д.
│
├── scripts/
│   ├── systems/
│   │   ├── CombatSystem.gd            # Основные правила боя
│   │   ├── InitiativeSystem.gd        # Подсчёт инициативы
│   │   ├── DamageCalculator.gd        # Формулы урона
│   │   ├── StatusEffectSystem.gd      # Статусы и дебаффы
│   │   └── TacticsSystem.gd           # ИИ напарников
│   │
│   ├── data/
│   │   ├── UnitData.gd                # Характеристики персонажей
│   │   ├── AbilityData.gd             # Данные приёмов
│   │   └── BalanceConstants.gd        # Константы баланса
│   │
│   └── utils/
│       ├── GridUtils.gd               # Работа с сеткой
│       ├── MathUtils.gd               # Расчёты
│       └── Logger.gd                  # Логирование

├── assets/
│   ├── models/ (3D models - placeholder)
│   ├── materials/ (Shader materials)
│   ├── audio/ (SFX)
│   └── ui/ (Fonts, icons)
│
└── project.godot
```

### Основные сцены

```
BattleScene (Node3D)
├── Camera3D (изометрическая)
├── DirectionalLight3D
├── BattleGrid (Node3D)
│   ├── GridCell x100 (CSGBox3D)
│   └── InteractiveObjects
├── Units (Node3D)
│   ├── Player (Unit3D)
│   ├── Enemy1 (Unit3D)
│   └── Enemy2 (Unit3D)
├── BattleUI (CanvasLayer)
│   ├── HPBars
│   ├── CardHand (5 карт)
│   ├── TurnTimeline
│   ├── LogPanel
│   └── ActionButtons
└── AudioStreamPlayer (BGM)
```

---

## GODOT-СПЕЦИФИЧНЫЕ РЕШЕНИЯ

### 1. Система таймлайна инициативы

```gdscript
# TurnTimeline.gd
extends Node

var turn_order: Array[Unit] = []  # Отсортированные юниты
var current_turn_index: int = 0

func calculate_initiative(units: Array[Unit]) -> void:
    var turn_data = []
    for unit in units:
        var init_roll = randi_range(1, 6)
        var total_init = unit.stats.initiative + unit.stats.reflex + init_roll
        turn_data.append({"unit": unit, "init": total_init})

    turn_data.sort_custom(func(a, b): return a["init"] > b["init"])
    turn_order = turn_data.map(func(d): return d["unit"])

func get_next_unit() -> Unit:
    var unit = turn_order[current_turn_index]
    current_turn_index = (current_turn_index + 1) % turn_order.size()
    return unit
```

### 2. Сетка и позиционирование

```gdscript
# GridUtils.gd
class_name GridUtils
extends Node

const GRID_SIZE: int = 10
const CELL_SIZE: float = 1.0  # 1 метр = 1 клетка

static func world_to_grid(world_pos: Vector3) -> Vector2i:
    var grid_x = int(world_pos.x / CELL_SIZE)
    var grid_z = int(world_pos.z / CELL_SIZE)
    return Vector2i(grid_x, grid_z)

static func grid_to_world(grid_pos: Vector2i) -> Vector3:
    return Vector3(grid_pos.x * CELL_SIZE, 0, grid_pos.y * CELL_SIZE)

static func get_distance(from: Vector2i, to: Vector2i) -> int:
    return maxi(absi(from.x - to.x), absi(from.y - to.y))  # Чебышёв

static func get_reachable_cells(from: Vector2i, max_distance: int) -> Array[Vector2i]:
    var cells: Array[Vector2i] = []
    for x in range(max(0, from.x - max_distance), min(GRID_SIZE, from.x + max_distance + 1)):
        for z in range(max(0, from.y - max_distance), min(GRID_SIZE, from.y + max_distance + 1)):
            var cell = Vector2i(x, z)
            if GridUtils.get_distance(from, cell) <= max_distance:
                cells.append(cell)
    return cells
```

### 3. Карточная система

```gdscript
# CardHand.gd
extends Node

var ability_pool: Array[Ability] = []  # Полный пул способностей персонажа
var current_hand: Array[Ability] = []  # 5 текущих карт

func draw_hand() -> void:
    current_hand.clear()
    for i in range(5):
        var random_ability = ability_pool[randi() % ability_pool.size()]
        current_hand.append(random_ability)
    emit_signal("hand_updated")

func reshuffle_hand(stamina_cost: int) -> bool:
    if current_unit.stamina >= stamina_cost:
        current_unit.stamina -= stamina_cost
        draw_hand()
        return true
    return false
```

### 4. Система повреждений

```gdscript
# DamageCalculator.gd
extends Node

static func calculate_damage(attacker: Unit, ability: Ability, defender: Unit, position_bonus: float = 1.0) -> int:
    var base_damage = ability.damage_base
    var scaling = get_scaling_bonus(attacker, ability)
    var critical = 1.0 if randf() < get_crit_chance(attacker) else 1.0
    var final_damage = int((base_damage + scaling) * position_bonus * critical) - defender.armor
    return maxi(final_damage, 1)  # Минимум 1 урон

static func get_scaling_bonus(attacker: Unit, ability: Ability) -> float:
    match ability.damage_type:
        "PHYSICAL": return attacker.stats.physique * 2
        "FIRE": return attacker.stats.psionics * 2
        "ELECTRIC": return attacker.stats.psionics * 2
        "PSIONIC": return attacker.stats.psionics * 2.5
        _: return 0.0

static func get_crit_chance(attacker: Unit) -> float:
    return 0.05 + (attacker.stats.reflex * 0.015)  # 5% + 1.5% за каждый уровень рефлекса
```

### 5. Input System (WASD + Mouse)

```gdscript
# BattleInput.gd
extends Node

signal movement_input(direction: Vector2i)
signal card_selected(card_index: int)
signal target_selected(grid_pos: Vector2i)
signal turn_skipped()

func _input(event: InputEvent) -> void:
    if event is InputEventKey and event.pressed:
        match event.keycode:
            KEY_W: emit_signal("movement_input", Vector2i(0, -1))
            KEY_A: emit_signal("movement_input", Vector2i(-1, 0))
            KEY_S: emit_signal("movement_input", Vector2i(0, 1))
            KEY_D: emit_signal("movement_input", Vector2i(1, 0))
            KEY_1, KEY_2, KEY_3, KEY_4, KEY_5:
                var index = event.keycode - KEY_1
                emit_signal("card_selected", index)
            KEY_SPACE:
                emit_signal("turn_skipped")

    if event is InputEventMouseButton and event.pressed:
        if event.button_index == MOUSE_BUTTON_LEFT:
            var ray_from = get_viewport().get_camera_3d().project_ray_origin(event.position)
            var ray_normal = get_viewport().get_camera_3d().project_ray_normal(event.position)
            var grid_pos = pick_grid_cell(ray_from, ray_normal)
            emit_signal("target_selected", grid_pos)
```

### 6. Система состояния юнита

```gdscript
# Unit.gd
extends Node3D
class_name Unit

@export var unit_name: String = "Unit"
@export var max_hp: int = 100
@export var max_stamina: int = 8
@export var stamina_regen: int = 4

var hp: int
var stamina: int
var grid_position: Vector2i
var status_effects: Dictionary = {}  # {"burning": 3, "bleeding": 2}
var ability_pool: Array[Ability] = []

func _ready() -> void:
    hp = max_hp
    stamina = max_stamina

func take_damage(amount: int) -> void:
    hp -= amount
    if hp <= 0:
        die()

func apply_status(status_name: String, duration: int) -> void:
    status_effects[status_name] = duration

func tick_status_effects() -> void:
    for status in status_effects.keys():
        status_effects[status] -= 1
        if status_effects[status] <= 0:
            status_effects.erase(status)
```

---

## ROADMAP ВАЙБКОДИНГА (Week-by-Week)

### Week 1: Foundation

- [ ] Создать базовую сцену боя (камера + сетка)
- [ ] Placeholder 3D модели (кубы с цветами)
- [ ] Unit класс + Эллиот + простой враг
- [ ] GridUtils и система сетки
- [ ] Таймлайн инициативы (список в консоли)
- [ ] WASD управление + визуал доступных клеток

### Week 2: Core Combat

- [ ] Карточная система (5 случайных карт)
- [ ] Простая система атак (выбрать врага → наносит урон)
- [ ] HP бары (UI Canvas)
- [ ] Логирование боя
- [ ] Turn manager (кто ходит → показать карты → ждать ввода)

### Week 3: Polish & Extra

- [ ] Система статус-эффектов
- [ ] Демон-Пёс (стайная механика)
- [ ] Взрывная бочка (интерактивный объект)
- [ ] Звуковые эффекты
- [ ] Анимация атак

### Week 4: Extended

- [ ] Система напарников + простой ИИ
- [ ] Более сложные враги
- [ ] Переходы сцен
- [ ] Главное меню

---

## КРИТИЧЕСКИЕ ПРОВЕРКИ

При разработке постоянно проверяй:

- [ ] **Производительность:** FPS > 60 с 100 объектами на сетке
- [ ] **Читаемость:** Четкие визуалы, не запутанный UI
- [ ] **Ввод:** Отзывчивый контроль, нет задержек
- [ ] **Баланс:** Враги не слишком сильные, не скучные
- [ ] **Лог:** Все действия зафиксированы для дебага

---

## ПОЛЕЗНЫЕ РЕСУРСЫ

- Godot 4 документация: https://docs.godotengine.org/en/stable/
- Тактический JRPG туториал (для вдохновения)
- GDScript best practices

## ТОЧКА ВХОДА

Начни с `BattleManager.gd` как главного контроллера. Он:
1. Инициализирует сцену
2. Создаёт юнитов
3. Запускает таймлайн
4. Ждёт ввода игрока
5. Применяет действие
6. Переходит к следующему юниту

Good luck! 🎮
