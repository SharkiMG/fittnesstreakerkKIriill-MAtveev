# fittnesstreakerkKIriill-MAtveev

**Лабораторные работы по БД**

Перечень [лабораторных работ](https://edu.irnok.net/doku.php?id=db:main#%D0%BB%D0%B0%D0%B1%D0%BE%D1%80%D0%B0%D1%82%D0%BE%D1%80%D0%BD%D0%B0%D1%8F_%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%B0_5_%D1%82%D1%80%D0%B8%D0%B3%D0%B3%D0%B5%D1%80%D1%8B)



# Постановка задачи (вариант 38)

**Фитнес-трекер и планировщик тренировок**

*Сущности:* Упражнения (наименование, группа мышц, тип), тренировки (дата, продолжительность), пользователи (ID, ФИО, цель).

*Процессы:* Пользователи выполняют тренировки, состоящие из нескольких упражнений с определенным количеством подходов и повторений.

*Выходные документы:*

  - Для заданного пользователя выдать прогресс по рабочему весу в определенном упражнении за последний месяц, отсортированный по дате.
  - Выдать статистику по наиболее часто прорабатываемым группам мышц за указанный период, отсортированную по убыванию частоты.

# Лабораторная работа 1 (Проектирование логической и физической модели БД)


## Промпт к Дипсик

```text
Лаба по проектированию информационной модели для реляционных баз данных.
Предполагаем Postgresql.
Есть ошибки, замечания, неточности?

# Фитнес-трекер и планировщик тренировок

## Постановка задачи

*Сущности:*
    Упражнения (наименование, группа_мышц, тип),
    Тренировки (дата, продолжительность), 
    Пользователи (ID, ФИО, цель).

*Процессы:* 
    Пользователи выполняют тренировки, состоящие из нескольких упражнений с определенным количеством подходов и повторений.

*Выходные документы:*

  - Для заданного пользователя выдать прогресс по рабочему весу в определенном упражнении за последний месяц, отсортированный по дате.

  - Выдать статистику по наиболее часто прорабатываемым группам мышц за указанный период, отсортированную по убыванию частоты.

## ER-Модель
### Базовые сущности

    Пользователь(ФИО, цель), ключ - ID
    Упражнение(наименование, группа_мышц, тип), ключ - ID  
    Тренировка(дата, продолжительность), ключ - ID

### Отношения

    [Пользователь]-1,Required------------------N,Optional-[Тренировка]

    [Тренировка]-N,Required-----Выполнение------M,Required-[Упражнение]
                              подходы, повторения, вес

## Логическая модель

Используя правила преобразования ER-диаграмм получаем четыре *таблицы*:

  - ```User(user_id, full_name, goal)```, primary key - user_id
  - ```Exercise(exercise_id, name, muscle_group, type)```, primary key - exercise_id
  - ```Workout(workout_id, user_id, date, duration)```, primary key - workout_id
  - ```WorkoutExercise(workout_id, exercise_id, sets, reps, weight)```, composite primary key - (workout_id, exercise_id)

## Физическая модель

Зададим типы данных для атрибутов:

  - ```user_id::SERIAL```
  - ```full_name::varchar(100)```
  - ```goal::text```
  - ```exercise_id::SERIAL```
  - ```name::varchar(100)```
  - ```muscle_group::varchar(50)```
  - ```type::varchar(50)```
  - ```workout_id::SERIAL```
  - ```date::date```
  - ```duration::integer``` (в минутах)
  - ```sets::integer```
  - ```reps::integer```
  - ```weight::decimal(5,2)``` (вес в кг)

## Проверка нормальных форм

### Первая нормальная форма
Все значения всех атрибутов атомарные.

### Вторая нормальная форма
Выполняется: нет зависимостей неключевых атрибутов от части ключа

### Третья нормальная форма
Нет транзитивных зависимостей неключей от неключей

### Нормальная форма Бойса-Кодда
Соблюдена.
```

Отличная работа! Модель хорошо продумана и логична, но есть несколько важных замечаний и улучшений, которые стоит внести для создания более качественной базы данных.

### Критические замечания и рекомендации

#### 1. Первичный ключ для `WorkoutExercise`
**Проблема:** Использование составного первичного ключа `(workout_id, exercise_id)` может быть ограничивающим.
*   **Ограничения:** Не позволяет добавить одно и то же упражнение несколько раз в одну тренировку (например, с разными подходами/весами).
*   **Гибкость:** Суррогатный ключ дает больше гибкости для будущих изменений.

**Исправление:**
```sql
-- Было: WorkoutExercise(workout_id, exercise_id, sets, reps, weight)
-- Стало:
WorkoutExercise(
    workout_exercise_id SERIAL PRIMARY KEY,
    workout_id INTEGER NOT NULL REFERENCES Workout(workout_id),
    exercise_id INTEGER NOT NULL REFERENCES Exercise(exercise_id),
    sets INTEGER NOT NULL,
    reps INTEGER NOT NULL,
    weight DECIMAL(5,2)
);
```

#### 2. Добавление ограничений целостности
**Проблема:** Отсутствуют проверки на корректность данных.
*   **Отрицательные значения:** Количество подходов, повторений и вес не могут быть отрицательными.
*   **Обязательные поля:** Некоторые поля должны быть обязательными для заполнения.

**Исправление:**
```sql
-- Добавляем CHECK constraints
CREATE TABLE WorkoutExercise (
    workout_exercise_id SERIAL PRIMARY KEY,
    workout_id INTEGER NOT NULL REFERENCES Workout(workout_id) ON DELETE CASCADE,
    exercise_id INTEGER NOT NULL REFERENCES Exercise(exercise_id),
    sets INTEGER NOT NULL CHECK (sets > 0),
    reps INTEGER NOT NULL CHECK (reps > 0),
    weight DECIMAL(5,2) CHECK (weight >= 0),
    UNIQUE(workout_id, exercise_id) -- Обеспечиваем уникальность комбинации
);
```

#### 3. Улучшение структуры данных
**Проблема:** Некоторые типы данных можно улучшить для большей гибкости.
*   **Продолжительность:** Хранить продолжительность в минутах - хорошо, но можно добавить возможность хранить в интервале.
*   **Дата и время:** Для тренировок важно и время, а не только дата.

**Исправление:**
```sql
-- Улучшенная таблица Workout
CREATE TABLE Workout (
    workout_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES Users(user_id) ON DELETE CASCADE,
    workout_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    duration INTERVAL, -- Более гибкий тип для продолжительности
    notes TEXT -- Дополнительные заметки о тренировке
);
```

---

### Исправленная и улучшенная модель

#### Логическая модель
*   **`Users`** (`user_id`, full_name, goal)
*   **`Exercise`** (`exercise_id`, name, muscle_group, type)
*   **`Workout`** (`workout_id`, `user_id`, workout_date, duration, notes)
*   **`WorkoutExercise`** (`workout_exercise_id`, `workout_id`, `exercise_id`, sets, reps, weight)

#### Физическая модель (DDL для PostgreSQL)
```sql
CREATE TABLE Users (
    user_id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    goal TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Exercise (
    exercise_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    muscle_group VARCHAR(50) NOT NULL,
    type VARCHAR(50) NOT NULL,
    description TEXT
);

CREATE TABLE Workout (
    workout_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES Users(user_id) ON DELETE CASCADE,
    workout_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    duration INTERVAL,
    notes TEXT
);

CREATE TABLE WorkoutExercise (
    workout_exercise_id SERIAL PRIMARY KEY,
    workout_id INTEGER NOT NULL REFERENCES Workout(workout_id) ON DELETE CASCADE,
    exercise_id INTEGER NOT NULL REFERENCES Exercise(exercise_id),
    sets INTEGER NOT NULL CHECK (sets > 0),
    reps INTEGER NOT NULL CHECK (reps > 0),
    weight DECIMAL(5,2) CHECK (weight >= 0),
    UNIQUE(workout_id, exercise_id)
);

-- Индексы для ускорения часто используемых запросов
CREATE INDEX idx_workout_user_date ON Workout(user_id, workout_date);
CREATE INDEX idx_workout_exercise_workout ON WorkoutExercise(workout_id);
CREATE INDEX idx_workout_exercise_exercise ON WorkoutExercise(exercise_id);
CREATE INDEX idx_exercise_muscle_group ON Exercise(muscle_group);
```

#### Проверка нормальных форм (для исправленной модели)
*   **1NF:** Соблюдена.
*   **2NF:** Соблюдена, так как все неключевые атрибуты полностью зависят от целого первичного ключа.
*   **3NF/BCNF:** Соблюдена. Нет транзитивных зависимостей.

---

### Примеры запросов для "Выходных документов"

**1. Прогресс по рабочему весу для заданного пользователя и упражнения за последний месяц:**
```sql
SELECT 
    w.workout_date,
    we.weight,
    we.sets,
    we.reps
FROM WorkoutExercise we
JOIN Workout w ON we.workout_id = w.workout_id
JOIN Exercise e ON we.exercise_id = e.exercise_id
WHERE w.user_id = 1  -- ID заданного пользователя
    AND e.exercise_id = 5  -- ID заданного упражнения
    AND w.workout_date >= CURRENT_DATE - INTERVAL '1 month'
ORDER BY w.workout_date;
```

**2. Статистика по наиболее часто прорабатываемым группам мышц за указанный период:**
```sql
SELECT 
    e.muscle_group,
    COUNT(*) as workout_count,
    COUNT(DISTINCT w.workout_id) as unique_workouts
FROM WorkoutExercise we
JOIN Workout w ON we.workout_id = w.workout_id
JOIN Exercise e ON we.exercise_id = e.exercise_id
WHERE w.workout_date BETWEEN '2024-01-01' AND '2024-01-31'  -- Заданный период
GROUP BY e.muscle_group
ORDER BY workout_count DESC;
```

**Итог:** После внесения этих правок ваша модель станет более гибкой, надежной и соответствующей стандартам реляционного проектирования. Удачи в сдаче лабы!

## Промпт 2

```text
В таблице WorkoutExercise добавлен суррогатный ключ, но при этом сохраняется уникальность комбинации (workout_id, exercise_id).
Не приведет ли это к избыточности? Может ли одна тренировка содержать одно и то же упражнение несколько раз с разными параметрами?
```

**Отличный вопрос! Вы правы, это важный нюанс проектирования.**

### Анализ требования

Давайте рассмотрим два сценария:

#### Сценарий 1: Одно упражнение один раз за тренировку
- **Текущий подход:** `UNIQUE(workout_id, exercise_id)` предотвращает дублирование
- **Плюсы:** Простая структура, гарантия уникальности
- **Минусы:** Нельзя добавить суперсеты или повторы одного упражнения

#### Сценарий 2: Одно упражнение несколько раз за тренировку
- **Пример:** Жим лежа 3 подхода, потом после других упражнений еще 2 подхода
- **Требуется:** Убрать `UNIQUE` ограничение, добавить `exercise_order` или `set_number`

### Рекомендация

Для **фитнес-трекера** более гибким будет **Сценарий 2**, так как:
1. **Суперсеты** - распространенная практика
2. **Разделение на части** - иногда упражнение выполняется в начале и конце тренировки
3. **Гибкость** - не ограничивает пользователя

### Улучшенная модель:

```sql
CREATE TABLE WorkoutExercise (
    workout_exercise_id SERIAL PRIMARY KEY,
    workout_id INTEGER NOT NULL REFERENCES Workout(workout_id) ON DELETE CASCADE,
    exercise_id INTEGER NOT NULL REFERENCES Exercise(exercise_id),
    exercise_order INTEGER, -- Порядок выполнения в тренировке
    sets INTEGER NOT NULL CHECK (sets > 0),
    reps INTEGER NOT NULL CHECK (reps > 0),
    weight DECIMAL(5,2) CHECK (weight >= 0)
    -- Убрано UNIQUE(workout_id, exercise_id)
);

-- Добавляем индекс для порядка выполнения
CREATE INDEX idx_workout_exercise_order ON WorkoutExercise(workout_id, exercise_order);
```

### Когда сохранить уникальность?

Только если бизнес-требования явно запрещают повторение упражнений в одной тренировке.

**Вывод:** Для фитнес-трекера лучше убрать ограничение уникальности и добавить поле для порядка выполнения. Это дает больше гибкости пользователям.

## Полученные диаграммы:
### ER-диаграмма

```mermaid
erDiagram
    USERS ||--o{ WORKOUT : performs
    WORKOUT ||--o{ WORKOUT_EXERCISE : contains
    EXERCISE ||--o{ WORKOUT_EXERCISE : included_in
    
    USERS {
        integer user_id PK "SERIAL"
        varchar full_name
        text goal
        timestamp created_at
    }
    
    EXERCISE {
        integer exercise_id PK "SERIAL"
        varchar name
        varchar muscle_group
        varchar type
        text description
    }
    
    WORKOUT {
        integer workout_id PK "SERIAL"
        integer user_id FK
        timestamp workout_date
        interval duration
        text notes
    }
    
    WORKOUT_EXERCISE {
        integer workout_exercise_id PK "SERIAL"
        integer workout_id FK
        integer exercise_id FK
        integer exercise_order
        integer sets
        integer reps
        decimal weight
    }
```

## Логическая модель в виде Диаграммы классов UML-2.4

```mermaid
classDiagram
    class Users {
        +user_id: Integer (PK)
        +full_name: String
        +goal: String
        +created_at: DateTime
        +getWorkouts() List~Workout~
    }
    
    class Exercise {
        +exercise_id: Integer (PK)
        +name: String
        +muscle_group: String
        +type: String
        +description: String
        +getWorkoutExercises() List~WorkoutExercise~
    }
    
    class Workout {
        +workout_id: Integer (PK)
        +user_id: Integer (FK)
        +workout_date: DateTime
        +duration: Interval
        +notes: String
        +getUser() Users
        +getExercises() List~WorkoutExercise~
    }
    
    class WorkoutExercise {
        +workout_exercise_id: Integer (PK)
        +workout_id: Integer (FK)
        +exercise_id: Integer (FK)
        +exercise_order: Integer
        +sets: Integer
        +reps: Integer
        +weight: Decimal
        +getWorkout() Workout
        +getExercise() Exercise
    }
    
    Users "1" -- "*" Workout : performs
    Workout "1" -- "*" WorkoutExercise : contains
    Exercise "1" -- "*" WorkoutExercise : included_in
```

## Физическая модель БД

```mermaid
erDiagram
    users {
        integer user_id PK "SERIAL"
        varchar full_name "NOT NULL"
        text goal
        timestamp created_at "DEFAULT CURRENT_TIMESTAMP"
    }
    
    exercise {
        integer exercise_id PK "SERIAL"
        varchar name "NOT NULL"
        varchar muscle_group "NOT NULL"
        varchar type "NOT NULL"
        text description
    }
    
    workout {
        integer workout_id PK "SERIAL"
        integer user_id FK "NOT NULL"
        timestamp workout_date "NOT NULL"
        interval duration
        text notes
    }
    
    workout_exercise {
        integer workout_exercise_id PK "SERIAL"
        integer workout_id FK "NOT NULL"
        integer exercise_id FK "NOT NULL"
        integer exercise_order
        integer sets "NOT NULL, CHECK (sets > 0)"
        integer reps "NOT NULL, CHECK (reps > 0)"
        decimal weight "CHECK (weight >= 0)"
    }
    
    users ||--o{ workout : "FOREIGN KEY (user_id) REFERENCES users(user_id)"
    workout ||--o{ workout_exercise : "FOREIGN KEY (workout_id) REFERENCES workout(workout_id)"
    exercise ||--o{ workout_exercise : "FOREIGN KEY (exercise_id) REFERENCES exercise(exercise_id)"
```

## Заключение

В первой лабораторной работе при помощи теоретического инструментария проектирования БД и большой языковой модели "Дипсик" произведено проектирование ER-, логической и физической модели базы данных варианта 38. Основные замечания, решенные в ходе выполнения лабораторной работы:

  1. Реализовано две итерации консультации с БЯМ;
  2. БЯМ исправила существенные ошибки проектирования (суррогатные ключи, ограничения целостности);
  3. Добавлены CHECK-ограничения для валидации данных;
  4. Улучшена гибкость модели (возможность повторения упражнений в одной тренировке);
  5. Оптимизированы типы данных (INTERVAL для продолжительности, TIMESTAMP для даты тренировки).

Ссылка на чат: https://chat.deepseek.com/a/chat/s/c8158baf-f8cf-43d5-8a2b-9d2b41cd6178


### Лабораторная работа 2.

## 1. Логическая модель данных
# Сущности и связи:
users — пользователи
exercises — упражнения
workouts — тренировки
workout_exercises — связующая таблица (упражнения в тренировке с подходами, повторениями, весом)

users (id, name, goal)
workouts (id, user_id, date, duration_minutes)
exercises (id, name, muscle_group, type)
workout_exercises (id, workout_id, exercise_id, sets, reps, weight_kg)

## 2. Coздание таблиц
# users:
<img width="399" height="233" alt="image" src="https://github.com/user-attachments/assets/9112741b-6430-46c3-b767-e1beba3ebeca" />

# exercises:
<img width="532" height="224" alt="image" src="https://github.com/user-attachments/assets/5943d921-f400-41c0-b3d0-7757f8851ec9" />

# workouts
<img width="647" height="227" alt="image" src="https://github.com/user-attachments/assets/5b4d0a86-1e1f-4809-82bf-e8b50645eb33" />

# workout_exercises:
<img width="718" height="304" alt="image" src="https://github.com/user-attachments/assets/91e51241-fb2b-4b3b-a8e3-0f242c61b121" />

## 3. Наполнение таблиц
# users:
<img width="495" height="315" alt="image" src="https://github.com/user-attachments/assets/48afa6ca-b1c3-4a14-bf88-2b10eebbd5e7" />
<img width="498" height="291" alt="image" src="https://github.com/user-attachments/assets/08698a62-525c-4235-b244-b71bc51d97ef" />

# exercises:
<img width="627" height="312" alt="image" src="https://github.com/user-attachments/assets/be811473-9e89-4506-93e0-cb85a634d4c0" />
<img width="693" height="326" alt="image" src="https://github.com/user-attachments/assets/3eea0e74-8a39-49f9-af6d-98c75fe19d4f" />

# workouts
<img width="694" height="312" alt="image" src="https://github.com/user-attachments/assets/037d104b-e830-4895-8e5f-42e99b8f8f5d" />
<img width="521" height="315" alt="image" src="https://github.com/user-attachments/assets/1e02e3fa-898e-4750-8b38-9029e9994ff5" />

# workout_exercises:
<img width="883" height="366" alt="image" src="https://github.com/user-attachments/assets/05a722df-448a-436c-9d51-8e3116a05143" />
<img width="756" height="410" alt="image" src="https://github.com/user-attachments/assets/0effd1b5-d167-470b-be52-b30593d7124d" />


## 4. Содержательные SELECT-запросы с JOIN
# Запрос 1: Для заданного пользователя выдать прогресс по рабочему весу в определенном упражнении за последний месяц, отсортированный по дате.
<img width="699" height="480" alt="image" src="https://github.com/user-attachments/assets/41a37912-205e-451d-b595-1b2e2ce9eac7" />

# Запрос 2: Выдать статистику по наиболее часто прорабатываемым группам мышц за указанный период, отсортированную по убыванию частоты
<img width="536" height="473" alt="image" src="https://github.com/user-attachments/assets/0b27e4df-a673-43f2-ba8d-49ea115a557a" />

## 5. Проверка нормальных форм
# 5.1 Проверка соответствия 4НФ (Fourth Normal Form)
Таблица workout_exercises не содержит многозначных зависимостей. Все атрибуты функционально зависят от составного ключа (workout_id, exercise_id).

# 5.2 Проверка соответствия 5НФ (Fifth Normal Form)
Все связи декомпозированы без потерь информации:
Отношение M:M между workouts и exercises вынесено в отдельную таблицу workout_exercises
Нет избыточных соединений при восстановлении исходных данных

✅ Вывод: База данных соответствует 4-й и 5-й нормальным формам.

# Лабораторная работа 3.

# Представление для отслеживания прогресса пользователей по упражнениям 
<img width="974" height="299" alt="image" src="https://github.com/user-attachments/assets/d186edb6-a755-4706-9d9d-d51b179db622" />
<img width="974" height="349" alt="image" src="https://github.com/user-attachments/assets/4e8f765f-00c5-4b9a-ae25-d351201d6e8f" />

# Статистика по частоте проработки групп мышц
<img width="974" height="340" alt="image" src="https://github.com/user-attachments/assets/3f402939-c90f-4e5e-ba0a-70387016b266" />
<img width="974" height="295" alt="image" src="https://github.com/user-attachments/assets/fc765cfc-6ed1-4abc-a26c-dba6354197b5" />

# Детализированная информация о тренировках
<img width="974" height="299" alt="image" src="https://github.com/user-attachments/assets/e3c8296a-c3e0-4639-9541-3755d9f2a99d" />
<img width="974" height="217" alt="image" src="https://github.com/user-attachments/assets/473a7544-87a0-4033-8b9d-3d8621c3b50b" />

# Ежемесячный прогресс пользователей
<img width="974" height="421" alt="image" src="https://github.com/user-attachments/assets/14df7511-ca86-476d-adf9-7b1ef35e1b56" />
<img width="974" height="223" alt="image" src="https://github.com/user-attachments/assets/32574bb7-aa1a-4c19-8507-36e783d5735f" />

# Получение прогресса пользователя по конкретному упражнению за последний месяц
<img width="974" height="633" alt="image" src="https://github.com/user-attachments/assets/a933c96f-427b-44bc-9099-9d59954f9377" />

# Статистика по группам мышц за указанный период
<img width="974" height="895" alt="image" src="https://github.com/user-attachments/assets/1071c309-52a6-4f0c-8585-9ed860599a4d" />

# Добавление новой тренировки с упражнениями 
<img width="974" height="997" alt="image" src="https://github.com/user-attachments/assets/99dc5b1e-be79-447c-9b0e-7b97f2a84854" />

# Получение персональных рекомендаций для пользователя
<img width="974" height="985" alt="image" src="https://github.com/user-attachments/assets/3fb59e75-ba65-460e-9fd4-a2f4e4e7be91" />

# Расчет общего обьема нагрузки для тренировки
<img width="974" height="311" alt="image" src="https://github.com/user-attachments/assets/f5276a7a-47ac-49f1-8ca1-82840f8f75a9" />

# Получение лучшего результата пользователя по упражнению
<img width="974" height="493" alt="image" src="https://github.com/user-attachments/assets/c30e124f-ae66-44a5-a448-ff181dc0640f" />

### Лабораторная работа 4

## Анализ производительности
## Созданы генераторы данных, заполняющие таблицы 20000 записями

# 1.1 Генератор для таблицы users
<img width="974" height="525" alt="image" src="https://github.com/user-attachments/assets/5d97df37-913c-426a-8bfe-4ad66b58514b" />

# 1.2 Генератор для таблицы lubsanova.exercises (100 записей)
<img width="883" height="1242" alt="image" src="https://github.com/user-attachments/assets/d23f7cdd-0c80-43c9-8f0a-91179e515a8d" />
<img width="306" height="238" alt="image" src="https://github.com/user-attachments/assets/2f22993c-9078-4c3f-99c1-f0ac61fc02b6" />

# 1.3 Генератор для таблицы lubsanova.workouts (50000 записей)
<img width="974" height="1232" alt="image" src="https://github.com/user-attachments/assets/f5848a9f-a774-4d69-abe1-ede1873811fe" />

# 1.4 Генератор для таблицы lubsanova.workout_exercises (200000 записей)
<img width="974" height="996" alt="image" src="https://github.com/user-attachments/assets/354d233b-8694-4ac2-9c86-f7eea31dea11" />
<img width="256" height="97" alt="image" src="https://github.com/user-attachments/assets/b6c40d65-37e6-467e-be80-778a15af8c18" />

## Анализ планов выполнения запросов

# 2. Анализ производительности ДО оптимизации
# 2.1. Запрос 1: Прогресс пользователя по упражнению за месяц
<img width="833" height="683" alt="image" src="https://github.com/user-attachments/assets/a1c28c47-baeb-4171-885e-d2d73d2c87ef" />


# 2.2. Запрос 2: Статистика по группам мышц за период
<img width="583" height="749" alt="image" src="https://github.com/user-attachments/assets/d8cbd0fc-2189-4a40-aa5b-edf69ab9e3ec" />

<img width="381" height="204" alt="image" src="https://github.com/user-attachments/assets/1edecc4f-9b0b-4694-838c-7896724c0bcf" />

# 2.3. Создание индексов для оптимизации
<img width="738" height="705" alt="image" src="https://github.com/user-attachments/assets/f4bd8af2-2c81-465c-bb84-404530e82a5a" />

## Обновление статистики
<img width="974" height="680" alt="image" src="https://github.com/user-attachments/assets/4a76f583-2b63-46fd-b3e4-07339967a9e4" />

# 3. Анализ производительности ПОСЛЕ оптимизации
# 3.1. Запрос 1: Прогресс пользователя по упражнению за месяц (после индексов)
<img width="639" height="525" alt="image" src="https://github.com/user-attachments/assets/4994135d-7ab8-4128-9ec6-25bbf57faaa1" />


# 3.2. Запрос 2: Статистика по группам мышц за период (после индексов)
<img width="378" height="212" alt="image" src="https://github.com/user-attachments/assets/b255289b-6f9d-4a35-abb6-de5477eb086e" />
<img width="590" height="747" alt="image" src="https://github.com/user-attachments/assets/d2e066b0-a943-452a-9004-2631485cbe50" />



### Лабораторная работа 5

## Реализация бизнес-логики на уровне БД и системы аудита
## 1. Триггеры каскадного удаления

# 1.1. Триггерная функция для удаления тренировок пользователя
<img width="953" height="570" alt="image" src="https://github.com/user-attachments/assets/67656c43-2af9-45f7-8253-7b6d5add2588" />

# 1.2 Триггер для удаления упражнений из тренировок
<img width="974" height="548" alt="image" src="https://github.com/user-attachments/assets/cc3affc4-1721-4d97-9845-4617f5ca6d8f" />

## 2. Система аудита изменений

# 2.1. Создание таблицы-журнала для аудита
<img width="824" height="694" alt="image" src="https://github.com/user-attachments/assets/cf1b354a-c054-4c79-b149-f8f2746fc613" />

# 2.2. Триггер для аудита INSERT операций
<img width="974" height="922" alt="image" src="https://github.com/user-attachments/assets/58ba115a-4e77-4b1d-bc6b-1f211e067004" />

# 2.3. Триггер для аудита UPDATE операций
<img width="974" height="988" alt="image" src="https://github.com/user-attachments/assets/816baa65-81a2-42a2-bfae-6863822c8358" />

# 2.4. Триггер для аудита DELETE операций
<img width="974" height="949" alt="image" src="https://github.com/user-attachments/assets/3d88813f-2b01-4f5b-897c-3502866d3bc7" />

## 3. Бизнес-правила и ограничения

# 3.1. Проверка реалистичности веса
<img width="974" height="603" alt="image" src="https://github.com/user-attachments/assets/8bdf137d-e81b-464e-9a9f-4b531a9d90e5" />

# 3.2. Проверка прогресса (вес не может уменьшаться без причины)
<img width="974" height="887" alt="image" src="https://github.com/user-attachments/assets/ad87275a-6298-4e6d-b434-b5e21c628e08" />

## 4. Дополнительные функции для бизнес-логики

# 4.1. Функция расчета тренировочного объема
<img width="970" height="870" alt="image" src="https://github.com/user-attachments/assets/f987b3d6-4b0b-474d-8b01-e83e7c6c4871" />

# 4.2. Функция для получения рекомендаций
<img width="974" height="1100" alt="image" src="https://github.com/user-attachments/assets/c40a9a78-f1d5-4616-92c8-6cf6c69dc30a" />

## 5. Тестирование работы триггеров и функций
<img width="844" height="663" alt="image" src="https://github.com/user-attachments/assets/6c7c4d60-2b7a-490f-991c-9aa99c090f55" />

```
