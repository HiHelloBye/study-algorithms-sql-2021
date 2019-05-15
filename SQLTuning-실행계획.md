
# SQL Tuning 


SQL 문제 유형
- 인덱스
	조건절에서 비교하는 컬럼에 인덱스가 없는 경우
	스칼라 서브 쿼리(데이터를 단 하나만 가져오는 쿼리)의 조인 조건으로 사용된 컬럼에 인덱스가 없는 경우
	START WIDHT, PRIOR(계층형) 구문에서 사용한 컬럼에 인덱스가 없는 경우


옵티마이저
" 가장 효율적인 방법으로 SQL을 수행할 최적의 경로를 생성해주는 DBMS의 핵심엔진이다 "

튜닝의 계획을 세웠으면 반드시 실행시켜봐야한다.

SQL 계획 확인  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57744379-f628ff80-7703-11e9-8c61-71147920d611.png"></img>


< 실행 계획에서 in 연산자의 변형 상황을 보려고 한다>
explain plan for 오라클의 CBO(비용 기반 옵티마이저)가 실행 계획을 세우게 한다.
주의 : 실제로 실행하지는 않는다(계획만 세움) 세운 계획이 달라도 실행 결과는 동일하다. 
검색 때 인덱스와 무관하게 같은 결과가 나온다. 다만 실행 계획을 진행하는데 비용이 더 추가된다.
예를 들어, 신림동에서 구의역까지 출퇴근을 할 때 다양한 경로와 비용이 있지만 결국 목적은 구의역에 도착하는 것이다.

ex)
```
explain plan for select * from emp where deptno in (10, 20);
```
실행계획 확인
```
select * from table(dbms_xplan.display);
```
select 문은 순차 검색을 진행한다(Table Access Full) 
where deptno in (10, 20) 구문은 실행 계획에서 filter("deptno" = 10 or "deptno" =20) 으로 바뀐다.

## 실행 계획
오라클이 테이블을 만들면 rowid 라는 필드가 자동으로 생긴다. 이 필드에는 실제 데이터가 있는 위치 값 등이 저장되어 있다.

select * from emp; 
위의 쿼리는 rowid가 안나온다. -> 필드를 지정하면 조회할 수 있다
select rowid, ename, empno from emp;

rowid user 검색 plan
```
explain plan for select rowid, ename, empno from emp where rowid='AAAE7tAABAAALHpAAA';
select * from table(dbms_xplan.display);
```
실행 결과  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57749401-e10a9b80-7718-11e9-8067-624e832b7317.png"></img>

<br><br>
테이블에 primary key를 추가하면 자동으로 인덱스가 만들어진다
이때 인덱스는 중복된 값을 허용하지 않으므로 unique 인덱스가 만들어져서 검색 시 속도가 빨라진다.

- PK 추가 방법
alter table 테이블 명 add constraint pk_명 (필드명);
```
--empno 필드에 pk 지정
alter table emp add constraint PK_EMP primary key (empno);
```
pk 지정 & 실행 계획 지정
```
alter table emp add constraint PK_EMP primary key(empno);

explain plan for select * from emp where empno = 8000;
select * from table(dbms_xplan.display);
```

인덱스 스캔 실행결과>  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57749893-f1237a80-771a-11e9-9965-00b45d8ad60c.png"></img>

(작업 건수가 적으면 차이가 별로 없음)

- 인덱스 만드는 명령어
create index 인덱스명 on 테이블명(인덱스를 적용할 컬럼)

- unique 인덱스 만드는 명령어
unique 인덱스는 컬럼의 값이 중복되지 않아야 한다. unique 제약조건이 있을 때 만들 수 있다.
create unique index 인덱스명 on 테이블명(인덱스를 적용할 컬럼)

```
--myemp 테이블 gender 필드에 인덱스를 만들어보자
create index IDX_MYEMP_GENDER on myemp(gender);

--인덱스가 생성되어 졌는지 user_indexes 테이블에서 확인
select * from user_indexes;

```

**인덱스는 메모리를 차지하기 때문에 검색에 쓰이지 않는 인덱스는 만들지 않는 것이 좋음**

인덱스 예시)
```
explain plan for select * from myemp where gender='M';
select * from table(dbms_xplan.display);
```

강제로 인덱스 실행시키기)
쿼리에 Hint 를 주어 강제로 index를 타도록 할 수 있다
힌트는 /*+ index(테이블, 인덱스명)*/ 를 select 구문 뒤에 둔다
```
explain plan for select /*+index(myemp, idx_myemp_gender)*/ * from myemp where gender='M';
select * from table(dbms_xplan.display);
```

힌트를 사용해 강제로 실행계획을 수정했는데 예측 비용은 table full scan 보다 훨씬 많이 든다. 이럴경우에는 힌트도 생략하고 가능하면 인덱스를 삭제하는 것이 좋다.

하지만 때로는 삭제해서는 안되는 경우도 있다. 예를들어, 특정 대상만 분포도를 많이 차지할 경우에 특정 대상에 대해서는 full scan를 하는 것이 좋다.
> (ex. 주소에서 서울시, 다른 시도시는 작음) 나머지 대상에 대해서는 index scan을 해야하는데, 이럴 경우에 서울시는 반드시 full scan을 하고 index scan 을 피하고 싶다면
> explain plan for select /*+index(my emp, idx_myemp_gender)*/ * from myemp where gender||''='M';
select * from table(dbms_xplan.display);
필드 쪽에 변형을 가해 인덱스를 피할 수 있다.  gender = 이 인덱스를 타지 gender ||'' 은 새로운 수식이라 이 수식은 index를 타지 않도록 되어 있다(힌트를 줘도 수식자체가 변형이 되어서 인덱스를 타지 않는다)

>실행 결과   
><img width="400" src="https://user-images.githubusercontent.com/28393778/57750881-dfdc6d00-771e-11e9-9130-9aee998fb73c.png"></img>


- 인덱스 삭제
drop index 인덱스명
```
drop index idx_myemp_gender;
```

- not unique index 만들기
```
create index idx_myemp_ename on myemp(ename);

explain plan for select * from myemp where ename='test9999';
select * from table(dbms_xplan.display);
```


인덱스 예문>
```
-- index 탄다
explain plan for select * from myemp where ename='test9999%';
select * from table(dbms_xplan.display);

-- index 무시, like 바로 뒤에 %가 있으면 시작 위치를 모르기 때문에
-- 인덱스를 타지 않는다
explain plan for select * from myemp where ename like'%9999';
select * from table(dbms_xplan.display);
```

### index 실행계획의 종류
#### INDEX SKIP SCAN
INDEX SKIP SCAN 이 발생하는 경우

- 두 개이상의 컬럼을 하나의 인덱스로 만들어 사용할 수 있다. 이를 결합 인덱스라고 한다.
- 만일 WHERE 조건절에 있는 필드가 별도의 인덱스가 없다면 당연히 인덱스를 타지 않는다
- 그러나 이 필드가 결합 인덱스의 형태로 다른 컬럼과 연결되어 있다면 WHERE 조건절에 2개의 필드가 다 나타나지 않아도 INDEX 스캔을 할 수 있다.
- 이때의 스캔방법을 INDEX SCAN이라고한다. 힌트를 주지 않으면 안타기 때문에 반드시 힌트를 주어야 한다.

- INDEX FAST FULL SCAN

```
explain plan for select count(*) from myemp;
select * from table(dbms_xplan.display);
```
실행결과>  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57753609-8af12480-7727-11e9-9b4e-5f41942dd201.png"></img>


- 인덱스 전/후 cost 차이 확인
```
explain plan for select max(hiredate) from myemp;
select * from table(dbms_xplan.display);

create index idx_myemp_hiredate on myemp(hiredate);
explain plan for select max(hiredate) from myemp;
select * from table(dbms_xplan.display);

```
index x  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57754005-75c8c580-7728-11e9-8d79-c9f7b34816a8.png"></img>

index o  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57754093-a6a8fa80-7728-11e9-866d-049a2117e4b4.png"></img>


- 서브쿼리를 사용했을 경우 실행계획 살펴보기
```
explain plan for select ename, (select dname from dept b where a.deptno=b.deptno) dname
from myemp a;
select * from table(dbms_xplan.display);
```

일반적으로 조인이 서브쿼리보다 빠르다. 그러나 서브쿼리가 드라이빙 되는 테이블의 크기가 작을 경우에 서브쿼리를 수행한 결과가 캐쉬가 된다. 즉 서브쿼리는 실행될 때마다 다시 테이블에 가서 데이터를 읽어오는게 아니라 이미 이전에 그 쿼리를 실행한 결과가 있을 경우에 결과만을 불러오지 계속 다시 수행되는 게 아니기 때문에 데이터 검색이 이루어 지는 테이블이 충분히 작다면 join 보다 빠를 수 있다
(항상은 아님)

 - 보통 join > 서브쿼리 > 함수 순으로 비용이 발생 


### 실행계획만으로는 실제 비용을 알 수 없다. 
실행계획은 그간 정보들을 통해 예측만 할 뿐. 실제 비용을 알기 위해서는 trace를 추적해 실행 결과를 확인해야한다

### 함수는 서브쿼리와 달리 실행될 때마다 호출된다. 실행계획에 그 호출 부분이 나오지 않아 조인보다 빨라 보일 수 있다.

함수 만들기>
```
create or replace function GetDname(pdeptno in number)
return varchar2 is
    v_dname varchar2(20);
begin
    select dname into v_dname
    from dept
    where deptno = pdeptno;
    
    return v_dname;
end;
/
explain plan for select ename, GetDname(deptno) from myemp;
select * from table(dbms_xplan.display);
```

실행결과>  
<img width="400" src="https://user-images.githubusercontent.com/28393778/57756952-5bdeb100-772f-11e9-9bc1-3d1c9045ecdf.png"></img>



<hr>

