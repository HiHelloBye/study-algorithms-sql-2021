
- 동적 실행
```
create or replace function GetTotal(tablename in varchar2)
return number is
    result number;
    s varchar2(1000); -- 2의 8승 -1
begin
    --select count(*) into result from emp;
    --select count(*) into result from dept;
    
    s:= ' select count(*) from ' || tablename;
    dbms_output.put_line(s);
    
    EXECUTE IMMEDIATE s into result; -- s를 실행하고 결과를 저장하자
    return result;
end;
/
select GetTotal('emp') from dual;
select GetTotal('dept') from dual;
```

- 함수작성예시
```
create or replace function fc_update_sal (v_empno in number)
return number
is
    v_sal emp.sal%type;
begin
    update emp set sal = sal * 1.1
    where empno = v_empno;
    commit;
    
    select sal into v_sal
    from emp
    where empno = v_empno;
    return v_sal; -- 리턴이 반드시 존재해야 합니다
end;
/

var salary number;
execute :salary:= fc_update_sal(7900);

print salary;

```

 - dept 테이블에 deptno, dname, loc -> exec insertDept(70, '영업', '광주');
```
create or replace procedure insertDept2
(v_deptno in number, v_dname in varchar2, v_loc in varchar2)
is
begin
    insert into dept values (v_deptno, v_dname, v_loc);
    commit;
end;
/

exec insertDept2(70, '영업', '광주');
```

### 암시적 커서의 속성
SQL%ROWCOUNT  : 해당 SQL 문에 영향을 받는 행의 수
SQL%FOUND  :해당 SQL문에 영향을 받는 행의 수가 한 개 이상일 경우 TRUE
SQL%NOTFOUND : 해당 SQL문에 영향을 받는 행의 수가 없을 경우 TRUE
SQL%ISOPEN  :항상 FALSE, 암시적 커서가 열려 있는지의 여부 검색

- 암시적 커서 예
```
create or replace procedure Implicity_Cursor (p_empno in emp.empno%type)
is
    v_sal   emp.sal%type;
    v_update_row    number;
begin
    select sal
    into v_sal
    from emp
    where empno = p_empno;
    
    --검색된 데이터가 있을 경우
    if sql%found then
        dbms_output.put_line('검색한 데이터가 존재합니다 : ' || v_sal);
        
        update emp
        set sal = sal * 1.1
        where empno = p_empno;
        --수정한 데이터의 카운트를 변수에 저장
        v_update_row := sql%rowcount;
        
        dbms_output.put_line('급여가 인상된 사원 수 : ' || v_update_row);
    end if;
    exception
        when no_data_found then
         dbms_output.put_line('검색한 데이터가 없네요...');
end;
/

execute Implicity_Cursor(7369);
```

- 문제 )  사원번호, 이름, 부서명 출력
```
create or replace procedure select_proc
is
    cursor sel_cursor is
        select e.empno, e.ename, d.dname 
        from emp e, dept d 
        where e.deptno = d.deptno;
    
    vempno  emp.empno%type;
    vname   emp.ename%type;
    vdname  dept.dname%type;
begin
    open sel_cursor;
    loop
        fetch sel_cursor into vempno, vname, vdname;
        dbms_output.put_line(to_char(vempno) || ' ' || vname || ' ' || vdname );
        exit when sel_cursor%notfound;
    end loop;
    close sel_cursor;
end;
/

exec select_proc

```

- 커서에 인자 넣기
```
create or replace procedure proc_select_emp(pdeptno in dept.deptno%type)
    is
        -- 커서 만들 때 파라미터를 따로 지정해주어야 한다
        cursor sel_cursor(v_deptno in dept.deptno%type) is
            select ename, dname, sal from emp a
            left outer join dept b on a.deptno=b.deptno
            where a.deptno = v_deptno;
            
            v_ename emp.ename%type;
            v_sal   emp.sal%type;
            v_dname dept.dname%type;
    begin
        open sel_cursor(pdeptno);
        loop
            fetch sel_cursor into v_ename,v_dname,v_sal;
            exit when sel_cursor%notfound;
            dbms_output.put_line(v_ename || ' ' || v_sal || ' ' || v_dname);
        end loop;
    end;
/

exec proc_select_emp(20);
```

- 파라미터 2개
```
create or replace procedure proc_select_emp(
    pdeptno in dept.deptno%type,
    pjob in emp.job%type)
    is
        -- 커서 만들 때 파라미터를 따로 지정해주어야 한다
        cursor sel_cursor(pdeptno in dept.deptno%type, pjob in emp.job%type) 
        is
            select ename, dname, sal from emp a
            left outer join dept b on a.deptno=b.deptno
            where a.deptno = v_deptno and a.job liek '%'||pjob|'%';
            
            v_ename emp.ename%type;
            v_sal   emp.sal%type;
            v_dname dept.dname%type;
    begin
        open sel_cursor(pdeptno, pjob);
        loop
            fetch sel_cursor into v_ename,v_dname,v_sal;
            exit when sel_cursor%notfound;
            dbms_output.put_line(v_ename || ' ' || v_sal || ' ' || v_dname);
        end loop;
    end;
/

exec proc_select_emp(20);
```

문제)
```
--employee 테이블에 새로운 필드 하나 추가하기
alter table employees add (read char(1));

--read라는 필드를 하나 추가함, 하려는 프로시저: employee테이블로부터
--데이터를 하나씩 읽어서 read 필드값을 1로 세팅하는 작업
create or replace procedure update_read_proc
is
    cursor sel_cursor is
        select first_name, employee_id from employees;
        
        vname employees.first_name%type;
        vempno employees.employee_id%type;
begin
    open sel_cursor;
    loop
        fetch sel_cursor into vname, vempno;
        exit when sel_cursor%notfound;
        dbms_output.put_line(vname || 'is updated');
        update employees set read = '1' where employee_id = vempno;
    end loop;
    close sel_cursor;
    commit;
end;
/
exec update_read_proc;

select first_name, read from employees;

```

- 다시 0으로 되돌리기
```
create or replace procedure update_read_proc2
is
    cursor sel_cursor is
        select first_name, employee_id from employees
        for update;    
    vname employees.first_name%type;
    vempno employees.employee_id%type;
begin
    open sel_cursor;
    loop
        fetch sel_cursor into vname, vempno;
        exit when sel_cursor%notfound;
        dbms_output.put_line(vname || 'is updated');
        update employees set read = '0' 
        where current of sel_cursor;
    end loop;
    close sel_cursor;
    commit;
end;
/
```

### 테이블 만들 때 주의사항

- 웹 서비스 테이블에는 ip주소가 포함되어야 함]
- 이전 상황을 백업하기 위해서 HISTORY 테이블이 있어야 한다 (데이터가 들어간 날짜(시간) 필드가 반드시 있어야 한다)

cf. null 이 있는 값이 있으면 그룹함수(ex. max, sum ...)는 선택할 수 없어 null을 가져옴
<h4> sequence 만들기

	create sequence board_seq increment by 1 start with 1;
	select board_seq.nextval from dual;

* drop으로 시퀀스 삭제

<h4>example)  

	insert into board values ((select nvl(max(id),0) +1 from board), '제목1', '내용1', sysdate);

## 트랜잭션 처리
#### 두 개의 테이블에 insert 하기 

```
create or replace procedure insert_order(product_id in number, client_id in number, cupon_type char)
is
    v_orderid number;
begin
    --order_id 를 가져오기 위한 쿼리
    select nvl(max(order_id), 0) +1
    into v_orderid
    from tb_order;
    
    dbms_output.put_line(v_orderid);
    insert into tb_order values(v_orderid, product_id, client_id, sysdate); -- tb_order 테이블에 데이터 넣기
    insert into tb_cupon values((select nvl(max(cupon_id), 0)+1 from tb_cupon), v_orderid, cupon_type, sysdate);
    -- 쿠폰테이블에 데이터 넣기
    commit; --문제 없으면 commit
    exception
        when others then
            dbms_output.put_line('error');
            rollback;
    end;
/

exec insert_order(1,1,'A');
exec insert_order(2,3,'AA'); -- 트랜잭션 처리 (one or nothing)
 
select * from tb_order;
select * from tb_cupon;
```
<hr>
<br>

## 예외처리

미리 정의된 예외의 종류

- NO_DATA_FOUND : SELECT문


예외처리 too_many_rows 예시>
```
create or replace procedure PreException_test
(v_deptno in emp.deptno%type)
    is
        v_emp   emp%rowtype;
    begin
        dbms_output.enable;
        
        select empno, ename, deptno
        into v_emp.empno, v_emp.ename, v_emp.deptno
        from emp
        where deptno = v_deptno;
        
        dbms_output.put_line('사번 : '|| v_emp.empno);
        dbms_output.put_line('이름 : '|| v_emp.ename);
        dbms_output.put_line('부서번호 :' || v_emp.deptno);
        
        exception
            when too_many_rows then
                dbms_output.put_line('TOO_MANY_ROWS 에러 발생');
                
            when no_data_found then
                dbms_output.put_line('NO_DATA_FOUND 에러 발생');
            
            when others then
                dbms_output.put_line('기타 에러 발생');
    end;
/
exec PreException_Test(20);
```
exec PreException_Test(70);  // NO_DATA_FOUND 에러

too_many_rows & no_data_found 해결방법 >
```
create or replace procedure PreException_test
(v_deptno in emp.deptno%type)
    is
        v_emp   emp%rowtype;
    begin
        dbms_output.enable; 
        for emp_list in
            (select empno, ename, deptno
            from emp
            where deptno = v_deptno)
            loop
                dbms_output.put_line('사번 : '|| v_emp.empno);
                dbms_output.put_line('이름 : '|| v_emp.ename);
                dbms_output.put_line('부서번호 :' || v_emp.deptno);
            end loop;
        exception
            when too_many_rows then
                dbms_output.put_line('TOO_MANY_ROWS 에러 발생');
                
            when no_data_found then
                dbms_output.put_line('NO_DATA_FOUND 에러 발생');
            
            when others then
                dbms_output.put_line('기타 에러 발생');
    end;
/
```


<br>

## PACKAGE

패키지는 오라클 데이터베이스에 저장되어 있는 서로 관련있는 PL/SQL 프로시져와 함수들의 집합.

패키지는 선언부와 본문 두 부분으로 이루어 진다.

- 선언절은 패키지에 포함된 PL/SQL 프로시저나, 함수, 커서, 변수, 예외 절

선언부>
```
create or replace package pgk_emp_info as
    
        procedure all_emp_info; -- 모든 사원의 사원 정보
        procedure all_sal_info; -- 모든 사원의 급여 정보
        procedure dept_emp_info(v_deptno in number); -- 특정 부서의 사원 정보
        procedure dept_sal_info(v_deptno in number); -- 특정 부서의 급여 정보
    end pgk_emp_info;
    /
```
본문>
```
CREATE OR REPLACE PACKAGE BODY pkg_emp_info AS

     -- 모든 사원의  사원 정보 
     PROCEDURE all_emp_info
     IS

         CURSOR emp_cursor IS
         SELECT empno, ename, to_char(hiredate, 'RRRR/MM/DD') hiredate
         FROM emp
         ORDER BY hiredate;

     BEGIN

         FOR  aa  IN emp_cursor LOOP

             DBMS_OUTPUT.PUT_LINE('사번 : ' || aa.empno);
             DBMS_OUTPUT.PUT_LINE('성명 : ' || aa.ename);
             DBMS_OUTPUT.PUT_LINE('입사일 : ' || aa.hiredate);

         END LOOP;

         EXCEPTION
           WHEN OTHERS THEN
             DBMS_OUTPUT.PUT_LINE(SQLERRM||'에러 발생 ');

     END all_emp_info;


     -- 모든 사원의  급여 정보 
     PROCEDURE all_sal_info
     IS
    
         CURSOR emp_cursor IS
         SELECT round(avg(sal),3) avg_sal, max(sal) max_sal, min(sal) min_sal
         FROM emp;
    
     BEGIN

         FOR  aa  IN emp_cursor LOOP
 
             DBMS_OUTPUT.PUT_LINE('전체급여평균 : ' || aa.avg_sal);
             DBMS_OUTPUT.PUT_LINE('최대급여금액 : ' || aa.max_sal);
             DBMS_OUTPUT.PUT_LINE('최소급여금액 : ' || aa.min_sal);
         
         END LOOP;


         EXCEPTION
            WHEN OTHERS THEN
               DBMS_OUTPUT.PUT_LINE(SQLERRM||'에러 발생 ');
     END all_sal_info;


     --특정 부서의  사원 정보
     PROCEDURE dept_emp_info (v_deptno IN  NUMBER)
     IS

         CURSOR emp_cursor IS
         SELECT empno, ename, to_char(hiredate, 'RRRR/MM/DD') hiredate
         FROM emp
         WHERE deptno = v_deptno
         ORDER BY hiredate;

     BEGIN

         FOR  aa  IN emp_cursor LOOP

             DBMS_OUTPUT.PUT_LINE('사번 : ' || aa.empno);
             DBMS_OUTPUT.PUT_LINE('성명 : ' || aa.ename);
             DBMS_OUTPUT.PUT_LINE('입사일 : ' || aa.hiredate);

         END LOOP;

        EXCEPTION
            WHEN OTHERS THEN
               DBMS_OUTPUT.PUT_LINE(SQLERRM||'에러 발생 ');

     END dept_emp_info;


     --특정 부서의  급여 정보
     PROCEDURE dept_sal_info (v_deptno IN  NUMBER)
     IS
    
         CURSOR emp_cursor IS
         SELECT round(avg(sal),3) avg_sal, max(sal) max_sal, min(sal) min_sal
         FROM emp 
         WHERE deptno = v_deptno;
             
     BEGIN

         FOR  aa  IN emp_cursor LOOP 
 
             DBMS_OUTPUT.PUT_LINE('전체급여평균 : ' || aa.avg_sal);
             DBMS_OUTPUT.PUT_LINE('최대급여금액 : ' || aa.max_sal);
             DBMS_OUTPUT.PUT_LINE('최소급여금액 : ' || aa.min_sal);
         
         END LOOP;

         EXCEPTION
             WHEN OTHERS THEN
                DBMS_OUTPUT.PUT_LINE(SQLERRM||'에러 발생 ');

     END dept_sal_info;        
    
  END pkg_emp_info;
/ 
```
실행>
```
exec pkg_emp_info.all_emp_info;
```

## TRIGGER

INSERT, UPDATA, DELETE문이 TABLE에 대해 행해질 때 묵시적으로 수행되는 PROCEDURE이다.
트리거는 TABLE과는 별도로 DATABASE에 저장된다.(VIEW에는 트리거를 걸 수 없음)

- 행 트리거: 컬럼의 각각의 행의 데이터 행 변화가 생길때 마다 실행되며, 그 데이터 형ㅇ


BEFORE : INSERT, UPDATE, DELETE 문이 실행되기 전에 트리거가 실행된다.
AFTER : INSERT, UPDATE, DELETE문이 실행된 후 트리거가 실행
TRIGGER_EVENT: INSERT, UPDATE, DELETE 중에서 한 개 이상 올 수 있다.

간단한 TRIGGER 예제)
```
create or replace trigger trigger_test
    before
        update on dept for each row
        
    begin
        dbms_output.put_line('변경 전 컬럼 값 : ' || :old.dname);
        dbms_output.put_line('변경 후 컬럼 값 : ' || :new.dname);
    end;
/
-- trigger 실행
update dept set dname = '재무부' where deptno = 30;
```
