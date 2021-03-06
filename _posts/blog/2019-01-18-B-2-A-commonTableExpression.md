---
title: MySQL 8.0 - Common Table Expression
author: min_cho
created: 2019/01/18
modified:
layout: post
tags: mysql mysql8
image:
  feature: mysql.png
categories: MySQL8
toc: true
toc_label: "My Table of Contents"
toc_icon: "cog"
---

------

## 개요
```
* MySQL 8.0 에서, 드디어 Common Table Expression 을 지원하게 되었다.

* 단순히 WITH 구문을 통해 Subquery로서 작동하는것 외에, Common Table Expression 의 "Recusive" 구문을 이용해 Hierarchy 결과 (ORACLE의 connect by 구문) 를 이용할 수 있다.
```

------

## 예제
### 데이터생성

```sql

DROP TABLE IF EXISTS emp;

CREATE TABLE emp (id int, name varchar(100), birth_year int, manager_id int, joined_year year);
INSERT INTO emp values (1, 'Matheus', 1975, null, 2016);
INSERT INTO emp values (2, 'Cindy', 1978, 1, 2016);
INSERT INTO emp values (3, 'Tim', 1978, 2, 2016);
INSERT INTO emp values (4, 'Chan', 1983, 3, 2016);
INSERT INTO emp values (5, 'Min', 1979, 3, 2018);
INSERT INTO emp values (6, 'Sam', 1976, 2, 2017);
INSERT INTO emp values (7, 'David', 1983, 6, 2016);
INSERT INTO emp values (8, 'Siena', 1985,1, 2016);
INSERT INTO emp values (9, 'Ted', 1980, 6, 2016);
INSERT INTO emp values (10, 'Stray', 1990, 6, 2016);       


+------+---------+------------+------------+-------------+
| id   | name    | birth_year | manager_id | joined_year |
+------+---------+------------+------------+-------------+
|    1 | Matheus |       1975 |       NULL |        2016 |
|    2 | Cindy   |       1978 |          1 |        2016 |
|    3 | Tim     |       1978 |          2 |        2016 |
|    4 | Chan    |       1983 |          3 |        2016 |
|    5 | Min     |       1979 |          3 |        2018 |
|    6 | Sam     |       1976 |          2 |        2017 |
|    7 | David   |       1983 |          6 |        2016 |
|    8 | Siena   |       1985 |          1 |        2016 |
|    9 | Ted     |       1980 |          6 |        2016 |
|   10 | Stray   |       1990 |          6 |        2016 |
+------+---------+------------+------------+-------------+

```

### 일반적인 사용
 * 일반적으로는 FROM 절에 나오는 sub-query 의 result set (Inline View)을 만들어내는 용도로 사용된다. 장점으로는 가독성이 용이하다.

```sql

---- 사원중 가장 나이가 많은 사람과 적은 사람은 누구인가?
SELECT name, birth_year from emp c1, (
   select
      min(birth_year) as min_b
      ,max(birth_year) as max_b
   from emp
   where joined_year=2016
) x
where c1.birth_year in (x.min_b , x.max_b);


---- 위와 같은 쿼리는 CTE를 사용하면 아래와 같이 나타낼 수 있다.
WITH cte (min_birth_year, max_birth_year) AS
(
   select
      min(birth_year)
      ,max(birth_year)
   from emp
   where joined_year=2016
)
SELECT name, birth_year from emp c1, cte c2
WHERE c1.birth_year in(c2.min_birth_year, c2.max_birth_year);


+---------+------------+
| name    | birth_year |
+---------+------------+
| Matheus |       1975 |
| Stray   |       1990 |
+---------+------------+
2 rows in set (0.00 sec)


---- CTE 는 하나의 쿼리에서 여러개를 사용할 수 있다.
WITH
 cte1 (birth_year) AS
   ( select min(birth_year) from emp where joined_year=2016 ),
 cte2 (birth_year) AS
   ( select max(birth_year) from emp where joined_year=2016 )
SELECT name, c.birth_year from emp c, cte1 c1, cte2 c2
WHERE c.birth_year in(c1.birth_year, c2.birth_year);

```

 * 위의 목적을 제외하고, 한 쿼리내에서 자주사용될 여지가 있는 Result set을 미리 만들어 두는것도 가능하다.

```sql

---- 1980년 1월 1일 이후 출생자들중, 연도별로 입사한 사람들과, 같은 메니져를 같는 사람 숫자는 몇명인가?
---- 기존쿼리
SELECT 'Joined Year', joined_year as val, count(*) from emp where  birth_year > 1980 group by joined_year
UNION ALL
SELECT 'Manager Name', b.name as val, count(*) from emp a INNER JOIN emp b on a.manager_id=b.id where a.birth_year > 1980 group by b.name;



---- 공통된 부분을 CTE로 만들어 낼 수 있다. (1980년생을 뽑아 먼저 CTE로 만들어 둔다.)
WITH cte (manager_id, joined_year) AS
(
   select
      manager_id
      ,joined_year
   from emp
   where birth_year > 1980
)
SELECT 'Joined Year', joined_year as val, count(*) from cte group by joined_year
UNION ALL
SELECT 'Manager Name', b.name as val, count(*) from cte a INNER JOIN emp b on a.manager_id=b.id group by b.name;

+--------------+---------+----------+
| Joined Year  | val     | count(*) |
+--------------+---------+----------+
| Joined Year  | 2016    |        4 |
| Manager Name | Matheus |        1 |
| Manager Name | Tim     |        1 |
| Manager Name | Sam     |        2 |
+--------------+---------+----------+

```


### RECURSIVE의 활용
 * MySQL을 사용하며 가장 아쉬웠던 부분이 바로, Hierarchy (계층형) 쿼리가 안된다는 점이었다.  MySQL New feature 설문조사 중  Community에서도 가장 원하는 기능이기도 했다.
 * 5.7까지는 이를 구현하기 위해, 동적 쿼리를 사용하거나, Depth의 한계를 정해둔 후 해당 Depth 만큼 Table 에 column을 만들어 구현하는 방식등을 사용하였다.
 * 사용법은 아래와 같다.

```sql

WITH RECURSIVE cte (n) AS
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 3
)
SELECT * FROM cte;


+------+
| n    |
+------+
|    1 |
|    2 |
|    3 |
+------+

```

  1. WITH 절 다음  RECURSIVE 구문을 넣는다.
  2. 첫번째 SELECT 구문을 이용해 초기값을 설정한다.
  3. 초기값을 설정한값은 Common Table에 들어가고, 해당 TABLE 을 이용해 UNION ALL 다음구문을 실행한다.
  4. 이는 RECURSIVE 구문에 의해 반복적으로  UNION ALL 다음구문 실행된 후,  Common Table 에 더해진다.
  5. WHERE 조건이 FALSE 상태가 되거나, 결과가 존재하지 않은경우 해당 구문은 멈추게 된다.

```sql

---- CTE와 RECURSIVE를 이용하여, 조직도를 확인해보자

WITH RECURSIVE emp_paths (level, id, name, path) AS
(
  SELECT 1 as level , id, name , name
    FROM emp
    WHERE  manager_id IS NULL
  UNION ALL
  SELECT ep.level + 1 level ,
         e.id, concat(LPAD('', 2*(ep.level) ,' ' ),'ㄴ' ,e.name) as name ,
         CONCAT(ep.path, '->', e.name)
    FROM emp_paths AS ep JOIN emp AS e
      ON ep.id = e.manager_id
)
SELECT * FROM emp_paths ORDER BY path;


+-------+------+----------------+----------------------------+
| level | id   | name           | path                       |
+-------+------+----------------+----------------------------+
|     1 |    1 | Matheus        | Matheus                    |
|     2 |    2 |   ㄴCindy      | Matheus->Cindy             |
|     3 |    6 |     ㄴSam      | Matheus->Cindy->Sam        |
|     4 |    7 |       ㄴDavid  | Matheus->Cindy->Sam->David |
|     4 |   10 |       ㄴStray  | Matheus->Cindy->Sam->Stray |
|     4 |    9 |       ㄴTed    | Matheus->Cindy->Sam->Ted   |
|     3 |    3 |     ㄴTim      | Matheus->Cindy->Tim        |
|     4 |    4 |       ㄴChan   | Matheus->Cindy->Tim->Chan  |
|     4 |    5 |       ㄴMin    | Matheus->Cindy->Tim->Min   |
|     2 |    8 |   ㄴSiena      | Matheus->Siena             |
+-------+------+----------------+----------------------------+




---- 사번이 5번에 대한 메니져 계층을 확인해보자.


WITH RECURSIVE emp_path_for_user (path, manager_id) as
(
  select name,manager_id
  from emp where id=5
  UNION ALL
  select concat(epfu.path,'->',e.name), e.manager_id
  from emp_path_for_user epfu, emp e
  where epfu.manager_id=e.id
)
SELECT PATH FROM emp_path_for_user where manager_id is NULL;


+--------------------------+
| PATH                     |
+--------------------------+
| Min->Tim->Cindy->Matheus |
+--------------------------+

```
