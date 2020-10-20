# moove_admission_test
Admission test for MOOVE program provided by Skolkovo (Moscow School of Management) and MTS

Предложенный датасет нормализован до 3й нормальной формы. Данные разложены из допущения, что первичным ключом датасета является email (он не может быть идентичным у нескольких сотрудников в одной организации), при этом row_id (первый безымянный столбец датасета) отражает хронологию изменений/добавлений данных каждого из сотрудников: предполагается, что  в один момент времени сотрудник может занимать 1 должность в 1м подразделении и сидеть в 1й комнате, т.е. row_id отражает лог изменений. При этом в один момент времени сотрудник может иметь несколько контактных телефонов. Следовательно, рабочие таблицы заполнены актуальными данными для каждого из сотрудников

Тест-кейсы по заданию

--1) получение всех имен и фамилий людей в определенном кабинете
select first_name, last_name
from employees e 
join depts d on d.employee_id =e.employee_id 
where room ='500';

--2) добавление нового телефона сотруднику
insert into contacts (employee_id, phone)
values (1,11111111111);

/*
delete from contacts 
where phone_id =167982

select *
from contacts c 
where phone_id =167981
*/

-- 3) получение списка телефонов всех сотрудников определенной должности в одном из отделов
select phone
from contacts c 
join depts d on d.employee_id =c.employee_id and d.dept ='IT' and d.post ='Intern'

-- 4) добавление нового сотрудника
insert into employees
values ('Test','Testov','test@example.ru')

/*
delete from employees e 
where first_name ='Test' and last_name = 'Testov'

select *
from employees e 
where first_name ='Test' and last_name = 'Testov'
*/

-- 5) перемещение сотрудника в другой отдел со сменой комнаты
update depts
set dept='Sales', room = '7000'
where employee_id =1

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Скрипты создания рабочих таблиц

-- создаем таблицу для заливки сырых данных через pgAdmin
create TABLE rawdata
(
    row_id SERIAL PRIMARY KEY,
    dept varchar (15),
    email varchar (40),
    first_name varchar (20),
    last_name varchar (20),
    phone numeric (11,0),
    post varchar (10),
    room int2
 );

--заливаем csv через pgAdmin

/*
Презюмируем, что первичным ключом датасета является email (он не может быть идентичным у нескольких сотрудников в одной организации),
при этом row_id (первый безымянный столбец датасета) отражает хронологию изменений данных каждого из сотрудников.
В таком случае, нам имеет смысл заполнить рабочие таблицы актуальными данными для каждого из сотрудников
*/

-- создаем промежуточную таблицу с указанием, какая строка данных является наиболее актуальной для каждого из сотрудников
create table rawdata2 as
(
select *, row_number () over (partition by email order by row_id asc) as versia
from rawdata
);

--создаем таблицу уникальных сотрудников
create table employees as
(
select distinct first_name, last_name, email
from rawdata2 r2
);
alter table employees 
add employee_id serial primary key;

-- создаем таблицу актуальных должностей и рабочих мест сотрудника
create table depts as
(
select e.employee_id, r.dept, r.post, r.room 
from rawdata2 r 
join (select email, max(versia) as max_versia
	  from rawdata2
	  group by email) r2 on r2.email=r.email and r2.max_versia=r.versia 
join employees e on e.email =r.email
);

-- создаем таблицу телефонов сотрудников (т.к. их может быть несколько, учитываем не только актуальные)
create table contacts
(
phone_id serial primary key,
employee_id serial references employees (employee_id),
phone numeric (11,0)
);
insert into contacts (employee_id, phone)
select e.employee_id, r.phone from rawdata2 r join employees e on e.email =r.email;


