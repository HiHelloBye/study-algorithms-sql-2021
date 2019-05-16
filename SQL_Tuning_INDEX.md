# SQL Trace

trace 
```
alter session set events '10046 trace name context forever, level 12';
```

trace 종료
```
alter session set sql_trace = false;
```


### 힌트를 줘서 강제 nested loop join 실행하도록
 /*+ use_nl(A) */ 의미: 조인 시 nested loop 조인을 하라는 의미, 테이블을 지정할 수 있음
뒤에 myemp A 라고 지정했기 때문에 A테이블을 nested loop join 하라는 의미이다.
```
explain plan for
select /*+ use_nl(A, B) */ ename, dname
from myemp A
left outer join dept B on A.deptno=B.deptno;

select * from table(dbms_xplan.display);
```

### 전체 데이터가 join 되어야 할 경우 hash join 이용

```
explain plan for
select ename, dname
from myemp A
left outer join dept B on A.deptno=B.deptno;

select * from table(dbms_xplan.display);
```

### cf. hash join? nested loop join?
온라인에서 사용자가 페이징 되어 있는 하나의 페이지만 보고자 할 때는 전체 데이터를 조인할 필요가 없다
맨 앞에 나오는 페이지 하나만 join 하면 된다. 그럴 경우에는 hash join 보다nested loop join을 사용한다.

오라클에서의 페이징 쿼리를 만들어 보자
-  1. rownum 필드 사용 
- 2. row_number 윈도우 함수 이용

```
-- step1. join, where, 조건절, 정렬 실행
--        함수로 가공작업을 하려고 할 때(예를들어, to_char/nvl 등등) 이 위치에서
--        함수를 호출하면 전체 데이터에 대한 함수를 호출하게 되기 때문에 시간이 많이 걸린다
SELECT empno, ename, dname, hiredate, sal
FROM emp A
LEFT OUTER JOIN dept B on A.deptno=B.deptno
--별도의 검색 조건이 없으면 이 부분에서 한다
ORDER BY hiredate DESC;

-- step2. rownum으로 새로운 번호 필드 부여
SELECT ROWNUM num, A.empno, A.ename, A.hiredate, A.sal
FROM
(
    SELECT empno, ename, dname, hiredate, sal
    FROM emp A
    LEFT OUTER JOIN dept B on A.deptno=B.deptno
    ORDER BY hiredate DESC
)A WHERE ROWNUM <= 10;

-- step3. 위 결과를 다시 서브쿼리로 감싼다. 이때 함수도 적용
SELECT AA.num, AA.empno, AA.ename, TO_CHAR(AA.hiredate, 'YYYY-MM-DD') hiredate, AA.sal, AA.dname
FROM
(
    SELECT ROWNUM num, A.empno, A.ename, A.hiredate, A.sal, A.dname
    FROM
    (
        SELECT empno, ename, dname, hiredate, sal
        FROM emp A
        LEFT OUTER JOIN dept B on A.deptno=B.deptno
        ORDER BY hiredate DESC
    )A WHERE ROWNUM <= 10
) AA WHERE AA.num >= 6;
-- 여기까지가 튜닝되어진 페이징 쿼리
```

cf. where 조건절에 검색 조건이 하나 있을 때
ex. where ename like '% %' and -
1=1 쿼리는 안쓰는 것이 좋음(시스템 성능을 많이 잡음)

## 윈도우 함수

분석을 도와주기 위해 오라클이 제공하는 함수. 기존의 group by와 그룹 함수들의 속도와 사용 문제때문에 윈도우 함수를 사용하는 것이 좋다. 
성능을 개선한 분석 함수들에는 row_number, rank, sum, count ... 등이 있다.

- 윈도우 함수는 구문에 반드시 over() 를 붙여야 한다.
- mssql 에서도 현재 윈도우 함수 개념을 받아 들이고 있다.

ex )
```
select deptno, ename, sum(sal) over() from emp;
```

ex) 부서 별 합계(partition by 필드명 - 그룹을 만들어 준다)
```
select deptno, ename, sum(sal) over(partition by deptno) from emp;
```
ex) 정렬
```
select deptno, empno, ename, row_number() over(order by empno desc) num from emp;
```

### 인덱스를 타지 않는 경우
- is null/ is not null 인 경우
(해결법: 필드에 null 제외시키기, 음수나 '' 등을 통해 실제 사용하지 않는 값을 넣어 null이 되지 않도록 한다. 이외에 비트맵 인덱스를 생성한다)
*비트맵 인덱스 : null 값이 만든 필드를 위한 인덱스

- 내외부 변형 발생 시 
내부변환 : 시스템 내부에서 문제 없음 판단 이 후 자체적으로 변환하는 것, 이에 대한 해결책은 실행계획을 보고 우변에 전달될 값에 정확한 타입을 지정
- 검색 조건에 함수를 사용할 경우
- 검색 조건에 수식이 있는 경우 
 : 수식의 경우 수식을 바꾸기 또는 수식에 인덱스를 만든다.

## 결합 인덱스

두 개 이상의 컬럼이 WHERE 조건 절에서 자주 같이 사용될 때 두 개 이상의 컬럼을 하나의 인덱스로 만들어 주는 것이 좋다.
한 컬럼값이 NULL 이라도 인덱스를 타게 된다. 그리고 하나의 컬럼만 WHERE 조건절에 나타나도 결합 인덱스를 사용할 수 있도록 힌트HINT를 줄 수 있다.

결합인덱스 예시)
선언
```
CREATE INDEX idx_complex012 ON myemp(deptno, job, ename);
```
활용
```
explain plan for
select * from myemp
where deptno=10 and job = 'CLERK' and ename='test112';
SELECT * FROM table(dbms_xplan.display);
```

#### 힌트는 시스템화황경이나 현재 데이터 상황에 따라 적용이 달라짐
힌트를 주었지만 index를 실행하지 않음
```
explain plan for select /*+INDEX_SS(myemp, idx_complex012*/ * from myemp
where job = 'CLERK';
SELECT * FROM table(dbms_xplan.display);
```


#### like 연산자가 검색조건 앞에 오면 인덱스를 타지 않는다

인덱스선언
```
create index idx_myemp_ename on myemp(ename);
```
인덱스를 타는 구문
```
explain plan for select * from myemp
where ename like 'test111%';
SELECT * FROM table(dbms_xplan.display);

explain plan for select * from myemp
where ename like 't%est';
SELECT * FROM table(dbms_xplan.display);
```

인덱스를 타지 않는 구문
```
explain plan for select * from myemp
where ename like '%test';
SELECT * FROM table(dbms_xplan.display);
```

: 해결책
- 필드를 나눈다. 성(first) 필드와 이름(name)필드를 나눠 따로 따로 만들어서 관리. 즉, like '%' 상황을 만들지 않도록 한다
- 도메인 인덱스 : 지금까지 정해지지 않았던 새로운 인덱스를 만들어 사용할 수 있다. 개발자가 원하는 인덱스 타입을 지정할 수 있다.(오라클 8i부터 등장하였다 . 비정형 text를 빠르게 검색하기 위해 도입하였다. 종류에는 intermedia text index, context, catalog 등이 있다


### 도메인 인덱스
도메인 인덱스를 만드는 방법

1. ctxsys 유저의 lock을 해제, ctxsys는 오라클을 설치하면 있는 계정이다
```
conn /as sysdba
alter user ctxsys account unlock;
```
2. 사용자에게 ctxapp 권한 부여
```
grant ctxapp to user01;
```
3. 특별한 타입의 인덱스를 만들자
ctxsys.context 타입 지정
```
create index idx_myemp_text on myemp(ename) indextype is ctxsys.context;
```

4. like 말고 contains 함수를 사용
```
explain plan for select * from myemp where contains(ename, 'test111') >0;
SELECT * FROM table(dbms_xplan.display);
```

실행 예시>  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57831923-62356180-77f1-11e9-9c19-b718f274a99e.png"></img>


## UNION 명령어
(가끔 사용)
: 두 개 이상의 데이터 집합을 하나로 합칠 때 사용한다.
- union all  단순 합치기
 union 합치고 distinct 적용하기 

유니온 예제)
```
select empno, ename
from emp
where empno<=8000
union all
select empno, ename
from myemp
where empno <=10;
```
예제 실행화면)  
<img width="100" src="https://user-images.githubusercontent.com/28393778/57832402-8e051700-77f2-11e9-8700-1d686a324860.png"></img>





<hr>
