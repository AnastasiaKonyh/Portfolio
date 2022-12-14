# Разработка BI платформы для оперативного анализа состояния бизнеса 

## Цель проекта:

Построить дашборд для изучения ситуации в интерент магазине и розничной торговле

## Техническое задание:

Дашборд должен интерактивно отображать следующую информацию:
1. Общее: интеренет-магазин и розница:
- Выручка, количество транзакций, объем продаж, средний чек
- Также должны дать возможность выбирать год и месяц
- Для каждой метрики нужно показать: 
	- значение за выбранный период
	- значение за такой же период прошлого года
  - изменение по сравнению с прошлым годом в процентах
  - нарастающий итог с начала выбранного года
- Разбивка выручки по брендам
- Реализовать внутри каждого бренда названия топ-5 продуктов по выручке, сумму продаж по ним и тренд
2. Интернет-магазин:
- Выручка, затраты на маркетинг, количество сессий, среднее количество товаров в заказе, колличество уникальных покупателей, количество транзакций, маржа
- Возможность посмотреть на топ-5 медиумов по марже, а также по каждому медиуму:
  - Выручка, затраты и маржа
  - Доля в выручке
  - Ранг канала по затратам
- Построить графики скользящего среднего по выручке с трехмесячным окном и скользящей суммы с полугодовым окном
- Показать помесячную динамику количества новых покупателей.
3. Розница:
- Посмотреть на работу отдела продаж по иерархии (регион - руководитель - подчиненнные). В этой визуализации показать метрики:
  - выручка
  - средний чек
  - количество проданных уникальных брендов
  - топ-3 бренда аутсайдера
- Разбивка по товарам и категориям:
  - выручка
  - продажи в штуках
  - доля выручки от заказов, в которых покупается от 3-х штук одного товара
  
## Описание проекта:

Исходные данные представляют собой набор файлов, содержащих информацию о каналах, клиентах, менеджерах, товаре и транзакциях.

На этапе загрузки данных почистила таблицы от пустых значений, удалила дубликаты, объеденила таблицы в справочнике о товарах. Исправила типы данных где необходимо. Создала настраиваемый столбец. Создала таблицу дат. Создала вспомагательную ощдиночную таблицуи построила связи между таблицами.

Дашборд построила на 4 страницах, также на отдельных страницах реализованы подсказака и деталицация

Для визуализации дашборда использовала:
- Карточка
- Матрица
- Гистограмма с группировкой
- Линейчатая диаграмма с накоплением
- Срез
- График
- круговая диаграмма

## Инструменты проекта
```
Power BI
Tabular Editor
DAX
```
