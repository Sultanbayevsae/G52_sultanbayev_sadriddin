# G52_sultanbayev_sadriddin
```sql
create table roles(
id serial primary key ,
roleName varchar(100) unique not null generated always as('STUDENT''MENTOR') stored
);
```

```sql
create table users(
id serial primary key,
fullName varchar(100) not null,
phone varchar(100) unique not null,
role_id int,
constraint role_constraint  foreign key (role_id) references roles(id) on delete set null
);
```

```sql
create table groups(
id serial primary key,
groupName varchar(50),
mentor_id int,
createdAt date not null default now(),
constraint group_constraint foreign key(mentor_id)references users on delete set null
);

```
```sql
create table students(
id serial primary key,
user_id int,
group_id int not null,
createdAt date not null default now(),
active boolean not null default false,
foreign key (user_id)references users(id) on delete cascade,
foreign key (group_id)references groups(id) on delete cascade
);

```
```sql
CREATE OR REPLACE VIEW for_10_students AS
SELECT s.id        AS student_id,
u.fullName  AS student_fullName,
s.createdAt AS student_createdAt,
g.groupName,
m.fullName  AS mentor_fullName
FROM students s
JOIN users u ON s.user_id = u.id
JOIN groups g ON s.group_id = g.id
LEFT JOIN users m ON g.mentor_id = m.id
ORDER BY s.createdAt DESC
LIMIT 10;

SELECT * FROM for_10_students;
```


```sql
CREATE MATERIALIZED VIEW group_by_student_find AS
SELECT g.id        AS group_id,
g.groupName,
g.createdAt AS group_createdAt,
COUNT(s.id) AS studentCount
FROM groups g
LEFT JOIN students s ON g.id = s.group_id
GROUP BY g.id, g.groupName, g.createdAt
ORDER BY studentCount DESC
LIMIT 3;
REFRESH MATERIALIZED VIEW group_by_student_find;
```


```sql
CREATE OR REPLACE FUNCTION groupOfMentor(p_mentor_id INT)
RETURNS TABLE
(
mentor_id       INT,
mentor_fullName VARCHAR,
groupName       VARCHAR,
group_createdAt DATE,
studentCount    INT
)
AS
$$
BEGIN
RETURN QUERY
SELECT u.id        AS mentor_id,
u.fullName  AS mentor_fullName,
g.groupName,
g.createdAt AS group_createdAt,
COUNT(s.id) AS studentCount
FROM groups g          JOIN users u ON g.mentor_id = u.id
LEFT JOIN students s ON g.id = s.group_id
WHERE g.mentor_id = p_mentor_id
GROUP BY u.id, u.fullName, g.groupName, g.createdAt;
END;

    $$ LANGUAGE plpgsql;
```


```sql
CREATE OR REPLACE PROCEDURE Activationstudent(p_group_id INT)
LANGUAGE plpgsql
AS
$$
BEGIN
UPDATE students
SET active = TRUE
WHERE group_id = p_group_id
AND active = FALSE;

    RAISE NOTICE 'Students have been activated: group_id = %', p_group_id;
END;
$$;
CALL Activationstudent(1);
```
```sql

CREATE TABLE change_logs (
id SERIAL PRIMARY KEY,
table_name VARCHAR NOT NULL,
column_name VARCHAR NOT NULL,
row_id INT NOT NULL,
old_value TEXT,
new_value TEXT,
changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION update_change()
RETURNS TRIGGER AS
$$
DECLARE
col     TEXT;
old_val TEXT;
new_val TEXT;
BEGIN
FOR col IN
SELECT column_name
FROM information_schema.columns
WHERE table_name = TG_TABLE_NAME
LOOP
EXECUTE format('SELECT ($1).%I::TEXT', col) INTO old_val USING OLD;
EXECUTE format('SELECT ($1).%I::TEXT', col) INTO new_val USING NEW;

            IF old_val IS DISTINCT FROM new_val THEN
                INSERT INTO change_logs (table_name, column_name, row_id, old_value, new_value)
                VALUES (TG_TABLE_NAME, col, OLD.id, old_val, new_val);
            END IF;
        END LOOP;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_students_update_log
AFTER UPDATE
ON students
FOR EACH ROW
EXECUTE FUNCTION update_change();

```

--


# Database savollar


***
## 1. Database nima?
Database (Ma'lumotlar bazasi) - Ma'lum turdagi obyektlarga tegishli bo'lgan ma'lummotlarning tizimli va tartibli
to'plami hisoblanadi. Bu ma'lumotlar to'plamida har xil ma'lumot turlari (datatype) ga kiruvchi ma'lumotlarni, masalan
raqam, matn, rasm va ko'plab boshqa turdagi ma'lumotlar saqlanadi.
***
## 2. RDBMS nima?
RDBMS - (Relational Database Management System) Relatsion ma'lumotlar bazalari, masalan PostgreSQL, OracleDB, MariaDB, dBASE,
Microsoft SQL kabi bazalarning ma'lumotlari ustida turli xil amaliyotlar bajarish, jumladan baza yaratish, jadval yaratish,
ma'lumotlarni qo'shish, ma'lum shartlarga asoslanib ma'lumotlar bazasidan ma'lumot olish kabi ishlarni amaliyotlarni bajarish
uchun ishlab chiqarilgan dasturiy ta'minot hisoblanadi. RDBMS turidagi ma'lumotlar bazasi odatda ma'lumotlarni jadval ko'rinishida
saqlaydi.
*** 
## 3. DBMS and RDBMS farqi
- DBMS - o'z nomi bilan, to'liq ma'lumotlar oqimini boshqarishda ishlatilinadigan boshqaruv tizimi hisoblanib, u orqali
  malumot saqlanishi, olinishi va bu jarayonlarning qanchalik tez yoki sekin bolishini o'z zimmasiga oluvchi dasturiy taminot.
- RDBMS - yani Relational DBMS xuddi DBMS ga o'xshash xususiyatlarga ega, lekin ma'lumotlar o'rtasida bog'liqliklar yaratish
- imkonini beruvchi ma'lumotlar bazasi boshqarish tizimi hisoblanadi. Unda DBMS dan farqli o'laroq, ikki jadvalda malum
  ustunlar o'rtasida `FOREIGN KEY` lar va maxsus cheklovlar orqali bog'liqlik yaratish mumkin.
***
## 4. What's the SQL? Why is the SQL used for?
SQL - Structured Query Language bu ma'lumotlar bazasida ishlash, ma'lumotlarni samarali/tartibli boshqarish
uchun yaratilgan dasturlash tili hisoblanadi. 1986 - yilda IBM kompaniyasi tomonidan ishlab chiqilgan, RDBMS turidagi
ma'lumotlar bazasi bilan ishlash uchun maxsus yaratilgan buyruqlar to'plami.
*** 
## 5. `PRIMARY KEY` va `FOREIGN KEY` farqi nimada?
`Primary Key` — bu jadvaldagi har bir qatorni unikal tarzda aniqlovchi ustun yoki ustunlar to‘plamidir. U NULL bo‘lishi mumkin emas va har bir qiymat yagona bo‘lishi shart.

`Foreign Key` — bu boshqa jadvaldagi Primary Key yoki unique ustunga ishora qiluvchi kalit. Jadval orasidagi bog‘liqlikni (relationship) tashkil etadi.

- Farqlari

Primary key — o‘z jadvalidagi asosiy identifikator.

Foreign key — boshqa jadval bilan bog‘liqlikni ifodalaydi.

Primary key NULL qabul qilmaydi, Foreign key esa NULL bo‘lishi mumkin (agar optional relationship bo‘lsa).

***
## 6. PL/pgSQL nima?
`PL/pgSQL` (Procedural Language/PostgreSQL Structured Query Language) — bu PostgreSQL uchun mo‘ljallangan procedural dasturlash tili. U orqali siz:

Funktsiyalar (FUNCTION)

Protseduralar (PROCEDURE)

Trigger funksiyalari (TRIGGER FUNCTION)

Boshqaruv strukturalari (IF, LOOP, FOR, WHILE)

yaratishingiz mumkin. PL/pgSQL SQL’ning imkoniyatlarini kengaytiradi va murakkab biznes logiclarini server darajasida amalga oshirish imkonini beradi.
***

## 7. Assert statement nima va nima uchun ishlatilinadi?
`ASSERT` statement — bu dasturda kutilgan shart bajarilganini tekshirish uchun ishlatiladi. Agar shart bajarilmasa, xatolik yuz beradi.
Bu asosan test qilish jarayonlarida ishlatiladi. Bu orqali database da noto'g'ri amaliyot bo'lmasligini tekshirish mumkin.
Asosiy maqsad - xatolik yuzaga kelishi mumkin bo'lgan holatlarda oldindan tekshirib kamchiliklarni oldini olish!
***
## 8. Trigger nima?
`Trigger`— bu ma’lumotlar bazasida ma’lum bir voqea sodir bo‘lganda avtomatik ravishda ishga tushadigan kod blokidir.
Masalan, `INSERT`, `UPDATE`, `DELETE` amaliyoti yuz berganda trigger orqali log yozish, validatsiya qilish kabi ishlarni avtomatik tarzda amalga oshirib
natijalarni boshqa bir jadvalda saqlash uchun imkon yaratadi.
***
## 9.Transaction va ACID prinsiplari

Transaction (tranzaksiya) — bu ma’lumotlar bazasida bajariladigan bir yoki bir nechta amallar to‘plami bo‘lib, ular to‘liq muvaffaqiyatli yoki umuman bajarilmasligi kerak.

ACID bu tranzaksiyalarning ishonchli ishlashini ta’minlovchi to‘rtta asosiy tamoyildir:

- A – Atomicity: Tranzaksiya to‘liq bajariladi yoki umuman bajarilmaydi.

- C – Consistency: Tranzaksiya ma’lumotlar bazasini bir holatdan boshqa yaroqli holatga o‘tkazadi.

- I – Isolation: Parallel tranzaksiyalar bir biriga ta’sir qilmaydi.

- D – Durability: Tranzaksiya yakunlangach, uning natijalari doimiy saqlanadi (hatto server ishdan chiqsa ham).
***
## 10. Indexing va uning vazifasi?
`Index` — bu ma’lumotlar bazasida tezkor qidiruv yani seach va saralash imkonini beruvchi tuzilma. Indeks yordamida SELECT, JOIN, WHERE kabi so‘rovlar ancha tezroq bajariladi.

Vazifalari:

- Ma’lumotlar ustida tezroq qidiruv.

- Unique qiymatlarni ta’minlash .

- Ma’lumotlarga tezkor kirish.

- WHERE shartlari, ORDER BY va JOIN ishlash tezligini oshirish.

Lekin indekslar har doim foydali emas — ular INSERT, UPDATE, DELETE operatsiyalarni sekinlashtirishi mumkin, chunki indekslar ham yangilanadi. Shuning uchun indekslar balanslangan holda ishlatiladi.
