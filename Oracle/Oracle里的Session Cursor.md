---
tags: [oracle, SQL优化]
title: Oracle里的Session Cursor
created: '2022-10-11T13:36:34.889Z'
modified: '2022-10-15T15:23:52.619Z'
---

Oracle里的Session Cursor

Session Cursor是当前会话解析和执行SQL的载体，也就是说，session游标用于在当前会话中解析和执行SQL。Oracle在解析和执行目标SQL时，会先去当前session的PGA中找是否有匹配的缓存session cursor。Oracle依靠session cursor来将目标SQL所涉及的数据从Buffer cache的对应数据块读到PGA里，然后在PGA中进行后续处理（如排序、表连接），最后将最终结果返回给用户。

与Shared Cursor一样，Session Cursor也是Oracle自定义的一种C语言复杂结构，也是以哈希表的方式缓存的，只不过是缓存在PGA中，而不是像Shared Cursor一样缓存在SGA的库缓存里。

# Session Cursor的含义
关于会话游标，有以下几点需要注意：

- Session Cursor与session是一一对应的，不同会话之间的session cursor无法共享，这与Shared Cursor有本质区别。

- Session Cursor是有生命周期的，每个session cursor在使用过程中都至少会经历一次Open、Parse、Bind、Execute、Fetch和Close中的一个或多个阶段。用过的session cursor不一定会缓存在对应session的PGA中，这取决于`SESSION_CACHED_CURSORS`的值是否大于0。

- Session Cursor也是以哈希表的方式缓存在PGA中，因此Oracle会通过哈希运算来存储和访问在当前session的PGA中的对应session cursor。

当Oracle第一次解析目标SQL（硬解析）时，会新生成一个Session Cursor和一对Shared Cursor（父游标和子游标）。其中，shared cursor会存储能被所有会话共享、重用的内容（解析树、执行计划等），而session cursor则会经历Open、Parse、Bind、Execute、Fetch和Close中的一个或多个阶段。

对于Oracle 11g之前的版本，在缓存session cursor的哈希表的对应哈希桶中，Oracle会存储目标SQL对应的父游标的库缓存对象句柄地址。即Oracle可以通过session cursor找到对应的parent cursor，进而找到对应child cursor中的解析树和执行计划。

一个Session Cursor只能对应一个Shared Cursor，而一个Shared Cursor可以同时对应多个Session Cursor。当目标SQL以硬解析的方式解析和执行完毕后，对应的shared cursor就已经被缓存到库缓存中，对应的session cursor也已经使用完毕。此时会存在以下两种情况：

- 如果参数`SESSION_CACHED_CURSORS`的值等于0，那么session cursor就会正常执行Close操作。当目标SQL再次执行（软解析）时，可以找到匹配的shared cursor，但是找不到匹配的session cursor。因此Oracle还必须为该SQL生成一个session cursor，并且再经历Open、Parse、Bind、Execute、Fetch和Close中的一个或多个阶段。

- 如果参数`SESSION_CACHED_CURSORS`的值大于0，那么当满足一定额外条件时，Oracle就不会对session cursor执行Close操作，而是会将其标记为Soft Close，同时将其缓存到当前session的PGA中。和软解析相比，此时就省掉了Open一个新的session cursor所需要耗费的时间和资源。这个过程就是所谓的**软软解析**。

软软解析与软解析相比，其好处主要体现在以下两个方面：

- 软软解析在对session cursor的使用方面比软解析更好。软软解析省去了Close一个现有session cursor以及Open一个新的session cursor所需要耗费的资源和时间。

- 在Oracle 11g之前，软软解析在对库缓存相关Latch的争用方面比软解析更好。缓存在PGA中的session cursor所在的哈希桶中已经存储了目标SQL的parent cursor的库缓存对象句柄地址，因此可以通过该地址直接访问对应的父游标，而不再需要先持有Library cache latch。

****
:deer:对shared cursor和session cursor之间关联关系的几点总结如下：

1. 无论是硬解析、软解析、还是软软解析，Oracle在解析和执行目标SQL时，都会先去当前session的PGA中寻找是否存在匹配的session cursor缓存。

2. 如果在当前session的PGA中找不到匹配的缓存session cursor，Oracle就会去库缓存中找是否存在匹配的parent cursor。如果找不到，Oracle就会生成一个新的session cursor和一对shared cursor；如果找到了匹配的父游标，但找不到匹配的子游标，就会生成一个新的session cursor和一个child cursor。这两个过程对应的都是**硬解析**。

3. 如果在当前session的PGA中找不到匹配的缓存session cursor，但在库缓存中找到了匹配的parent cursor和child cursor，Oracle就会新生成一个session cursor，并重用找到的匹配的shared cursor。这个过程对应的是**软解析**。

4. 如果在当前session的PGA中找到了匹配的缓存session cursor，就可以通过该会话游标直接访问到目标SQL对应的parent cursor。这个过程对应的就是**软软解析**。

****

# Session Cursor相关参数解析

## `OPEN_CURSORS`
参数`OPEN_CURSORS`用于设定单个session中同时能够以Open状态并存的session cursor的总数。超过这个值会报错ORA-1000: maximum open cursors excedded。

视图`v$open_cursor`可以用来查询数据库中状态为OPEN或者已经被缓存在PGA中的session cursor的数量和具体信息。也可以从`v$sysstat`中查到当前所有以OPEN状态存在的session cursor的总数。

```sql
--查看参数值
> show parameter open_cursors;

--查看当前会话的session cursor信息
> select sid from v$mystat where rownum<2;
   SID
--------
   145
> select count(*) from v$open_cursor where sid=145;

--查看当前所有状态为open的会话游标的总数
> select name,value from v$sysstat where name='opened cursors current';
```

## `SESSION_CACHED_CURSORS`
参数`SESSION_CACHED_CURSORS`用于设定单个session中能够以Soft Closed状态缓存在PGA中的session cursor的总数。

```sql
> show paramter session_cached_cursors;
```

Oracle会用LRU算法来管理这些已缓存的session cursor，所以即便某个session以soft closed状态缓存在PGA中的session cursor已经达到了参数`SESSION_CACHED_CURSORS`的上限也没有关系。

Session Cursor能够被缓存到PGA中是有额外的限制条件的。在Oracle 11gR2中，一个session cursor能够被缓存到PGA中的必要条件是该会话游标所对应的SQL**解析和执行的次数要超过3次**。

## `CURSOR_SPACE_FOR_TIME`
参数`CURSOR_SPACE_FOR_TIME`在Oracle **11.1.0.7**及后续版本中已经过时。原因是Oracle引入该参数的初衷是为了缓解child cursor上库缓存相关Latch的争用，但是从11gR1开始Mutex已经全面替换了各种库缓存相关Latch。

Session Cursor是有生命周期的，每个session cursor在使用过程中至少都会经历一次Open、Parse、Bind、Execute、Fetch和Close中的一个或多个阶段。当一个目标SQL所对应的session cursor状态是Execute时（即SQL正在执行），Oracle会把该SQL对应的child cursor给**Pin**在库缓存中，以防止该SQL对应的解析树和执行计划被**age out**出库缓存。这个Pin的动作是通过先持有库缓存相关Latch，再持有Library Cache Pin这个Enqueue来实现的。Enqueue是Oracle中比Latch要重的一种锁，它的持有方式遵循先进先出的原则，并且它的持有不具有原子性，所以持有Enqueue往往需要先持有具备原子性的Latch。

在`CURSOR_SPACE_FOR_TIME`默认是**FALSE**的情况下，一旦SQL执行完毕（Execute状态结束），对应的child cursor就可以不Pin在库缓存中了，Oracle会释放child cursor上的Library Cache Pin，目标SQL对应的解析树和执行计划就可以被age out出库缓存了。一个session cursor只会Open和Close一次，但是中间可能会反复经历Parse、Bind、Execute和Fetch的过程，也就是会反复经历——先持有库缓存相关Latch，再持有Library Cache Pin——这个过程，如果这个过程的并发量较高，就可能导致相关的Latch争用。

在`CURSOR_SPACE_FOR_TIME`设置为TRUE后（11gR1之前的版本），session cursor每次execute完以后依然会持有对应的child cursor上的Library Cache Pin，直到session cursor状态变为Close。这可以减少库缓存相关Latch的争用，但是带来的副作用就是当shared pool空间紧张时，本来可以置换出库缓存的child cursor在一段时间就得长驻在库缓存中，从而增加对shared pool的压力。

# Session Cursor的种类和用法
Oracle数据库中的session cursor又细分为三种类型，分别是隐式游标（implicit cursor）、显式游标（explicit cursor）、以及参考游标（Ref cursor）。

## 隐式游标
隐式游标是Oracle数据库中最常见的一种session cursor。隐式游标的生命周期（Open、Parse、Bind、Execute、Fetch和Close）完全由SQL引擎或PL/SQL引擎自动来完成。我们在SQLPlus或者PL/SQL代码中直接执行SQL语句时，Oracle实际上自动帮我们创建了隐式游标来作为SQL执行的载体。

Oracle中的隐式游标具有以下四种常用属性：
- `SQL%FOUND`
- `SQL%NOTFOUND`
- `SQL%ISOPEN`
- `SQL%ROWCOUNT`

1. SQL%FOUND属性

SQL%FOUND表示一条SQL语句被执行成功后受其影响而改变的记录数是否大于或等于1。SQL%FOUND通常适用于执行INSERT、UPDATE、DELETE的DML语句，以及**SELECT INTO**语句。

在一条DML语句被执行前，SQL%FOUND的值是NULL。当该DML语句被执行且成功改变了一条或以上的记录时，或者SELECT INTO语句成功返回一条或以上的记录时，SQL%FOUND的值为TRUE，否则为FALSE。

```sql
--在PL/SQL代码中使用SQL%FOUND
declare
  dept_no number(4) :=50;
begin
  delete from dept where deptno=dept_no;
  if sql%found then
    insert into dept vlaues(50,'Physics','LA');
  end if;
  commit;
end;
/
```

在PL/SQL代码中使用SELECT INTO语句时，**当且仅当对应的SELECT语句的返回结果只有一条记录时**，Oracle才不会报错。如果SELECT的返回结果为0，会报错`NO_DATA_FOUND`；如果SELECT返回的结果大于1，则会报错`TOO_MANY_ROWS`。

2. SQL%NOTFOUND属性

SQL%NOTFOUND属性表示一条SQL语句被执行成功后受其影响而改变的记录数是否为0。SQL%NOTFOUND也适用于执行INSERT、UPDATE、DELETE的DML语句，以及SELECT INTO语句。

在一条DML语句被执行前，SQL%NOTFOUND的值是NULL。当该DML语句被执行且没有改变任何记录时，或者SELECT INTO语句没有返回任何记录时，SQL%FOUND的值为TRUE，否则为FALSE。

3. SQL%ISOPEN属性

SQL%ISOPEN表示隐式游标是否处于Open状态。**对于隐式游标而言，SQL%ISOPEN的值永远是FASLE**。因为Oracle一旦执行完隐式游标对应的SQL语句后就会自动Close该隐式游标。

4. SQL%ROWCOUNT属性

SQL%ROWCOUNT表示一条SQL语句成功执行后受其影响而改变的记录的数量。SQL%ROWCOUNT也适用于执行INSERT、UPDATE、DELETE的DML语句，以及SELECT INTO语句。

当与DML语句联合使用时，SQL%ROWCOUNT表示该DML语句成功执行后改变的的记录数。如果DML语句没有改变任何记录，或者SELECT INTO语句没有返回任何记录时，SQL%ROWCOUNT的值为0。当SELECT INTO返回超过一条以上的记录时，Oracle会报错**TOO_MANY_ROWS**，此时SQL%ROWCOUNT的返回值是**1**，而不是SELECT INTO语句实际查询到的记录数。

```sql
--在PL/SQL中使用SQL%ROWCOUNT
declare
  dept_no number(4) := 50;
begin
  delete from dept where deptno=dept_no;
  dbms_output.put_line('Number of departments deleted: ' || to_char(sql%rowcount));
  commit;
end;
/
```

>:star:SQL%ROWCOUNT的值仅表示**最近一次**执行的SQL语句的SQL%ROWCOUNT值，后续执行的SQL的SQL%ROWCOUNT值会**覆盖**之前执行的SQL语句对应的SQL%ROWCOUNT值。如果后续要用到某条SQL的SQL%ROWCOUNT值，那么在该SQL执行完后就要把它的SQL%ROWCOUNT值保存到一个变量里。


## 显式游标
显示游标是另一种session cursor，通常用于PL/SQL代码（存储过程、函数、package包）。显示游标的定义和生命周期中的**Open、Fetch和Close**均在PL/SQL代码中显式地控制。

Oracle中的显示游标具有以下四种常用属性（其中CURSORNAME为自定义的显示游标名称）：

- `CURSORNAME%FOUND`
- `CURSORNAME%NOTFOUND`
- `CURSORNAME%ISOPEN`
- `CURSORNAME%ROWCOUNT`

1. CURSORNAME%FOUND属性

表示指定的显示游标是否至少有一条记录被Fetch了。当一个显示游标被Open以后，如果还一次都没有被Fetch，那么CURSORNAME%FOUND的值是NULL。当该显示游标被Fetch后，CURSORNAME%FOUND的值为TRUE，**直到全部Fetch完毕**。全部Fetch完毕后，如果再执行一次Fetch操作，Oracle不会报错，但是CURSORNAME%FOUND的值为FALSE。

如果一个显示游标还没有被Open，此时如果试图使用CURSORNAME%FOUND属性，则Oracle会报错**INVALID_CURSOR**。

```sql
declare
  cusrsor c1 is select ename,sal from emp where rownum<11;
  my_ename emp.ename%type;
  my_salary emp.sal%type;
begin
  open c1;     --//手动打开游标
  loop
    fetch c1 into my_ename,my_salary;    --//手动fetch游标
    if c1%found then
      dbms_output.put_line('name=' || my_ename || ',salary=' || my_salary);
    else
      exit;
    end if;
  end loop;
  close c1;      --//手动关闭游标
end;
/
```

2. CURSORNAME%NOTFOUND属性

表示指定的显示游标是否已经Fetch完毕了。当一个显示游标被Open以后，如果还一次都没有被Fetch，那么CURSORNAME%NOTFOUND的值是NULL。与CURSORNAME%FOUND属性刚好相反，当该显示游标被Fetch后，CURSORNAME%NOTFOUND的值为**FALSE**，直到全部Fetch完毕。全部Fetch完毕后，如果再执行一次Fetch操作，Oracle不会报错，但是CURSORNAME%NOTFOUND的值为TRUE。

如果一个显示游标还没有被Open，此时如果试图使用CURSORNAME%NOTFOUND属性，则Oracle同样会报错**INVALID_CURSOR**。

3. CURSORNAME%ISOPEN属性

表示指定的显示游标是否被Open了。该属性通常被用于标准Exception处理流程中，用于Close那些发生了Exception而导致显示游标没有正常关闭的情形。

4. CURSORNAME%ROWCOUNT属性

表示指定的显示游标迄今为止一共Fetch了多少行记录。当一个显示游标被Open以后，如果还一次都没有被Fetch，那么CURSORNAME%ROWCOUNT的值是0。当一个显示游标被Open以后，如果Fetch后返回的结果是空的，那么CURSORNAME%ROWCOUNT的值也是0。只要这个显示游标被Fetch过一次并且返回的结果集不为空，那么CURSORNAME%ROWCOUNT的值就是迄今为止Fetch得到的行记录的数目。

如果一个显示游标还没有被Open，此时如果试图使用CURSORNAME%ROWCOUNT属性，则Oracle同样会报错**INVALID_CURSOR**。

>:snake:对于显示游标，需要注意以下两点：
>
>- 显示游标的标准用法是先Open，再Fetch，然后用一个while循环逐条处理数据，最后Close。
>- 在while循环内部处理完一条记录后，一定要记得执行Fetch操作来跳到下一条记录，否则就是死循环。


## 参考游标
和显示游标类似，参考游标也通常用于PL/SQL代码（存储过程、函数、package包）。参考游标的定义和生命周期中的**Open、Fetch和Close**也都是在PL/SQL代码中显式地控制。

Oracle中的参考游标也具有以下四种常用属性（其中CURSORNAME为自定义的参考游标名称）：

- `CURSORNAME%FOUND`
- `CURSORNAME%NOTFOUND`
- `CURSORNAME%ISOPEN`
- `CURSORNAME%ROWCOUNT`

参考游标的这四种属性的定义与显示游标的是一样的。

参考游标是Oracle数据库中最灵活的一种session cursor，主要体现在以下三个方面：

1. 参考游标的定义方式非常灵活，具有多种定义方式。

```sql
--定义一
type typ_cur_emp is ref cursor return emp%rowtype;
cur_emp typ_cur_emp;

--定义二
type typ_result is record(ename emp.ename%type, sql emp.sal%type);
type typ_cur_strong is ref cursor return type_result;
cur_emp typ_cur_strong;

--定义三
type typ_cur_weak is ref cursor;
cur_emp typ_cur_weak;

--定义四
cur_emp SYS_REFCURSOR:
```

2. 参考游标的Open方式也非常灵活，它可以不和某个固定的SQL绑定在一起。这意味着参考游标可以随时Open，并且每次Open所对应的SQL语句都可以不一样。

```sql
create package pck_refcoursor_open_demo as
  type gencurtyp is ref cursor;
  procedure open_cv (generic_cv in out gencurtyp, choice int);
end pck_refcoursor_open_demo;
/

create package body pck_refcoursor_open_demo as
  procedure open_cv (generic_cv in out gencurtyp, choice int) is
  begin
    --//三次Open操作中generic_cv对应的SQL语句都是不同的
    if choice = 1 then
      open generic_cv for select * from emp;
    elsif choice = 2 then
      open generic_cv for select * from dept;
    elsif choice = 3 then
      open generic_cv for select * from dept_temp;
    end if;
  end;
end pck_refcoursor_open_demo;
/
```

3. 参考游标可以作为存储过程的输入参数和函数的输出参数。

```sql
declare
  type typ_cur_emp is ref cursor return emp%rowtype;
  cur_emp typ_cur_emp;

  --//定义一个以参考游标为输入的存储过程
  procedure process_emp_cv (emp_cv in typ_cur_emp) is
    person emp%rowtype;
  begin
    dbms_output.put_line('-----');
    loop
      fetch emp_cv into person;
      exit when emp_cv%notfound;
      dbms_output.put_line('name = ' || person.ename);
    end loop;
  end;

begin
  open cur_emp for select * from emp where rownum<11;
  process_emp_cv(cur_emp);
  close cur_emp;

  open cur_emp for select * from emp where ename like 'C%';
  process_emp_cv(cur_emp);
  close cur_emp;
end;
/
```






