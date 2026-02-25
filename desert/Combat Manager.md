# ⚙️ Combat Manager — структура и функции (на примере Hyper Light Drifter-style)

---

## 🎯 Цель

**CombatManager** координирует все аспекты боя:

- учёт врагов и игрока,
    
- контроль урона,
    
- тайминг начала/конца схватки,
    
- визуальную и аудио обратную связь,
    
- триггеры событий (двери, лут, музыка, checkpoint и т.д.).
    

---

## 🧩 Роль в архитектуре игры

`[Player]    │    ├── input → CombatManager → handle attack → check hits    │ [Enemies] ← status, AI, damage feedback    │    └── CombatManager → reward, remove, trigger arena clear`

CombatManager — “прослойка” между индивидуальными бойцами (игрок, враги) и игровой логикой уровня.

---

## 🧱 Основная структура класса

`# CombatManager.gd extends Node class_name CombatManager  var player : Player var enemies : Array = [] var active : bool = false var combat_zone : Area2D var encounter_started : bool = false var encounter_cleared : bool = false var combo_counter : int = 0 var last_hit_time : float = 0.0 var kill_count : int = 0  signal encounter_started signal encounter_cleared signal entity_damaged(target, amount) signal entity_killed(target)`

---

## ⚙️ Основные функции

|Метод|Назначение|
|---|---|
|`start_encounter()`|Активирует бой, включает музыку, закрывает двери, сбрасывает счётчики.|
|`register_enemy(enemy)`|Добавляет врага в список активных.|
|`unregister_enemy(enemy)`|Удаляет врага при смерти, проверяет завершение боя.|
|`process_hit(attacker, target, damage)`|Центральная точка обработки урона.|
|`check_combat_state()`|Проверяет, остались ли враги живыми.|
|`end_encounter()`|Завершает бой, открывает двери, даёт награды, триггерит эффекты.|

---

## 🔁 Логика работы по шагам

### 1️⃣ Начало боя

- Игрок входит в **зону боя (combat_zone)**.
    
- CombatManager получает сигнал:  
    `on_player_enter_zone()` → `start_encounter()`.
    
- Все враги активируются, двери блокируются.
    
- Музыка/освещение переключаются на “боевой режим”.
    

`func start_encounter():     active = true     encounter_started = true     combo_counter = 0     kill_count = 0     emit_signal("encounter_started")     for e in enemies:         e.activate()`

---

### 2️⃣ Обработка атак

Любое оружие, удар мечом или снаряд вызывает у CombatManager событие:  
`process_hit(attacker, target, damage)`

`func process_hit(attacker, target, damage):     if not active:         return      if target.has_method("take_damage"):         target.take_damage(damage)         emit_signal("entity_damaged", target, damage)          if target.health <= 0:             kill_count += 1             emit_signal("entity_killed", target)             unregister_enemy(target)             player.stats.energy += 1  # пример: энергия за убийство`

---

### 3️⃣ Проверка состояния боя

CombatManager отслеживает, сколько врагов осталось:

`func unregister_enemy(enemy):     if enemies.has(enemy):         enemies.erase(enemy)     if enemies.size() == 0:         end_encounter()`

---

### 4️⃣ Завершение боя

Когда все враги уничтожены:

- Музыка плавно переходит в “спокойную” тему.
    
- Двери открываются.
    
- Счётчики боя записываются (для статистики).
    
- Возможен спавн награды или воспроизведение эффекта.
    

`func end_encounter():     active = false     encounter_cleared = true     emit_signal("encounter_cleared")     open_doors()     show_victory_flash()`

---

## ⚡ Расширенные функции

|Имя|Назначение|
|---|---|
|`apply_combo_bonus()`|Повышает урон при цепных убийствах без получения урона.|
|`track_player_performance()`|Запоминает среднее время убийства врагов, точность и урон.|
|`pause_combat()` / `resume_combat()`|Приостанавливает бой при катсценах или паузе.|
|`spawn_wave(wave_data)`|Создаёт врагов волнами.|
|`apply_area_effect(type)`|Применяет особые эффекты зоны (например, “кровавая туманность”, “энергетический шторм”).|

---

## 🎵 Интеграция с другими системами

|Система|Взаимодействие|
|---|---|
|**AI Manager**|Передаёт сигнал `activate()` врагам при начале боя.|
|**Camera Controller**|Включает “battle zoom” и тряску при попаданиях.|
|**Audio Manager**|Меняет фон и эффекты (начало боя, добивание, победа).|
|**UI / HUD**|Показывает индикаторы здоровья врагов и счетчик комбо.|
|**Level Logic**|Контролирует двери, чекпоинты, триггеры прогресса.|

---

## 💫 Принципы дизайна (в духе Hyper Light Drifter)

1. **Боевые зоны — это “мини-арены”.**  
    CombatManager активируется, когда игрок входит в область.
    
2. **Никакой случайности.**  
    Урон фиксированный, тайминги предсказуемы.
    
3. **Мгновенная обратная связь.**  
    Все события (урон, смерть, удар) сопровождаются светом, звуком и остановкой времени (hit stop).
    
4. **Скорость реакции.**  
    CombatManager работает с минимальной задержкой — урон и фидбек синхронизированы с кадром попадания.
    
5. **Чистая сцепка систем.**  
    CombatManager ничего не знает о конкретных типах врагов, только о базовых интерфейсах (`take_damage()`, `activate()`, `is_dead()`).
    

---

## 📊 Пример данных (структура encounter-а)

`var encounter_data = {     "id": "arena_forest_1",     "waves": [         { "enemies": ["Slasher", "Shooter", "Shooter"], "spawn_delay": 0.0 },         { "enemies": ["Brute"], "spawn_delay": 2.0 }     ],     "rewards": { "heal_item": true, "key_drop": false } }`

---

## 🧠 Итог

**CombatManager** в духе _Hyper Light Drifter_:

- минималистичен,
    
- реактивен,
    
- не управляет напрямую игроком,
    
- отвечает за **боевую динамику арены**, а не за механику боя.
    

Он держит баланс между “чистым боем” и “контролем ритма” —  
чтобы каждый encounter ощущался как **отдельная мини-битва с началом, пиком и концом**.