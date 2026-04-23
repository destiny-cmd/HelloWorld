# SQL 学习笔记 ——一个被SQL Server折磨过的人写的

## 前言

这份笔记记录了我在数据库课上折腾SQL的全过程。用的是SQL Server 2022 + SSMS 22，被折磨得够呛，但该学的还是学了。以后打算换MySQL + Navicat，SQL Server这个大块头真的不适合学生用。

---

## 一、环境那些破事

### 启动方式（每次开机的固定仪式）

1. `Win + R` → `services.msc`
2. 找 **SQL Server (MSSQLSERVER)** → 右键启动
3. 等它变绿
4. 再开 SSMS

> 评价：纯唐。摇签求签，运气好一次到位，运气不好关掉重来。825MB内存就为了写几条SQL，微软你是认真的吗。

### 血泪教训：永远不要手动移动SQL Server的安装文件夹

把文件夹从 `C:\FOR WORKING\` 移到 `C:\Do\` 之后，服务直接启动失败。注册表里记录路径的地方多达十几处，包括：

- `HKLM\...\Parameters` 里的 SQLArg0/1/2（主程序、日志、master数据库路径）
- `HKLM\...\Setup` 里的 SQLPath、SQLProgramDir、SQLDataRoot、SQLBinRoot
- `HKLM\...\MSSQLServer` 里的 BackupDirectory

改了一个小时还没改完，最后直接卸载重装。**结论：装好之后别动，路径选默认。**

---

## 二、SQL Server的坑

### 中文表名必须加方括号

```sql
-- 死路
SELECT * FROM 教师;
-- 活路
SELECT * FROM [教师];
SELECT t.[教工号] FROM [教师] t;
```

### char类型会偷偷补空格

`char(6)` 存 `'张宏'` 实际是 `'张宏 '`，查询时加 `RTRIM` 去空格：

```sql
WHERE RTRIM([姓名]) = '张宏'
```

### 和MySQL的主要差异

|功能|SQL Server|MySQL|
|---|---|---|
|取前N条|`TOP 3`|`LIMIT 3`|
|建表并插入|`SELECT * INTO 新表 FROM 旧表`|`CREATE TABLE 新表 AS SELECT`|
|标识符引用|`[]`|` `` `|
|当前时间|`GETDATE()`|`NOW()`|

### "编辑前200行"别用

时不时卡死，直接写查询语句验证：

```sql
SELECT * FROM [表名];
```

### 限制内存（救命用）

```sql
EXEC sp_configure 'max server memory (MB)', 1024;
RECONFIGURE;
```

---

## 三、INSERT 插入

### 基本插入

```sql
INSERT INTO [教师] ([教工号], [姓名], [性别], [职称], [学院名称], [聘任时间])
VALUES ('0001', '张明', '男', '教授', '电信学院', '2010-09-01');
```

### 查询结果存新表（SELECT INTO）

```sql
-- 自动建表
SELECT * INTO [男教师表]
FROM [教师]
WHERE [性别] = '男';

-- 带统计
SELECT [性别], COUNT(*) AS [人数] INTO [男女教师人数表]
FROM [教师]
GROUP BY [性别];
```

### 查询结果插入已有表

```sql
INSERT INTO [男教师表]
SELECT * FROM [教师] WHERE [性别] = '男';
```

### 批量插入（从另一张表筛选）

```sql
-- 给某专业学生批量开课，成绩暂空
INSERT INTO [选课] ([学号], [课程编号], [成绩])
SELECT [学号], '0005', NULL
FROM [学生]
WHERE [专业编号] = '0001';
```

---

## 四、DELETE 删除

### 基本删除

```sql
-- 姓张的全删
DELETE FROM [教师] WHERE RTRIM([姓名]) LIKE '张%';

-- 最后一个字是杨的
DELETE FROM [教师] WHERE RTRIM([姓名]) LIKE '%杨';

-- 姓名第二个字是明的
DELETE FROM [教师] WHERE SUBSTRING(RTRIM([姓名]), 2, 1) = '明';
```

### 子查询删除

```sql
-- 删除没有授课的教师
DELETE FROM [教师]
WHERE [教工号] NOT IN (SELECT [教工号] FROM [授课]);

-- 删除没有选课的学生
DELETE FROM [学生]
WHERE [学号] NOT IN (SELECT [学号] FROM [选课]);
```

### 关联删除

```sql
-- 删除体育学院教师的授课记录
DELETE FROM [授课表1]
WHERE [教工号] IN (
    SELECT [教工号] FROM [教师]
    WHERE [学院名称] LIKE '%体育%'
);
```

### 恢复删除的数据

```sql
-- 删了之后后悔了，从原表重新插回来
INSERT INTO [授课表1]
SELECT * FROM [授课]
WHERE [教工号] IN (
    SELECT [教工号] FROM [教师]
    WHERE [学院名称] LIKE '%体育%'
);
```

---

## 五、UPDATE 修改

### 基本修改

```sql
-- 改名字
UPDATE [教师]
SET [姓名] = '刘莉'
WHERE RTRIM([姓名]) = '刘丽';

-- 不及格加5分
UPDATE [选课]
SET [成绩] = [成绩] + 5
WHERE [成绩] < 60;

-- 45到60分之间加15分
UPDATE [选课]
SET [成绩] = [成绩] + 15
WHERE [成绩] BETWEEN 45 AND 60;
```

### 子查询修改

```sql
-- 改为该课平均分
UPDATE [选课]
SET [成绩] = (
    SELECT AVG([成绩]) FROM [选课]
    WHERE [课程编号] = '0002'
)
WHERE [学号] = '0001' AND [课程编号] = '0002';

-- 跨表条件修改
UPDATE [授课]
SET [上课教室] = '1109'
WHERE [教工号] = (
    SELECT [教工号] FROM [教师] WHERE RTRIM([姓名]) = '张明'
)
AND [课程编号] = (
    SELECT [课程编号] FROM [课程] WHERE [课程名称] LIKE '%数据库%'
);
```

### 修改姓名中的某个字

```sql
-- 把姓李的学生名字第2个字改为"雷"
UPDATE [学生]
SET [姓名] = LEFT(RTRIM([姓名]), 1) + '雷' + SUBSTRING(RTRIM([姓名]), 3, LEN(RTRIM([姓名]))-2)
WHERE RTRIM([姓名]) LIKE '李%';

-- 把姓王的教师名字第3个字改为"军"
UPDATE [教师]
SET [姓名] = LEFT(RTRIM([姓名]), 2) + '军'
WHERE RTRIM([姓名]) LIKE '王%';
```

### TOP限制修改行数

```sql
-- 前三位教师职称改为讲师
UPDATE [教师]
SET [职称] = '讲师'
WHERE [教工号] IN (
    SELECT TOP 3 [教工号] FROM [教师] ORDER BY [教工号]
);
```

### 字段置空

```sql
UPDATE [授课]
SET [上课教室] = ''
WHERE [教工号] = (
    SELECT [教工号] FROM [教师] WHERE RTRIM([姓名]) = '王兰'
);
```

---

## 六、常用字符串函数

|函数|用途|例子|
|---|---|---|
|`RTRIM()`|去右侧空格|`RTRIM([姓名])`|
|`LEFT(str, n)`|取左边n个字|`LEFT([姓名], 1)` 取姓|
|`RIGHT(str, n)`|取右边n个字|`RIGHT(RTRIM([姓名]), 1)` 取最后一字|
|`SUBSTRING(str, start, len)`|截取字符串|`SUBSTRING([姓名], 2, 1)` 取第2个字|
|`LEN()`|字符串长度|`LEN(RTRIM([姓名]))`|
|`YEAR()`|取年份|`YEAR([聘任时间])`|

---

## 七、总结

SQL本身不难，难的是环境。SQL Server这个东西：

- 内存吃得多，启动玄学
- 中文支持差，到处要加`[]`
- char类型补空格，查询要RTRIM
- 文件夹绝对不能乱移，注册表路径一大堆

下次直接上 **MySQL 8.0 + Navicat**，轻量、好用、不折腾。

这门课学完最大的收获不是SQL语法，而是：**数据库装好之后，路径选默认，别乱动。**

---

_写于2026年4月，被SQL Server折磨后的幸存者记录_