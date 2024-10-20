##  Повышение производительности данных через оптимизацию структуры данных и SQL-запросов в PostgreSQL

1) Код генерации таблиц 
```sql
-- Таблица полов
CREATE TABLE Gender (
    id SERIAL PRIMARY KEY,
    gender_name TEXT NOT NULL
);

-- Таблица работников
CREATE TABLE Employee (
    id SERIAL PRIMARY KEY,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    middle_name TEXT,
    email TEXT NOT NULL,
    date_of_birth DATE,
    gender_id INT REFERENCES Gender(id) -- Ссылка на справочную таблицу Gender
);

--Таблица компаний
CREATE TABLE Company (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    year_founded INT,
    description text,
	industry TEXT
);

-- таблица должностей 
CREATE TABLE Position (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    responsibilities TEXT
);

-- Таблица навыков 
CREATE TABLE Skill (
    id SERIAL PRIMARY KEY,
    skill_name TEXT NOT NULL,
    skill_level INT CHECK (skill_level IN (1, 2, 3, 4, 5)), -- уровни: 1 - начальный, 2 - ниже среднего, 3 - средний, 4 - выше среднего, 5 - экспертный
    UNIQUE (skill_name, skill_level) -- составное уникальное ограничение
);

-- Таблица проектов
CREATE TABLE Project (
    id SERIAL PRIMARY KEY,
    project_name TEXT NOT NULL,
    description TEXT
);

-- Таблица, связывающая сотрудников с компаниями и позициями
CREATE TABLE Employee_Position (
    id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES Employee(id),
    company_id INT REFERENCES Company(id),
    position_id INT REFERENCES Position(id),
    start_date DATE,
    end_date DATE
);

-- Таблица, связывающая сотрудников с навыками
CREATE TABLE Employee_Skill (
    id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES Employee(id),
    skill_id INT REFERENCES Skill(id)
);

-- Таблица, связывающая сотрудников с проектами
CREATE TABLE Employee_Project (
    id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES Employee(id),
    project_id INT REFERENCES Project(id),
    start_date DATE,
    end_date DATE,
    role_in_project TEXT
);
```
2) Код генерации данных для таблиц 

Вставка полов сотрудников 
```sql 
INSERT INTO gender (gender_name)
VALUES
('Мужчина'),
('Женщина');
```
Генерация сотрудников 10 000 000 записей
```sql
BEGIN;
DO $$
DECLARE
    v_first_name TEXT;
    v_last_name TEXT;
    v_middle_name TEXT;
    v_email TEXT;
    v_date_of_birth DATE;
    v_gender_id INT;
    i INT;
BEGIN
    FOR i IN 1..10000000 LOOP
        v_first_name := substring(md5(random()::text), 1, (random() * 250 + 5)::INT);
        v_last_name := substring(md5(random()::text), 1, (random() * 250 + 5)::INT);
        IF random() > 0.5 THEN
            v_middle_name := substring(md5(random()::text), 1, (random() * 250 + 5)::INT);
        ELSE
            v_middle_name := NULL;
        END IF;
        v_email := substring(md5(random()::text), 1, (random() * 245 + 10)::INT) || '@yandex.ru';
        v_date_of_birth := date '1950-01-01' + (random() * 20541)::INT;
        v_gender_id := CASE WHEN random() < 0.5 THEN 1 ELSE 2 END;

        INSERT INTO Employee (first_name, last_name, middle_name, email, date_of_birth, gender_id)
        VALUES (v_first_name, v_last_name, v_middle_name, v_email, v_date_of_birth, v_gender_id);
    END LOOP;
END $$;
COMMIT;
```
Генерация 50 000 компаний с указанием отрасли
```sql

DO $$
DECLARE
    i INT;
    random_year INT;
    random_industry TEXT;
    random_description TEXT;
BEGIN
    FOR i IN 1..50000 LOOP
        random_year := FLOOR(RANDOM() * (2020 - 1950 + 1)) + 1950;  -- Генерация случайного года от 1950 до 2020
        
        -- Генерация случайного описания от 500 до 2000 знаков
        random_description := SUBSTRING(md5(RANDOM()::text), 1, (FLOOR(RANDOM() * (1500)) + 500)::integer);
        
        -- Выбор случайной строки из списка отраслей
        random_industry := (
            SELECT industry
            FROM (VALUES 
                ('Промышленность'),
                ('Сельское хозяйство'),
                ('Лесное хозяйство'),
                ('Строительство'),
                ('Прочие виды деятельности сферы материального производства'),
                ('Обслуживание сельского хозяйства'),
                ('Связь'),
                ('Торговля и общественное питание'),
                ('Материально-техническое снабжение и сбыт'),
                ('Заготовки'),
                ('Информационно-вычислительное обслуживание'),
                ('Операции с недвижимым имуществом'),
                ('Общая коммерческая деятельность по обеспечению функционирования рынка'),
                ('Геология и разведка недр, геодезическая и гидрометеорологическая службы'),
                ('Жилищное хозяйство'),
                ('Коммунальное хозяйство'),
                ('Непроизводственные виды бытового обслуживания населения'),
                ('Здравоохранение, физическая культура и социальное обеспечение'),
                ('Народное образование'),
                ('Культура и искусство'),
                ('Наука и научное обслуживание'),
                ('Финансы, кредит, страхование, пенсионное обеспечение'),
                ('Управление'),
                ('Общественные объединения')
            ) AS industries(industry)
            ORDER BY RANDOM()
            LIMIT 1
        );

        -- Вставка записи в таблицу Company
        INSERT INTO Company (name, year_founded, description, industry)
        VALUES ('Компания ' || i, random_year, random_description, random_industry);
    END LOOP;
END $$;
```
Генерация должностей 
```sql 
DO $$
DECLARE
    i INT;
    random_responsibilities TEXT;
    random_length INT;
BEGIN
    FOR i IN 1..1000 LOOP
        -- Генерация случайной длины от 500 до 2000 знаков
        random_length := FLOOR(RANDOM() * (1500)) + 500;

        -- Генерация случайного текста для responsibilities
        random_responsibilities := REPEAT(CHR(FLOOR(65 + RANDOM() * 25)::INT), random_length);
        
        -- Вставка записи в таблицу employees.position
        INSERT INTO employees."position" (title, responsibilities)
        VALUES ('Должность ' || i, random_responsibilities);
    END LOOP;
END $$;
```

Генерация навыков, 200 навыков по 5 уравней на каждый (грейд)  
```sql 
DO $$
DECLARE
    i INT;
    skill_level INT;
BEGIN
    FOR i IN 1..200 LOOP
        FOR skill_level IN 1..5 LOOP
            -- Вставка записи в таблицу Skill
            INSERT INTO Skill (skill_name, skill_level)
            VALUES ('навык ' || i, skill_level);
        END LOOP;
    END LOOP;
END $$;
```
Генерация проектов 

```sql 
DO $$
DECLARE
    i INT;
    random_description TEXT;
    random_length INT;
BEGIN
    FOR i IN 1..500000 LOOP
        -- Генерация случайной длины для description от 500 до 2000 знаков
        random_length := FLOOR(RANDOM() * (1500)) + 500;

        -- Генерация случайного текста для description
        random_description := REPEAT(CHR(FLOOR(65 + RANDOM() * 25)::INT), random_length);

        -- Вставка записи в таблицу Project
        INSERT INTO Project (project_name, description)
        VALUES ('Проект ' || i, random_description);
    END LOOP;
END $$;
```
Генерация записей связывающий сотрудников с компаниями и позициями employee_Position
```sql 
DO $$
DECLARE
    i INT;
    employee_id INT;
    company_id INT; 
    position_id INT;
    start_date DATE;
    end_date DATE;
BEGIN
    FOR i IN 1..30000000 LOOP
        -- Генерация случайных значений для employee_id, company_id, position_id и дат
        employee_id := trunc(random() * 10000000 + 1); -- employee_id от 1 до 10000000
        company_id := trunc(random() * 50000 + 1); -- company_id от 1 до 50000
        position_id := trunc(random() * 1000 + 1); -- position_id от 1 до 1000

        -- Генерация случайной start_date от 1980 до 2022
        start_date := date '1980-01-01' + (trunc(random() * 15340))::int; -- 15340 дней = (2022-1980) * 365

        -- Генерация end_date: start_date + случайное количество лет от 1 до 3 лет
        end_date := start_date + (trunc(random() * 3 + 1) * interval '1 year')::interval;

        -- Вставка данных
        INSERT INTO employee_Position (employee_id, company_id, position_id, start_date, end_date)
        VALUES (employee_id, company_id, position_id, start_date, end_date);

        -- Для уменьшения нагрузки на систему делаем commit каждые 10000 записей
        IF i % 10000 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
END $$;
```
генерация employee_skill таблица связи между сотрудником и его навыками 

```sql
DO $$
DECLARE
    i INT;
    random_employee_id INT;
    random_skill_id INT;
BEGIN
    FOR i IN 1..3000000 LOOP
        -- Генерация случайного employee_id от 1 до 10000000
        random_employee_id := FLOOR(RANDOM() * 10000000) + 1;

        -- Генерация случайного skill_id от 1 до 1000
        random_skill_id := FLOOR(RANDOM() * 1000) + 1;

        -- Вставка записи в таблицу employees.employee_skill
        INSERT INTO employees.employee_skill (employee_id, skill_id)
        VALUES (random_employee_id, random_skill_id);

        -- Делать паузу на каждый 10 000 записей, чтобы не перегружать транзакцию
        IF i % 100000 = 0 THEN
            PERFORM pg_sleep(0.1); -- Неявная пауза для снижения нагрузки
        END IF;
    END LOOP;
END $$;
```
Генерация Employee_Project связующая таблица сотрудников и проектов 

```sql
DO $$
DECLARE
    i INT;
    random_employee_id INT;
    random_project_id INT;
    random_start_date DATE;
    random_end_date DATE;
    random_role TEXT;
    random_length INT;
BEGIN
    FOR i IN 1..20000000 LOOP
        -- Генерация случайного employee_id от 1 до 10000000
        random_employee_id := FLOOR(RANDOM() * 10000000) + 1;

        -- Генерация случайного project_id от 1 до 500000
        random_project_id := FLOOR(RANDOM() * 500000) + 1;

        -- Генерация случайной даты start_date от 1980-01-01 до 2022-12-31
        random_start_date := DATE '1980-01-01' + (FLOOR(RANDOM() * 15706))::INT;  -- 15706 = количество дней между 1980-01-01 и 2022-12-31

        -- Генерация end_date как start_date + от 1 до 5 лет
        random_end_date := random_start_date + (FLOOR(RANDOM() * (5 * 365)) + 365)::INT;  -- Минимум 1 год, максимум 5 лет

        -- Генерация случайной длины текста для role_in_project от 500 до 2000 знаков
        random_length := FLOOR(RANDOM() * (1500)) + 500;
        random_role := REPEAT(CHR(FLOOR(65 + RANDOM() * 25)::INT), random_length);

        -- Вставка записи в таблицу Employee_Project
        INSERT INTO Employee_Project (employee_id, project_id, start_date, end_date, role_in_project)
        VALUES (random_employee_id, random_project_id, random_start_date, random_end_date, random_role);
    END LOOP;
END $$;
```

3) Запросы для оптимизации 


```sql 
explain analyze SELECT s.skill_name, AVG(DATE_PART('year', AGE(e.date_of_birth))) AS avg_age
FROM employee e
JOIN employee_skill es ON e.id = es.employee_id
JOIN skill s ON es.skill_id = s.id
GROUP BY s.skill_name;
 
 -- запрос первый (Список работников с необходимыми скилами, работавшие над проектами сдатой начала позже 2020 года, в отрасли в отрослях) с фильрацией по возрасту 
explain analyze SELECT e.first_name, e.last_name, s.skill_name, s.skill_level
FROM Employee e
JOIN Employee_Skill es ON e.id = es.employee_id
JOIN Skill s ON es.skill_id = s.id
JOIN Employee_Position ep ON e.id = ep.employee_id
JOIN Company c ON ep.company_id = c.id
WHERE c.industry = 'Культура и искусство' and s.id in (1,3,5) and e.date_of_birth >= '1980-01-01'
AND e.id IN (
    SELECT employee_id
    FROM Employee_Project
    WHERE start_date >= '2020-01-01'
);


--запрос отобразит компании, в которых сотрудники с навыками уровня 5 работали на проектах, начавшихся с 1 января 2020 года.  
explain analyze SELECT c.name AS company_name, s.skill_name, COUNT(es.id) AS employee_count
FROM employee_skill es
JOIN employee e ON es.employee_id = e.id
JOIN employee_position ep ON e.id = ep.employee_id
JOIN company c ON ep.company_id = c.id
JOIN skill s ON es.skill_id = s.id
JOIN employee_project epr ON e.id = epr.employee_id
WHERE s.skill_level = 5
AND epr.start_date >= '2020-01-01' 
GROUP BY c.name, s.skill_name
ORDER BY employee_count DESC;


3й запрос 
--в скольки проектах и компаниях работал Сотрудник за последние 5 лет  
explain analyze select e.first_name, e.last_name,
       COUNT(DISTINCT c.id) AS company_count,     -- Количество уникальных компаний
       COUNT(DISTINCT epj.id) AS project_count    -- Количество уникальных проектов
FROM employee e
JOIN employee_project epj ON e.id = epj.employee_id
JOIN employee_position epo ON e.id = epo.employee_id
JOIN company c ON epo.company_id = c.id
WHERE epj.start_date >= NOW() - INTERVAL '5 years'
GROUP BY e.id;
```

4) Скриншоты анализа запроса 1 и 3 
ссыылка 1
ссылка 2 

5) Изменение изменение базы и добавление индексов 


Вынос отрасли в отдельную справочную таблицу и замена тестовых строк с text на варчар
```sql

CREATE TABLE industry (
    id serial4 NOT NULL,
    industry_name text NOT NULL
);

INSERT INTO industry (industry_name)
SELECT DISTINCT industry
FROM employees.company
WHERE industry IS NOT NULL;

ALTER TABLE employees.company
ADD COLUMN industry_id int4;


UPDATE employees.company c
SET industry_id = ir.id
FROM industry ir
WHERE c.industry = ir.industry_name;

ALTER TABLE employees.company
DROP COLUMN industry;

ALTER TABLE employees.company
    ALTER COLUMN "name" TYPE varchar(255),
    ALTER COLUMN description TYPE varchar(2000);
```
Замена Text на в varchar в других таблицах 
```sql
ALTER TABLE employees.employee
    ALTER COLUMN first_name TYPE varchar(255),
    ALTER COLUMN last_name TYPE varchar(255),
    ALTER COLUMN middle_name TYPE varchar(255),
    ALTER COLUMN email TYPE varchar(255);

ALTER TABLE employees.employees.project 
	ALTER COLUMN project_name TYPE varchar(255),
	ALTER COLUMN description TYPE varchar(2000)

ALTER TABLE employees.employees.skill 
	ALTER COLUMN skill_name TYPE varchar(255)
	
	
ALTER TABLE employees.employees."position" 
	ALTER COLUMN title TYPE varchar(255),	
	ALTER COLUMN responsibilities TYPE varchar(2000)

ALTER TABLE employees.employees.gender 
	ALTER COLUMN gender_name TYPE varchar(100)
```

Разбиение employees.project на партиции по годам 

```sql
DO $$ 
DECLARE 
    start_year int := 1980;
    end_year int := 2030;
BEGIN 
    FOR year IN start_year..end_year LOOP
        EXECUTE format('
            CREATE TABLE employees.employee_project_%s PARTITION OF employees.employee_project_new
            FOR VALUES FROM (''%s-01-01'') TO (''%s-01-01'');
        ', year, year, year + 1);
    END LOOP;
END $$;



DO $$ 
DECLARE 
    start_year int := 1980;
    end_year int := 2030;
BEGIN 
    FOR year IN start_year..end_year LOOP
        EXECUTE format('
            INSERT INTO employees.employee_project_%s
            SELECT * FROM employees.employee_project
            WHERE start_date >= ''%s-01-01'' AND start_date < ''%s-01-01'';
        ', year, year, year + 1);
    END LOOP;
END $$;


CREATE INDEX idx_employee_project_start_date_new
ON employees.employee_project_new (start_date);


CREATE INDEX idx_employee_project_start_date_new
ON employees.employee_project_new (start_date);


ALTER TABLE employees.employee_project_new
ADD CONSTRAINT employee_project_employee_id_fkey_new 
FOREIGN KEY (employee_id) REFERENCES employees.employee(id);

ALTER TABLE employees.employee_project_new
ADD CONSTRAINT employee_project_project_id_fkey_new 
FOREIGN KEY (project_id) REFERENCES employees.project(id);


ALTER TABLE employees.employee_project DROP CONSTRAINT employee_project_pkey;
DROP INDEX IF EXISTS employees.idx_employee_project_start_date;
ALTER TABLE employees.employee_project DROP CONSTRAINT employee_project_employee_id_fkey;
ALTER TABLE employees.employee_project DROP CONSTRAINT employee_project_project_id_fkey;

ALTER TABLE employees.employee_project RENAME TO employee_project_old;

ALTER TABLE employees.employee_project_new RENAME TO employee_project;

DROP TABLE employees.employee_project_old;

```

