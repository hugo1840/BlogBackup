---
tags: [oracle]
title: Oracle游标的分类与用法
created: '2023-07-23T03:57:47.287Z'
modified: '2023-07-23T04:08:58.487Z'
---

Oracle游标的分类与用法

游标即Cursor，是Oracle内存中指向私有SQL区的一个指针，其中存储了处理某个SELECT语句或者DML语句的相关信息。

Oracle游标可以分为隐式游标和显式游标，两种游标都可以在PL/SQL脚本中使用。

# 隐式游标
隐式游标由PL/SQL创建和管理，用户无法控制。用户每执行一条SELECT或DML语句时就会打开一个隐式游标。

隐式游标在对应的SQL执行完毕后就会自动关闭，但是游标中的信息会保留到执行下一条SELECT或DML语句。

如果后续要使用隐式游标中的信息，最好将其保存到一个本地变量中，以免被其他操作的游标信息覆盖。

## 获取游标数据
用户可以通过以下两种途径来获取隐式游标中的数据：
- 利用SELECT INTO来获取游标数据；
- 在FOR LOOP循环中使用隐式游标。

**示例**：通过SELECT获取游标中的数据。
```sql
SELECT 列名 INTO 变量名 FROM 表名;
```
SELECT INTO通过隐式游标来获取和保存数据。


**示例**：通过FOR LOOP获取游标中的数据。
```sql
BEGIN
  FOR item IN (
    SELECT last_name, job_id
    FROM employees
    WHERE job_id LIKE '%CLERK%'
    AND manager_id > 120
    ORDER BY last_name
  )
  LOOP
    DBMS_OUTPUT.PUT_LINE
      ('Name = ' || item.last_name || ', Job = ' || item.job_id);
  END LOOP;
END;
/
```

## 游标内置变量
隐式游标支持以下内置变量：
- `SQL%ISOPEN`：对于隐式游标，该变量值始终为FALSE。因为隐式游标会自动关闭。
- `SQL%FOUND`：
  - 等于NULL，如果没有执行SELECT或DML语句；
  - 等于TRUE，如果SELECT或DML语句返回或操作的行数不少于一行；
  - 其他情况下都等于FALSE。
- `SQL%NOTFOUND`：
  - 等于NULL，如果没有执行SELECT或DML语句；
  - 等于FALSE，如果SELECT或DML语句返回或操作的行数不少于一行；
  - 其他情况下都等于TRUE。
  - `SQL%NOTFOUND`不适用于SELECT INTO语句。
- `SQL%ROWCOUNT`：
  - 等于NULL，如果没有执行SELECT或DML语句；
  - 等于SELECT或DML语句返回或操作的行数。
  - 如果不带BULK COLLECT关键字的SELECT INTO语句返回了多行，PL/SQL会报错TOO MANY ROWS。此时`SQL%ROWCOUNT`的值等于1,而不是实际的行数。
  - `SQL%ROWCOUNT`的值不受事务状态变化的影响。如果事务回滚了，`SQL%ROWCOUNT`的值并不会变回到SAVEPOINT之前。


**示例**：
```sql
DROP TABLE employees_temp;
CREATE TABLE employees_temp AS
  SELECT * FROM employees;

DECLARE
  mgr_no NUMBER(6) := 122;
BEGIN
  DELETE FROM employees_temp WHERE manager_id = mgr_no;
  DBMS_OUTPUT.PUT_LINE
    ('Number of employees deleted: ' || TO_CHAR(SQL%ROWCOUNT));
END;
/
```


# 显式游标
显式游标由用户创建和管理。用户必须手动DECLARE和DEFINE一个显示游标，并关联到目标SQL。

## 获取游标数据
用户可以通过以下两种途径来获取显式游标中的数据：
- 先OPEN显式游标，然后FETCH游标中的数据，最后CLOSE游标；
- 在FOR LOOP循环中使用显式游标。

**示例**：通过FETCH获取游标中的数据。
```sql
DECLARE
  CURSOR c1 IS
    SELECT last_name, job_id FROM employees
    WHERE REGEXP_LIKE (job_id, 'S[HT]_CLERK')
    ORDER BY last_name;

  v_lastname  employees.last_name%TYPE;  -- 定义v_lastname的数据类型与employees.last_name相同
  v_jobid     employees.job_id%TYPE;     

  CURSOR c2 IS
    SELECT * FROM employees
    WHERE REGEXP_LIKE (job_id, '[ACADFIMKSA]_M[ANGR]')
    ORDER BY job_id;

  v_employees employees%ROWTYPE;  -- 定义v_employees用于存放employees中的一整行数据

BEGIN
  OPEN c1;
  LOOP  
    FETCH c1 INTO v_lastname, v_jobid;
    EXIT WHEN c1%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE( RPAD(v_lastname, 25, ' ') || v_jobid );
  END LOOP;
  CLOSE c1;
  DBMS_OUTPUT.PUT_LINE( '-------------------------------------' );

  OPEN c2;
  LOOP  
    FETCH c2 INTO v_employees;
    EXIT WHEN c2%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE( RPAD(v_employees.last_name, 25, ' ') ||
                               v_employees.job_id );
  END LOOP;
  CLOSE c2;
END;
/
```

**示例**：通过FOR LOOP获取游标中的数据。
```sql
DECLARE
  CURSOR c1 IS
    SELECT last_name, job_id FROM employees
    WHERE job_id LIKE '%CLERK%' AND manager_id > 120
    ORDER BY last_name;
BEGIN
  FOR item IN c1
  LOOP
    DBMS_OUTPUT.PUT_LINE
      ('Name = ' || item.last_name || ', Job = ' || item.job_id);
  END LOOP;
END;
/
```


## 游标传入参数
显式游标可以接受传入参数，在每次打开游标时获取对应的结果集。传入参数可以通过DEFAULT关键字指定默认值。
```sql
DECLARE
  CURSOR c (job VARCHAR2, max_sal NUMBER DEFAULT 6000) IS
    SELECT last_name, first_name, (salary - max_sal) overpayment
    FROM employees
    WHERE job_id = job
    AND salary > max_sal
    ORDER BY salary;
 
  PROCEDURE print_overpaid IS
    last_name_   employees.last_name%TYPE;
    first_name_  employees.first_name%TYPE;
    overpayment_      employees.salary%TYPE;
  BEGIN
    LOOP
      FETCH c INTO last_name_, first_name_, overpayment_;
      EXIT WHEN c%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(last_name_ || ', ' || first_name_ ||
        ' (by ' || overpayment_ || ')');
    END LOOP;
  END print_overpaid;
 
BEGIN
  DBMS_OUTPUT.PUT_LINE('----------------------');
  DBMS_OUTPUT.PUT_LINE('Overpaid Stock Clerks:');
  DBMS_OUTPUT.PUT_LINE('----------------------');
  OPEN c('ST_CLERK');
  print_overpaid; 
  CLOSE c;

  DBMS_OUTPUT.PUT_LINE('----------------------');
  DBMS_OUTPUT.PUT_LINE('Overpaid Stock Clerks:');
  DBMS_OUTPUT.PUT_LINE('----------------------');
  OPEN c('ST_CLERK', 5000);
  print_overpaid; 
  CLOSE c;
 
  DBMS_OUTPUT.PUT_LINE('-------------------------------');
  DBMS_OUTPUT.PUT_LINE('Overpaid Sales Representatives:');
  DBMS_OUTPUT.PUT_LINE('-------------------------------');
  OPEN c('SA_REP', 10000);
  print_overpaid; 
  CLOSE c;
END;
/
```

## 游标内置变量
显示游标中内置变量的含义与隐式游标略有不同：
- `SQL%ISOPEN`：如果显式游标已经打开，值为TRUE，否则为FALSE。
- `SQL%FOUND`：
  - 等于NULL，如果显式游标已经打开，但是还没有FETCH过数据；
  - 等于TRUE，如果最近一次FETCH操作返回了一行数据；
  - 其他情况下都等于FALSE。
- `SQL%NOTFOUND`：
  - 等于NULL，如果显式游标已经打开，但是还没有FETCH过数据；
  - 等于FALSE，如果最近一次FETCH操作返回了一行数据；
  - 其他情况下都等于TRUE。
- `SQL%ROWCOUNT`：
  - 等于0，如果显式游标已经打开，但是还没有FETCH过数据；
  - 等于已经FETCH的数据行数。


# 游标变量
游标变量与显式游标类似，但是还支持以下特性：
- 可以被重复利用，对应不同的SQL；
- 可以赋值；
- 可以用于表达式；
- 可以作为子程序参数，在不同的子程序间传递结果集；
- 可以作为宿主变量，在PL/SQL存储过程和客户端之间传递结果集；
- 不能接受传入参数。


**References**:
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/static-sql.html#GUID-F1FE15F9-5C96-4C4E-B240-B7363D25A8F1
【2】 https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/static-sql.html#GUID-4A6E054A-4002-418D-A1CA-DE849CD7E6D5


