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
 # 6 Напишите запрос, который для каждого дня посчитает с нарастающим итогом количество успешных уроков, количество уроков, которые прогуляли студенты, и общее количество уроков. Не нужно учитывать пробные уроки.

Сколько всего уроков было 13 января 2016 года?

 ````sql
 with classes_counts as 
(select date_trunc('day', class_start_datetime) as class_day,
       coalesce(class_status, 'NA') as class_status,
       count(id_class) as classes
from skyeng_db.classes
where class_type != 'trial'
group by 1, 2)
select distinct class_day,
       sum(case when class_status='success' then classes else 0 end) over(order by class_day rows between unbounded preceding and current row) as succesful_classes,
       sum(case when class_status='failed_by_student' then classes else 0 end) over(order by class_day rows between unbounded preceding and current row) as failed_by_student_classes,
       sum(classes) over(order by class_day rows between unbounded preceding and current row) as total_classes
from classes_counts
order by class_day
````
# 7 Напишите запрос, который позволит выбрать до 10 учителей из каждого департамента. 
 ````sql
 with data_with_id as
(
select id_teacher,
       department,
       max_teaching_level,
       city,
       country,
       row_number() over(partition by department order by id_teacher) as id_in_dept
from skyeng_db.teachers
where department is not null
)
select *
from data_with_id
where id_in_dept <= 10
order by department, id_in_dept
````
# 8. Посчитайте, сколько оплат успешно проходит каждый день. Вычислите скользящее среднее с окном 7, 31 день так, чтобы текущая строка была в середине диапазона.
 ````sql
 with daily_dinamics as
 	(select date_trunc ('day', transaction_datetime) as pay_day
		,count (distinct id_transaction) pay_count
	from skyeng_db.payments
	where status_name = 'success'
		and operation_name in ('Покупка уроков', 'Начисление корпоративному клиенту')
		and id_transaction is not null
	group by 1
	)
select pay_day, paay_count
	,avg (pay_count) over (order by pay_day rows between 3 preceding and 3 following) as ma_7
	,avg (pay_count) over (order by pay_day rows between 15 preceding and 15 following) as ma_31
from daily_dinamics
````
# 9. Подготовьте выборку, которая отвечает на вопросы:
- сколько оплат в месяц делают клиенты
- сколько времени проходит от оплпты до оплаты
- когда нужно начинать звонить студенту и "будить" его
 ````sql
 select user_id
 	,payment_amount
	,transaction_datetime
	,id_transaction
	,row_number () over (partition by user_id order by transaction_datetime) as номер_оплаты
	,row_number () over (partition by user_id
		,date_trunc ('month', transaction_datetime) 
		order by transaction_datetime rows between unbounded preceding and current row) as номер_оплаты_месяц
	,sum (payment_amount) over (partition by user_id, date_trunc ('month' , transaction_datetime)
		order by transaction_datetime rows between unbounded precedin and current row) as сумма_платежей
	,lead (transaction_datetime) over (partition by user_id order by transaction_datetime) as след_оплата
	,lag (transaction_datetime) over (partition by user_id order by transaction_datetime) as пред_оплата
from skyeng_db.payments
where status_name = 'success'
	and operation_name in ('Покупка уроков', 'Начисление корпоративному клиенту')
	and id_transaction is not null
order by user_id
````
		
	



