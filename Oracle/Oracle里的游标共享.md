---
tags: [oracle, SQL优化]
title: Oracle里的游标共享
created: '2022-10-20T07:20:28.890Z'
modified: '2022-10-20T13:24:53.971Z'
---

Oracle里的游标共享
游标共享是指shared cursor间的共享，即重用存储在child cursor里的解析树和执行计划。

**场景一**：很多OLTP类型的应用系统的开发人员在开发阶段并未意识到硬解析的危害，所以没有使用绑定变量，等到系统上线后才发现问题。此时如果要使用绑定变量，意味着绝大多数SQL都得改写，这个代价会很大。

**场景二**：即使应用系统在开发阶段使用了绑定变量，但在默认情况下也会受到绑定变量窥探的影响。其副作用在于，一旦启用了绑定变量窥探，使用了绑定变量的目标SQL就只会沿用之前硬解析时所产生的解析树和执行计划，即使这种做法完全不适合当前的情形。

为了解决以上两种场景对应的问题，Oracle引入了常规游标共享和自适应游标共享。前者用于解决第一种场景对应的问题，后者用于解决第二种场景对应的问题。

# 常规游标共享
当开启常规游标共享后，Oracle在实际解析目标SQL之前，会先用系统产生的绑定变量来替换SQL文本中**where条件或者values子句**（INSERT语句）中的具体输入值。Oracle里系统产生的绑定变量的命名规则是`SYS_B_n`，其中`n=0,1,2,...`。

常规游标共享受参数`CURSOR_SHARING`的控制，其取值可以是EXACT、SIMILAR、FORCE。

1. **EXACT**

EXACT是`CURSOR_SHARING`参数的默认值，表示**不**会用系统产生的绑定变量来替换目标SQL文本中where条件或者values子句中的具体输入值。

2. **SIMILAR**

当`CURSOR_SHARING=SIMILAR`时，Oracle会用系统产生的绑定变量来替换目标SQL文本中where条件或者values子句中的具体输入值，但不一定会重用解析树和执行计划。对于不安全的谓词条件，即便用系统产生的绑定变量替换后的SQL文本是一模一样的，对于每一个不同的输入值，Oracle都会执行一次硬解析。这还可能会导致一个父游标下存在多个解析树和执行计划完全相同的子游标。

>:eagle:**注**：由于相关bug太多，Oracle 12c及后续版本中SIMILAR将过时，不再被继续支持。

>:moon:**安全的谓词条件**，是指如果一个谓词条件所在的目标SQL的**执行计划并不随该谓词条件的输入值的变化而变化**。比如，主键列上等值查询的谓词条件就是安全的谓词条件。Oracle中典型的不安全的谓词条件有范围查询（>、>=、<、<=、between）、使用了带通配符`%`的like、以及在有直方图统计信息的目标列上的等值查询等。

3. **FORCE**

当`CURSOR_SHARING=FORCE`时，Oracle会用系统产生的绑定变量来替换目标SQL文本中where条件或者values子句中的具体输入值，并且一定会无条件重用之前硬解析时生成的解析树和执行计划（由于自适应游标共享的引入，这里描述的行为不再适用于Oracle 11g及其后续版本）。

# 自适应游标共享
绑定变量窥探的副作用在于，使用了绑定变量的目标SQL只会沿用之前硬解析时所产生的解析树和执行计划，即使这种做法完全不适合当前的情形。在Oracle 10g及其后续版本中，Oracle会自动收集直方图统计信息，这意味着Oracle有更大的概率会知道目标列实际数据的分布情况，也就是说绑定变量窥探的副作用将会更加明显。

为了解决绑定变量窥探带来的问题，Oracle 11g中引入了自适应游标共享（Adaptive Cursor Sharing）。自适应游标共享可以让使用了绑定变量的目标SQL在启用了绑定变量窥探的前提条件下，不再只沿用之前硬解析时所产生的解析树和执行计划，而是可以在多个执行计划之间自适应地做出选择。Oracle只需要在它认为目标SQL的执行计划可能发生变化时，触发该SQL重新做一次硬解析就行。

具体来说，Oracle会根据执行目标SQL时所对应的**runtime统计信息**（比如耗费的逻辑读和CPU时间、对应结果集的行数等）的变化，以及当前传入的绑定变量输入值所在的谓词条件的**可选择率**，来综合判断是否需要触发目标SQL的硬解析动作。

## Bind Sensitive & Bind Aware
自适应游标共享要做的第一件事情就是扩展游标共享（Extended Cursor Sharing），主要就是将目标SQL对应的child cursor标记为Bind Sensitive。**Bind Sensitive**是指Oracle觉得某个包含绑定变量的目标SQL的执行计划**可能**会随着所传入的绑定变量输入值的变化而变化。

满足下面三个条件时，目标SQL对应的child cursor就会被标记为Bind Sensitive：

- 启用了绑定变量窥探；
- 目标SQL使用了绑定变量（可以是SQL自带的绑定变量、也可以是开启常规游标共享后系统自动生成的绑定变量）；
- 目标SQL使用的是不安全的谓词条件。

自适应游标共享要做的第二件事情就是将目标SQL对应的child cursor标记为Bind Aware。**Bind Aware**是指Oracle已经**确定**某个包含绑定变量的目标SQL的执行计划会随着所传入的绑定变量输入值的变化而变化。

满足下面两个条件时，目标SQL对应的child cursor就会被标记为Bind Aware：

- 目标SQL对应的child cursor之前已经被标记为Bind Sensitive；
- 目标SQL在接下来**连续两次**执行时，所对应的runtime统计信息与之前硬解析时的runtime统计信息存在较大差异。

对于自适应游标共享而言，`V$SQL`中的`IS_BIND_SENSITIVE`、`IS_BIND_AWARE`和`IS_SHAREABLE`分别表示child cursor是否Bind Sensitive、Bind Aware和可共享的。这里“可共享”是指其解析树和执行计划能被重用。

与自适应游标共享相关的有两个重要视图：

- `V$SQL_CS_STATISTICS`用于显式指定child cursor中存储的runtime统计信息。
- `V$SQL_CS_SELECTIVITY`用于显示指定的、已经被标记为**Bind Aware**的child cursor中存储的包含绑定变量的谓词条件所对应的可选择率的范围。

当一个被标记为Bind Aware的child cursor所对应的目标SQL再次执行时，Oracle就会比较当前传入的绑定变量值所在的谓词条件的可选择率，以及该SQL之前硬解析时同名谓词条件在`V$SQL_CS_SELECTIVITY`中对应的可选择率的范围，并以此来决定当前的执行是用硬解析还是软解析。

## 自适应游标共享的执行流程
Oracle数据库中自适应游标共享的整体执行流程如下：

1. 当目标SQL第一次被执行时，Oracle会使用**硬解析**，同时会根据一系列条件（比如SQL有没有使用绑定变量、参数CURSOR_SHARING的值、绑定变量所在的列是否有直方图、where条件是等值查询还是范围查询等）来判断是否将该SQL所对应的child cursor标记为**Bind Sensitive**。对于标记为Bind Sensitive的child cursor，Oracle会把执行该SQL时所对应的runtime统计信息额外存储到该SQL的child cursor中。

2. 当目标SQL被第二次执行时，Oracle会使用**软解析**，并且会重用该SQL第一次执行时所产生的child cursor中存储的解析树和执行计划。

3. 当目标SQL被第三次执行时，如果对应的child cursor已经被标记为Bind Sensitive，同时在第二次和第三次执行该SQL时所记录的runtime统计信息和第一次硬解析时所记录的runtime统计信息均存在较大差异，那么该SQL在第三次执行时就会使用**硬解析**。Oracle此时会产生一个新的child cursor，并且这个新的子游标会被标记为**Bind Aware**。

4. 对于标记为Bind Aware的child cursor所对应的目标SQL，当该SQL再次被执行时，Oracle就会根据当前传入的绑定变量值所对应的谓词条件的**可选择率**，来决定此时是用硬解析还是软解析。如果当前传入的的绑定变量值所在谓词条件的可选择率处于该SQL之前硬解析时同名谓词条件在`V$SQL_CS_SELECTIVITY`中记录的可选择率的范围之内，此时Oracle就会使用软解析（或者软软解析），并且重用相关child cursor中的解析树和执行计划，反之则是硬解析。

5. 如果是硬解析，并且本次硬解析产生的执行计划和原有的child cursor中的执行计划相同，则此时Oracle除了会生成一个新的child cursor之外，还会把存储相同执行计划的原有child cursor标记为非共享。在把原有child cursor标记为非共享的同时，Oracle还会**合并**存储了相同执行计划的原有child cursor和新生成的child cursor。


## 自适应游标共享的缺陷
自适应游标共享虽然在一定程度上缓解了绑定变量窥探带来的副作用，但也并不是完美的，它可能存在如下缺陷：

- 可能导致一定数量的硬解析。
- 可能导致一定数量的额外的child cursor挂在同一个parent cursor下。这会增加软解析时查找匹配child cursor的工作量，同时为了存储这些额外的child cursor，shared pool在空间方面也会承受额外的压力。

如果因为开启自适应游标共享而导致产生了过多的child cursor，进而导致shared pool的空间紧张或者过多的mutex等待，则可以通过以下任意一种方式来禁用自适应游标共享。

- 将隐含参数`_OPTIMIZER_EXTENDED_CURSOR_SHARING`和`_OPTIMIZER_EXTENDED_CURSOR_SHARING_REL`的值均设为NONE，这样就相当于关闭了可扩展游标共享。一旦可扩展游标共享被禁用，所有的child cursor就不能再被标记为Bind Sensitive。
- 将隐含参数`_OPTIMIZER_ADAPTIVE_CURSOR_SHARING`的值设置为FALSE。一旦该隐含参数被设置为FALSE，则所有的child cursor都将不能再被标记为Bind Aware（即使它们已经被标记为了Bind Sensitive）。

需要额外注意的是，自适应游标共享在Oracle **11g**中有一个硬限制：只有当目标SQL中绑定变量（不管是目标SQL自带的还是开启常规游标共享后系统生成的）的数量**不超过14个**时，自适应游标共享才会生效；因为一旦超过14个，该SQL对应的child cursor就永远不会被标记为Bind Sensitive。



