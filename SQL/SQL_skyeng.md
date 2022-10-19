## Онлайн школа английского языка
### Описание данных:
В схеме skyeng_db существуют такие таблицы:

- **`students`** — данные о студентах (потенциальных, которые только оставили заявку, и тех, кто действительно оплатил обучение),
- **`orders`** — данные о заявках на обучение,
- **`classes`** — данные об уроках,
- **`payments`** — данные об оплате,
- **`teachers`** — данные об учителях.

**students**

- **`user_id`** — ****ID студента,
- **`student_sex`** — пол студента,
- **`geo_cluster`** — география студента,
- **`country_name`** — название страны,
- **`region_name`** — название региона,
- **`email_domain`** — домен почты студента.

**orders**

- **`user_id`** — ID студента,
- **`order_datetime`** — дата и время заявки,
- **`id_order`** — ID заявки,
- **`page_name`** — название страницы, с которой оставлена заявка.

**payments**

- **`user_id`** — ID студента,
- **`id_transaction`** — ID транзакции,
- **`operation_name`** — название операции,
- **`status_name`** — название статуса транзакции,
- **`classes`** — количество уроков в транзакции,
- **`payment_amount`** — сумма операции,
- **`transaction_datetime`** — дата и время операции.

**classes**

- **`user_id`** — ID студента,
- **`id_class`** — ID урока,
- **`class_start_datetime`** — дата и время начала урока,
- **`class_end_datetime`** — дата и время окончания урока,
- **`class_removed_datetime`** — дата и время удаления данных об уроке,
- **`id_teacher`** — ID преподавателя,
- **`class_status`** — статус урока,
- **`class_status_datetime`** — дата постановки статуса урока,
- **`class_type`** — тип урока.

**teachers**

- **`id_teacher` —** ID преподавателя,
- **`age` —** возраст преподавателя,
- **`city` —** город преподавателя,
- **`country` —** страна преподавателя,
- **`department` —** отдел преподавателя,
- **`max_teaching_level` —** максимальный дозволенный уровень студента,
- **`max_teaching_level_id` —** ID  максимального уровня студента,
- **`language_group` —** тип учителя по признаку родного языка.

# 1 KPI
Необходимо посчитать характеристики оплат в разрезе кластеров географии (geo_cluster) в 2016 году.
Нас интересуют операции типа “Покупка уроков” или “Начисление корпоративному клиенту” со статусом платежа = success (успешная транзакция).

````sql
select students.geo_cluster,
       count(distinct id_transaction) as payments,
       count(distinct payments.user_id) as students,
       count(distinct id_transaction):: float 
/ count(distinct payments.user_id) as payments_per_student,
       sum(payment_amount) as total_amount,
       avg(payment_amount) as avg_amount,
       sum(classes) as classes_purchased,
       sum(payment_amount) / sum(classes) as avg_class_price,
       avg(classes) as avg_classes_in_purchase
from skyeng_db.payments
join skyeng_db.students
	on students.user_id = payments.user_id
where operation_name in ('Покупка уроков','Начисление корпоративному клиенту')   
           and date_trunc('year',transaction_datetime) = '2016-01-01'
           and status_name = 'success'
group by students.geo_cluster
order by total_amount desc
````
# 2
Отдел контроля качества хочет узнать, есть ли преподаватели, нуждающиеся в дополнительном обучении. Вас просят помочь найти таких преподавателей и студентов, чтобы попытаться выяснить причины, побудившие ученика уйти. На обсуждении было решено, что нужно выгрузить всех учителей, которые обучали студентов, внёсших только одну оплату.
````sql
select c.user_id,
       c.id_teacher,
       t.department,
       t.age,
       t.city,
       t.country,
       t.max_teaching_level,
       count(distinct id_class) as classes_count
from skyeng_db.payments
join skyeng_db.classes c 
	on c.user_id  = payments.user_id
join skyeng_db.teachers t 
	on t.id_teacher  = c.id_teacher 
where operation_name in ('Покупка уроков','Начисление корпоративному клиенту')
      and status_name = 'success'
      and c.class_type != 'trial'
      and c.class_status = 'success'
group by c.user_id,
       c.id_teacher,
       t.department,
       t.age,
       t.city,
       t.country,
       t.max_teaching_level
having count(distinct id_transaction) = 1 and count(distinct id_class)>=4
limit(1000)
````
# 3
Найдите тех учеников, которые проходили уроки в 2016 году, но ни разу не оплатили обучение. Выведите таких учеников и количество пройденных ими уроков. Для самопроверки добавьте поле с количеством оплат.
````sql
select c.user_id,
       count(distinct id_transaction) as payments,
       count(distinct id_class) as classes
from skyeng_db.classes c
left join skyeng_db.payments 
	on c.user_id  = payments.user_id
	and operation_name in ('Покупка уроков'
                       ,'Начисление корпоративному клиенту')
            and status_name = 'success'
where c.class_type != 'trial'
      and c.class_status = 'success'
      and payments.user_id is null
      and date_part('year', c.class_start_datetime) = 2016 
group by c.user_id
````
# 4. Распределение нагрузки
Найдите преподавателя, который провел максимальное количество уроков в 2016 году. 

Интересуют только уроки, которые были списаны с баланса, то есть успешно пройдены (`success`) или прогуляны (`failed_by_student`) студентом. Не включайте в выборку триальные уроки.
````sql
with t_lessons as
  (select id_teacher
    ,count (id_class) count_classes
   from skyeng_db.classes
   where date_trunc('years', class_start_datetime) = '2016.01.01'
    and class_status in ('success', 'failed_by_student')
    and class_type != 'trial'
   group by id_teacher
   )
 select t.*
 from skyeng_db.teachers t
 join t_lessons on t_lessons.id_teacher = t.id_teacher
 order by count_classes desc
 limit 1
 ````
 # 5. Планирование нагрузки
 Напишите запрос, который покажет распределение учителей по количеству проводимых в месяц уроков (опять же - не триальных, а только успешных или прогулянных самим студентом).
 
 ````sql
 with t_m as
  (select id_teacher as учителя
    , date_part ('month', class_start_datetime) as месяц
    , count (id_class) as уроки
  from skyeng_db.classes
  where date_trunc ('years', class_start_datetime) = '2016.01.01'
    and class_status in ('success', 'failed_by_student')
    and class_type != 'trial'
   group by учителя, месяц
   )
 select уроки
  ,count(учителя)
 from t_m
 group by 1
 ````

