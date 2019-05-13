##   PL/SQL Study


(쿼리 명령어가 아니면 세미콜론(;) 안붙여도 됨)

계정 lock 해제하기
alter user hr account unlock;

hr 계정 패스워드 수정하기
alter user hr identified by 1234;

## PL/SQL
<h3>--set serveroutput on ; -- 로그온 할 때마다 설정해주어야 출력가능
</h3>
- PL/SQL 출력
begin  
dbms_output.put_line('Hello')
;
end;
/

- 상수
-- 상수 변수에 constant 키워드를 붙이면 상수가 된다.
declare
    x constant number := 10;
    y number;
    name varchar2(10):='홍길동';
begin
    dbms_output.put_line(x);
    dbms_output.put_line(y);
    dbms_output.put_line(name);
    
    x:= 100; -- error발생
end;
/

- 값 입력받기
-- 사용자로부터 입력을 받을 때
declare
    x number;
    y number;
    z number;
begin
    x := &xnum; -- &를 붙이면 값을 외부로부터 입력을 받는다
    y := &ynum;
    z := x+y;
    dbms_output.put_line(z);
end;
/

- 급여 찾기
-- 사원번호 7934인 사람의 이름과 급여를 출력하라
declare
    vname varchar2(10);
    vsal  number;
    vno   number;
begin
    select ename, sal into vname, vsal
    from emp
    where empno=&vno;
    dbms_output.put_line(vname ||'의 급여는'|| vsal ||'입니다');
end;
/

- PL/SQL JOIN 쿼리
declare 
    vfirst_name varchar2(40);
    vsalary number;
    vdname varchar2(40);
begin
    select first_name, salary,department_name dname
    into vfirst_name, vsalary, vdname
    from employees a
    left outer join departments b on a.department_id=b.department_id
    where employee_id =&empno;
    dbms_output.put_line('이름 ' || vfirst_name);
    dbms_output.put_line('급여 ' || vsalary);
    dbms_output.put_line('부서 ' || vdname);
end;

## %TYPE 데이터형
기술한 DB 컬럼 정의를 정확히 알지 못하는 경우에 사용

## if문
--if문
declare
    vempno emp.empno%TYPE;
    vename emp.ename%TYPE;
    vdeptno emp.deptno%type;
    vdname varchar2(20):= null;
    
begin
    select empno, ename, deptno into vempno, vename, vdeptno
    from emp
    where empno=&empno;
    
    if     (vdeptno=10) then vdname :='총무부';
    elsif(vdeptno=20) then vdname :='인사부';
    elsif(vdeptno=30) then vdname :='홍보부';
    elsif(vdeptno=40) then vdname :='개발부';
    else                     vdname :='부서 할당 안됨';
    end if;
    dbms_output.put_line(vdname);
end;
/

- if문 문제1
--if문 문제1
declare
    vempno emp.empno%TYPE;
    vcomm emp.comm%TYPE;
begin
    select empno,comm
    into vempno, vcomm
    from emp
    where empno = &empno;
    
    if(vcomm > 0) then
        dbms_output.put_line(vcomm);
    else
        dbms_output.put_line('보너스가 없습니다');
    end if;
end;
/

- if문 문제2
declare 
    vempno  emp.empno%TYPE; 
    vename  emp.ename%TYPE;
    vsal    emp.sal%TYPE;
begin
    select empno, ename, sal 
    into vempno, vename, vsal
    from emp
    where empno=&empno;
    
    if(vsal>=4000) then dbms_output.put_line( '연봉 : '||vsal ||'세금 '||vsal * 0.04);
    elsif(vsal >=3000 and vsal<4000) then dbms_output.put_line('연봉 : '||vsal ||'세금 '||vsal * 0.03);
    elsif(vsal >=2000 and vsal<3000) then dbms_output.put_line('연봉 : '||vsal ||'세금 '||vsal * 0.02);
    else dbms_output.put_line('연봉 : '||vsal ||'세금 '||vsal * 0.01);
    end if;
end;
/

(and연산은 최대한 안하는걸로

## CASE 명

declare
    vempno emp.empno%TYPE;
    vename emp.ename%TYPE;
    vdeptno emp.deptno%TYPE;
    vdname varchar2(40);
begin
    select empno, ename, deptno
    into vempno, vename, vdeptno
    from emp
    where empno=&empno;
    
    vdname := case(vdeptno)
                when 10 then '총무부'
                when 20 then '홍보부'
                when 30 then '영업부'
                when 40 then '개발부'
              end;
    dbms_output.put_line(vename || '' || vdname);
end;
/

## LOOP 
loop 문	무조건 진행한다

loop
exit when 조건식 -- 조건식을 만족하면 루프를 벗어남
end loop;

-- 반복문 2 (구구단)
declare
    vdan number(2) := &vdan;
    I    number(2) default 0;
    TOT  number := 0;
begin
    I:=2;
    loop
        I := I + 1;
        TOT := vdan * I;
        dbms_output.put_line(vdan || ' * ' || I || ' = ' || TOT);
        exit when I = 9;
    end loop;
end;
/

## for 문
증감치가 없음
형식
for 변수 in 시작...종료 loop

end loop 로 마감

예제)
-- for 반복문 예제
declare
    i number(2);
begin
    for i in 1..10 loop
        dbms_output.put_line(i);
    end loop;
end;
/

예제 반대로 출력)
declare
    vdan    number(2) :=2;
    I       number(2) default 0;
    tot     number :=0;
    
begin
    for I in reverse 1..9 loop
        tot := vdan * I;
        dbms_output.put_line(vdan || ' * ' || I || ' = ' || tot);
    end loop;
end;
/

--for문 리스트로 받기
begin
    for emp_list in (select empno, ename, sal from emp) loop
        dbms_output.put_line(emp_list.empno || ' ' || emp_list.ename || ' ' || emp_list.sal);
    end loop;
end;
/


#### for문과 loop, while문 구분하기

- for : 무조건 실행, 언제 끝날지 카운트로 알 수 있는 경우
- loop, while  : 조건에 의해 종료
카운트 의외의 조건에 조건에 의해 종료 while은 한 번도 수행이 안될 수 있음



보통 1, 2, 3, 4 이런 식으로 카운트를 해서, 언제쯤 일이 끝날지 알 수 있는 경우에는 주로 for문을 사용한다. 위처럼 카운트를 해서 종료 조건을 아는 것이 아니라 비밀번호 오류 시 처리나, 사용자가 값 입력을 잘못 입력해서 오류 처리나, 데이터의 끝이라거나 등 이런식으로 카운트 종료 상황을 알 수 없을 경우에는 loop 나 while 문을 사용한다.

loop 문은 반드시 한번은 수행을 한다. while 문은 때로는 단 한번도 수행을 할 가능성이 없을 때 사용하는 것이 좋다.

형식
while 조건식 loop 
end loop

<hr>

### 함수의 문법
```
create or replace function test1
return number is
    x number;
    y number;
begin
    x := 100;
    y := 100;
    return x+y;
end;
/

select test1 from dual;
```

```
-- 함수명을 GetTableCount;
create or replace function GetTableCount
return number is
    cnt number;
begin
    select count(*) into cnt from tab;
    return cnt;
end;
/

select GetTableCount() from dual;
```
-- 문제2 부서코드가 10인 부서에서 최고 급여액을 받는 사람을 반환하는 함수 만들기 : GetMaxSalName()
create or replace function GetMaxSalName
return varchar2 
is
    maxSalName varchar2(10);
begin
    select ename 
    into maxSalName
    from emp
    where deptno=10 and sal = (select max(sal) from emp);
    return maxSalName;
end;
/

select GetMaxSalName() from dual;

-- 함수가 아닌 서브쿼리로 호출하기
begin
    select ename
    into result
    from emp where deptno=10 and sal = (select max(sal) result from emp where deptno=10);
    return result;
end;
/


- 파라미터로 값 넘기기  
create or replace function GetDouble(v in number)
return number
is
begin
    return v*2;
end;
/

select ename, GetDouble(sal) from emp;


> Written with [StackEdit](https://stackedit.io/)  
