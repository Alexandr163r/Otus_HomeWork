## Триггеры, поддержка заполнения витрин

1) Подготавливаем все необходимое, все есть в скрипте но для удобства продублирую, плюс добавил результаты селекта отчета.

```sql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);


INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);


SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

good_name               |sum         |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|

CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
2) наша витринная таблица good_sum_mart пустая, что не есть хорошо, мы же не хотим потерять текущее состояние продаж.

```sql 
SELECT * FROM good_sum_mart;
good_name|sum_sale|
---------+--------+
```
решаем данный вопрос инсертом
```sql 
INSERT INTO good_sum_mart (good_name, sum_sale)
SELECT G.good_name, SUM(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```
Проверяем 
```sql 
SELECT * FROM good_sum_mart;
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|
```
Вот теперь все хорошо, и данные соответствуют нашему отчету на текущей момент.

3) создаем функцию триггера

```sql 
CREATE OR REPLACE FUNCTION update_good_sum_mart() 
RETURNS trigger AS 
$$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE good_sum_mart
        SET sum_sale = sum_sale + (NEW.sales_qty * (SELECT good_price FROM goods WHERE goods_id = NEW.good_id))
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);

        IF NOT FOUND THEN
            INSERT INTO good_sum_mart (good_name, sum_sale)
            VALUES (
                (SELECT good_name FROM goods WHERE goods_id = NEW.good_id),
                NEW.sales_qty * (SELECT good_price FROM goods WHERE goods_id = NEW.good_id)
            );
        END IF;
   
    ELSIF TG_OP = 'UPDATE' THEN
    UPDATE good_sum_mart
    SET sum_sale = sum_sale - (OLD.sales_qty * (SELECT good_price FROM goods WHERE goods_id = OLD.good_id))
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

        UPDATE good_sum_mart
        SET sum_sale = sum_sale + (NEW.sales_qty * (SELECT good_price FROM goods WHERE goods_id = NEW.good_id))
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);

    ELSIF TG_OP = 'DELETE' THEN
        UPDATE good_sum_mart
        SET sum_sale = sum_sale - (OLD.sales_qty * (SELECT good_price FROM goods WHERE goods_id = OLD.good_id))
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
Как то странно отображение функции выглядит в формате md ну ладно, особенности синтексиса наверное (извините если что)

создаем триггер

```sql 
CREATE TRIGGER trg_update_good_sum_mart
AFTER INSERT OR UPDATE OR DELETE
ON sales
FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart();
```
4) Проверяем что вышло:
Вставка, и сразу добавим новый тавар, а то мало ли с новым не работало бы.
```sql
INSERT INTO goods (goods_id, good_name, good_price) 
VALUES (3, 'Пивасик', 75.00);

INSERT INTO sales (good_id, sales_qty) 
VALUES (3, 10);

SELECT * FROM good_sum_mart;

good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|
Пивасик                 |      750.00|
```
Апдейт 
```sql
UPDATE sales SET sales_qty = 15 WHERE sales_id = 3;
UPDATE sales SET sales_qty = 20 WHERE sales_id = 5;
SELECT * FROM good_sum_mart;
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       13.00|
Пивасик                 |     1500.00|
```
Превый апдейт был случайностью, я хотел больше пивасика но перепутал со спичками) 
Удаление 

```sql 
DELETE FROM sales WHERE sales_id = 3;
SELECT * FROM good_sum_mart;
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Пивасик                 |     1500.00|
Спички хозайственные    |        5.50|
```
Удалил лишние спички.
В целом вроде все работает.

5) что нам собственно это дает

- Распределение нагрузки  и производительность, но об этом уже говорилось, хотя нут наверное нужно уточнить что не только отчет делается быстрее но и в целом нагрузка для его получение размазывается равномерно в процессе работы с продажами. Сюда же наверное и скорость выполнения, селект по витрине займет минимальное количество времени. 
- Учет изменения цен, ибо могут поменять в любое время. а пока тянется отчет.
- Из первого пунка выходит еще один упрощение интеграции, ибо можно быстро запихнуть в джейсон по требованию клиента, и отправить куда то.

Наверное все, больше в голову ничего не приходит.
