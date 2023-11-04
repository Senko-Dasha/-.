Обо мне:
Привет, меня зовут Дарья, я начинающий аналитик данных. 
Прохожу обучение от Skypro по специальности "Аналитик даных". В этом репозитории вы сможете найти некоторые из моих проектов, выполненных во время обучения и практики.

Навыки и умения:
Работаю в Excel: сводные таблицы, формулы включая ВПР, Power Query, Power Pivot, диаграммы, статистические методы анализа данных, распределения и т.д.
- Обладаю знаниями и опытом написания сложных SQL-запросов для анализа данных, выгрузка данных из СУБД (PostgreSql): оконные функции, агрегация и группировка данных, очистка «сырых» данных и т.д.
- Имею знания в области А/Б тестировании
- Знаю теорию и основы анализа данных: теории вероятностей и математической статистики, непрерывные и дискретные распределения, проверка гипотез, построение доверительных интервалов.
- Имею навыки проведения анализа продаж, прибыли, маржинальности, ассортимента и анализ динамики различных групп клиентов по коммерческим показателям (revenue, маржа, коммерческая прибыль, затраты). Провожу винтажный и когортный анализы.


Проекты:
Проект №1: Калькулятор юнит-экономики онлайн-кинотеатра

Что необходимо было сделать:

1. "Нашему отделу аналитики необходимо посчитать юнит-экономику продукта и предложить сценарий по настройке параметров для выхода на 25-процентную маржинальность (это всё пойдет на слайды презентации для стратсессии в конце квартала) и собрать хорошую наглядную визуализацию, где будет показано, кто, где и в каком объеме смотрит фильмы на нашей платформе."
2. "Просьба собрать калькулятор юнит-экономики нашего продукта, поскольку сейчас очень не хватает такой автоматизированной системы для быстрого принятия решений."
Как решал:

1. Для начала определить, что является юнитом.

2. Посчитать юнит-экономику продукта и предложить сценарий по настройке параметров для выхода на 25%-ную маржинальность.

3. Выбрать оптимальный вариант расчета Retention.

4. Собрать визуализации основных бизнес-показателей.

5. Исследовать данные о пользователях и их поведении.

6. ссылка на проект: https://docs.google.com/spreadsheets/d/1cVb45TvFf9a4mzppRXClR_lSyDuz-GCdCk-hbgH3v5Q/edit?usp=sharing

7. Проект №2: Наша задача — смоделировать изменение балансов студентов.
8. Задание 1:

Посмотрите на изменения балансов студентов (на примере топ-1000 строк), собранных из CTE. 

Какие данные вас смущают? Какие вопросы стоит задавать дата-инженерам и владельцам таблиц? 

Задание 2:

Создайте визуализацию  итогового результата. 

Какие выводы можно сделать из получившейся визуализации?

Решение:
with    first_payment as 
        (select   user_id
                , min(transaction_datetime::date) date_first_payment
        from skyeng_db.payments
        where status_name = 'success' 
        group by user_id
        ),
all_dates as 
        (select distinct(class_start_datetime::date) as dt
         from skyeng_db.classes
            where class_start_datetime::date between '2016-01-01' and '2016-12-31'
        ),
all_dates_by_user as
        (select   user_id
                , dt
            from  first_payment
             join all_dates 
                on all_dates.dt >=date_first_payment::date
        ),
payments_by_dates as
        (select   user_id
                , transaction_datetime::date as payment_date
                , sum(classes) transaction_balance_change
            from skyeng_db.payments 
            where status_name = 'success' 
            group by  user_id
                    , transaction_datetime::date
            ),
payments_by_dates_cumsum as
        (select    a.user_id
                , a.dt
                , transaction_balance_change
                , sum (coalesce(transaction_balance_change, 0)) over (partition by a.user_id order by a.dt) as transaction_balance_change_cs
        from all_dates_by_user a  
           left join payments_by_dates b
                on a.user_id = b.user_id
                and a.dt = b.payment_date),  
classes_by_dates as 
        (select   user_id
                , class_start_datetime::date as class_date 
                , count (id_class)* -1 as classes
        from skyeng_db.classes
        where class_status in ('success' , 'failed_by_student')
            and class_type != 'trial'
        group by user_id
                ,class_date),
classes_by_dates_dates_cumsum as
            (select   a.user_id
                    , a.dt
                    , classes
                    , sum (coalesce(classes, 0)) over (partition by a.user_id order by a.dt) as classes_cs
            from all_dates_by_user a  
              left join classes_by_dates c
                    on a.user_id = c.user_id
                    and a.dt = c.class_date),
balances as
        (select  p.user_id
                ,p.dt
                ,transaction_balance_change
                ,transaction_balance_change_cs
                ,classes
                ,classes_cs
                ,(classes_cs + transaction_balance_change_cs) as balance
        from payments_by_dates_cumsum p
            join classes_by_dates_dates_cumsum cl
            on p.user_id = cl.user_id
            and p.dt = cl.dt)
select dt
        , sum (transaction_balance_change)  transaction_balance_change
        , sum (transaction_balance_change_cs)  transaction_balance_change_cs
        , sum (classes)  classes
        , sum (classes_cs)  classes_cs
        , sum (balance)  balances
from balances
group by dt
order by dt
