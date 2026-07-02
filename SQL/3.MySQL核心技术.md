## 查询
### 教学内容
+ 多表连接查询：INNER JOIN、LEFT JOIN、RIGHT JOIN、自连接
+ 聚合查询与分组：GROUP BY + HAVING
+ 嵌套查询：IN、EXISTS、ANY/ALL
+ 集合操作：UNION



### 实战查询
查询1：查询所有预约的完整信息（预约人 + 会议室）

```sql
-- ============================================================
-- 查询1：查询所有预约的完整信息（预约人 + 会议室）
-- ============================================================
SELECT 
    r.reservation_id,
    e.name AS employee_name,
    e.department,
    mr.room_name,
    mr.floor,
    r.subject,
    r.start_time,
    r.end_time,
    r.status
FROM reservation r
INNER JOIN employee e ON r.employee_id = e.employee_id
INNER JOIN meeting_room mr ON r.room_id = mr.room_id
ORDER BY r.start_time DESC;
```



查询2：查询某员工（张三）的所有预约

```sql
-- ============================================================
-- 查询2：查询某员工（张三）的所有预约
-- ============================================================
SELECT 
    mr.room_name,
    r.subject,
    r.start_time,
    r.end_time,
    r.status
FROM reservation r
INNER JOIN meeting_room mr ON r.room_id = mr.room_id
INNER JOIN employee e ON r.employee_id = e.employee_id
WHERE e.name = '张三'
ORDER BY r.start_time DESC;
```



查询3：统计每个会议室的预约次数和使用总时长（已审批的）

```sql
-- ============================================================
-- 查询3：统计每个会议室的预约次数和使用总时长（已审批的）
-- ============================================================
SELECT 
    mr.room_name,
    COUNT(r.reservation_id) AS booking_count,
    SUM(TIMESTAMPDIFF(HOUR, r.start_time, r.end_time)) AS total_hours
FROM meeting_room mr
LEFT JOIN reservation r ON mr.room_id = r.room_id AND r.status = 'approved'
GROUP BY mr.room_id, mr.room_name
ORDER BY total_hours DESC;
```



查询4：查询空闲会议室（某时间段内未被预约的）

```sql
-- ============================================================
-- 查询4：查询空闲会议室（某时间段内未被预约的）
-- 示例：查询2026-06-23 14:00:00 到 16:00:00 空闲的会议室
-- ============================================================
SELECT mr.*
FROM meeting_room mr
WHERE mr.status = 'available'
  AND mr.room_id NOT IN (
    SELECT room_id 
    FROM reservation 
    WHERE status IN ('pending', 'approved')
      AND NOT (end_time <= '2026-06-23 14:00:00' OR start_time >= '2026-06-23 16:00:00')
  );
```



查询5：查询待审批的预约（管理员待办事项）

```sql
-- ============================================================
-- 查询5：查询待审批的预约（管理员待办事项）
-- ============================================================
SELECT 
    r.reservation_id,
    e.name AS applicant,
    e.department,
    mr.room_name,
    r.subject,
    r.start_time,
    r.end_time,
    TIMESTAMPDIFF(HOUR, r.created_at, NOW()) AS waiting_hours
FROM reservation r
INNER JOIN employee e ON r.employee_id = e.employee_id
INNER JOIN meeting_room mr ON r.room_id = mr.room_id
WHERE r.status = 'pending'
ORDER BY r.created_at ASC;
```



查询6：统计每个部门的使用情况（预约次数 TOP 5）

```sql
-- ============================================================
-- 查询6：统计每个部门的使用情况（预约次数 TOP 5）
-- ============================================================
SELECT 
    e.department,
    COUNT(r.reservation_id) AS booking_count,
    SUM(TIMESTAMPDIFF(HOUR, r.start_time, r.end_time)) AS total_hours
FROM employee e
INNER JOIN reservation r ON e.employee_id = r.employee_id
WHERE r.status = 'approved'
GROUP BY e.department
ORDER BY booking_count DESC
LIMIT 5;
```



查询7：查询与某预约冲突的其他预约（冲突检测）

```sql
-- ============================================================
-- 查询7：查询与某预约冲突的其他预约（冲突检测）
-- 示例：检测与预约ID=1的预约冲突的其它预约
-- ============================================================
SELECT 
    r2.reservation_id,
    e.name AS employee_name,
    r2.start_time,
    r2.end_time,
    r2.status
FROM reservation r1
INNER JOIN reservation r2 
    ON r1.room_id = r2.room_id 
    AND r1.reservation_id != r2.reservation_id
    AND r2.status IN ('pending', 'approved')
    AND r2.start_time < r1.end_time 
    AND r2.end_time > r1.start_time
CROSS JOIN employee e ON r2.employee_id = e.employee_id
WHERE r1.reservation_id = 1;
```



### 实操任务
任务1：根据选题设计不少于5个查询业务。

任务2：编写SQL语句实现查询业务。





## 索引
### 教学内容
+ B+树索引原理简介
+ 索引的创建与删除
+ 索引使用场景与代价
+ 使用`EXPLAIN`分析查询计划
+ 慢查询日志的开启与分析



### 索引优化
使用 EXPLAIN 分析查询计划

```sql
-- ============================================================
-- 使用 EXPLAIN 分析查询计划
-- ============================================================
EXPLAIN SELECT * FROM reservation 
WHERE room_id = 1 
  AND start_time >= '2026-06-23 00:00:00' 
  AND end_time <= '2026-06-23 23:59:59'
  AND status = 'approved';

-- 观察：type、possible_keys、key、rows、Extra 字段

```



**1. type（连接类型/访问类型）**

+ **含义**：表示MySQL在表中查找数据的方式，是衡量**查询性能优劣**的核心指标。性能从**最好到最差**的排序大致如下：

| 类型 | 含义 | 性能等级 |
| :--- | :--- | :--- |
| **system** | 表中只有一行数据（系统表），是const的特例 | 最好 |
| **const** | 通过主键（PRIMARY KEY）或唯一索引（UNIQUE）与常量值匹配，最多返回一行 | 极优 |
| **eq_ref** | 多表连接时，使用主键或唯一索引作为连接条件，对于前表的每一行，后表仅返回一行 | 优良 |
| **ref** | 使用普通非唯一索引，匹配到多行（常见于等值条件） | 良好 |
| **range** | 使用索引进行范围扫描（如 `BETWEEN`、`>`、`<`、`IN` 等） | 一般 |
| **index** | 全索引扫描（扫描整个索引树，不读数据行） | 差 |
| **ALL** | **全表扫描**（扫描所有数据行，没有使用索引） | 极差（需重点优化） |


**2.possible_keys**

+ **含义**：显示MySQL**可能会选择**用来执行本次查询的索引列表。这些是查询优化器在分析阶段认为可用的索引。
+ **重要提示**：它只是“候选名单”，并不代表最终一定使用。如果该字段为`NULL`，则说明没有可用的索引供优化器选择（通常意味着需要创建新索引）。



**3.key**

+ **含义**：显示MySQL在 `possible_keys` 列表中**实际决定使用**的那个索引名称。
+ **注意**：如果 `key` 显示的索引名不在 `possible_keys` 中，通常发生在**覆盖索引**场景（即查询所需数据全部在索引中，无需回表），优化器会直接选择覆盖索引，而不一定选择最优的筛选索引。
+ 如果 `key` 为 `NULL`，则表示本次查询走了全表扫描（`type` 通常也会是 `ALL`）。



**4. rows**

+ **含义**：MySQL**预估**需要读取的行数（注意：是一个估算值，并非精确值）。
+ **作用**：用于评估查询的“工作量”。该值越小，通常意味着查询效率越高。如果该值远大于表中的实际记录数，说明索引选择性较差。



**5. Extra**

+ **含义**：包含MySQL执行查询时的**额外辅助信息**，非常重要，常用来判断是否出现了性能陷阱。

| 常见Extra值 | 含义 | 对性能的影响 |
| :--- | :--- | :--- |
| **Using index** | **覆盖索引**，查询所需数据全部在索引中获取，无需回表读取数据行 | **优秀** |
| **Using where** | 使用了 `WHERE` 条件过滤数据，但该条件无法完全通过索引完成（需要回表） | 一般 |
| **Using index condition** | 使用了索引下推，部分过滤在索引层完成，减少回表次数 | 较好 |
| **Using where; Using index** | 既使用了覆盖索引，又用了WHERE过滤（通常是最优情况） | 优秀 |
| **Using filesort** | 需要额外排序操作，无法利用索引顺序（性能消耗较大） | **需优化** |
| **Using temporary** | 需要使用临时表（常见于 `GROUP BY` 或 `DISTINCT` 复杂查询） | **需重点优化** |
| **Using join buffer** | 连接查询时未使用索引，需要内存块进行循环匹配（即块嵌套循环连接） | **需优化** |


### 管理索引
```sql
-- ============================================================
-- 查看当前库中的所有索引
-- ============================================================
SELECT 
    TABLE_NAME AS '表名',
    INDEX_NAME AS '索引名',
    COLUMN_NAME AS '列名',
    NON_UNIQUE AS '是否允许重复(0=唯一)',
    SEQ_IN_INDEX AS '列在索引中的顺序'
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- ============================================================
-- 删除索引
-- ============================================================
DROP INDEX 索引名 ON 表名;
```





### 慢查询
**关于慢查询的说明：**在SQL开发和数据库运维中，慢查询（Slow Query） 是指执行时间超过预设阈值（例如1秒或3秒）的SQL语句。

它不是指SQL写错了（语法错误），而是指SQL执行得太慢，导致系统响应延迟、用户等待，甚至拖垮整个数据库服务。可以将其理解为数据库里的“交通拥堵”——虽然路（表）是通的，但车（数据）跑不起来。

```sql
-- ============================================================
-- 慢查询日志配置（MySQL 8.0）
-- ============================================================
-- 查看慢查询日志状态
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- 设置慢查询阈值为1秒（临时生效）
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log = ON;

-- 查看慢查询日志内容
-- 日志文件通常在 /var/log/mysql/mysql-slow.log 或 C:\ProgramData\MySQL\MySQL Server 8.0\Data\ 下
```



**分析导致慢查询的原因**

| 原因分类 | 具体表现 | 通俗理解 |
| :--- | :--- | :--- |
| **缺少有效索引** | WHERE条件字段未建索引；或索引虽建但被SQL写法“绕开”了（如对索引字段用了函数计算）。 | 相当于一本书没有目录，只能一页一页翻找。 |
| **查询数据量过大** | 没有加LIMIT分页，一次性SELECT * 扫描了千万行数据返回给应用。 | 搬家时不分批次，试图一趟把所有家当搬完，自然卡在半路。 |
| **表连接（JOIN）过多或不当** | 多张大数据表做笛卡尔积式的关联，且连接字段未索引。 | 把几个大型仓库的门同时打开，让搬运工来回乱窜找匹配项。 |
| **锁等待与资源争抢** | 查询被其他正在执行的大事务（UPDATE/INSERT）持有的行锁或表锁阻塞。 | 高速公路上发生了交通事故（长事务），后面的车（查询）全部被迫停下等待。 |
| **数据库参数配置不当** | 内存缓冲区（Buffer Pool）太小，导致频繁的磁盘I/O读取，而不是读取内存。 | 厨房太小，放不下常用食材，导致每次做菜都得现去楼下菜市场买。 |
| **硬件资源瓶颈** | CPU飙高、磁盘IOPS达到上限、内存不足触发SWAP交换。 | 路本身太窄（硬件差），车流量一大就必然瘫痪。 |


## 视图
### 教学内容
+ 视图的概念与作用
+ 创建视图：`CREATE VIEW ... AS SELECT ...`
+ 视图的可更新性限制



### 视图创建和使用
视图1：预约详情视图（简化常用连表查询）

```sql
-- ============================================================
-- 视图1：预约详情视图（简化常用连表查询）
-- ============================================================
CREATE VIEW v_reservation_detail AS
SELECT 
    r.reservation_id,
    r.subject,
    r.start_time,
    r.end_time,
    r.status AS reservation_status,
    r.created_at,
    r.approved_at,
    e.employee_id,
    e.name AS employee_name,
    e.department,
    e.email,
    mr.room_id,
    mr.room_name,
    mr.floor,
    mr.capacity,
    mr.equipment,
    c.check_in_time,
    c.actual_attendees
FROM reservation r
LEFT JOIN employee e ON r.employee_id = e.employee_id
LEFT JOIN meeting_room mr ON r.room_id = mr.room_id
LEFT JOIN check_in c ON r.reservation_id = c.reservation_id;

-- 使用视图查询（简化的查询语句）
SELECT * FROM v_reservation_detail 
WHERE start_time >= CURDATE() 
ORDER BY start_time;
```



视图2：今日预约视图

```sql
-- ============================================================
-- 视图2：今日预约视图
-- ============================================================
CREATE VIEW v_today_reservations AS
SELECT 
    room_name,
    FLOOR,
    SUBJECT,
    employee_name,
    start_time,
    end_time,
    reservation_status
FROM v_reservation_detail
WHERE DATE(start_time) = CURDATE()
ORDER BY start_time;

-- 使用视图
SELECT * FROM v_today_reservations;
```



视图3：会议室使用统计视图

```sql
CREATE VIEW v_room_statistics AS
SELECT 
    mr.room_id,
    mr.room_name,
    mr.floor,
    mr.capacity,
    COUNT(r.reservation_id) AS total_bookings,
    SUM(CASE WHEN r.status = 'approved' THEN 1 ELSE 0 END) AS approved_bookings,
    SUM(TIMESTAMPDIFF(HOUR, r.start_time, r.end_time)) AS total_usage_hours,
    AVG(c.actual_attendees) AS avg_actual_attendees
FROM meeting_room mr
LEFT JOIN reservation r ON mr.room_id = r.room_id
LEFT JOIN check_in c ON r.reservation_id = c.reservation_id
GROUP BY mr.room_id;

-- 使用视图
SELECT * FROM v_room_statistics
```



### 管理视图
```sql
-- ============================================================
-- 查看当前库中的所有视图
-- ============================================================
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.VIEWS 
WHERE TABLE_SCHEMA = DATABASE();

-- ============================================================
-- 删除视图
-- ============================================================
DROP VIEW [IF EXISTS] 视图名;
```



### 实操任务
任务1：基于项目选题，设计视图，实现简化查询

任务2：使用视图查询



## 存储过程
### 教学内容
+ 存储过程的定义与调用
+ 参数类型：IN、OUT、INOUT
+ 流程控制：IF、CASE、LOOP、WHILE
+ 游标的使用
+ 异常处理



### 创建和调用存储过程
预约会议室存储过程（带冲突检测）

```sql
-- ============================================================
-- 存储过程1：预约会议室（带冲突检测）
-- ============================================================
DELIMITER //

CREATE PROCEDURE sp_create_reservation(
    IN p_employee_id VARCHAR(20),
    IN p_room_id INT,
    IN p_subject VARCHAR(200),
    IN p_start_time DATETIME,
    IN p_end_time DATETIME,
    OUT p_result_code INT,      -- 0=成功, 1=时间无效, 2=会议室冲突, 3=会议室不存在/不可用
    OUT p_result_msg VARCHAR(100)
)
sp_body: BEGIN
    DECLARE v_room_status VARCHAR(20);
    DECLARE v_conflict_count INT;
    
    -- 1. 校验时间有效性
    IF p_start_time >= p_end_time THEN
        SET p_result_code = 1;
        SET p_result_msg = '开始时间必须早于结束时间';
        LEAVE sp_body;
    END IF;
    
    -- 2. 校验会议室是否存在且可用
    SELECT status INTO v_room_status 
    FROM meeting_room 
    WHERE room_id = p_room_id;
    
    IF v_room_status IS NULL THEN
        SET p_result_code = 3;
        SET p_result_msg = '会议室不存在';
        LEAVE sp_body;
    END IF;
    
    IF v_room_status != 'available' THEN
        SET p_result_code = 3;
        SET p_result_msg = '会议室当前不可用（状态：' || v_room_status || '）';
        LEAVE sp_body;
    END IF;
    
    -- 3. 检测时间冲突（同一会议室，时间段重叠，且预约状态为待审批或已审批）
    SELECT COUNT(*) INTO v_conflict_count
    FROM reservation
    WHERE room_id = p_room_id
      AND status IN ('pending', 'approved')
      AND start_time < p_end_time
      AND end_time > p_start_time;
    
    IF v_conflict_count > 0 THEN
        SET p_result_code = 2;
        SET p_result_msg = CONCAT('该时间段会议室已被预约，存在 ', v_conflict_count, ' 个冲突预约');
        LEAVE sp_body;
    END IF;
    
    -- 4. 插入预约记录
    INSERT INTO reservation (employee_id, room_id, subject, start_time, end_time, status)
    VALUES (p_employee_id, p_room_id, p_subject, p_start_time, p_end_time, 'pending');
    
    SET p_result_code = 0;
    SET p_result_msg = '预约成功，等待管理员审批';
    
END //

DELIMITER ;


-- ============================================================
-- 调用示例
-- ============================================================
CALL sp_create_reservation(
    'EMP002', 
    1, 
    '项目启动会议', 
    '2026-06-25 10:00:00', 
    '2026-06-25 12:00:00', 
    @code, 
    @msg
);
SELECT @code AS result_code, @msg AS result_message;
```



### 管理存储过程
```sql
-- ============================================================
-- 查看当前库中的所有存储过程
-- ============================================================
SELECT ROUTINE_NAME
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE() -- DATABASE() 函数返回当前连接的数据库名
  AND ROUTINE_TYPE = 'PROCEDURE';

-- ============================================================
-- 删除存储过程
-- ============================================================
DROP PROCEDURE [IF EXISTS] 存储过程名;
```



### 实操任务
任务1：分析小组选题项目中的业务，选择合适的业务，然后使用存储过程实现该业务



## 触发器
### 教学内容
+ 触发器的类型：BEFORE/AFTER INSERT/UPDATE/DELETE
+ 触发器的使用场景
+ 新旧值的引用：NEW、OLD
+ 触发器的管理与查看



### 创建触发器
触发器：预约状态变更时自动记录审批时间

```sql
-- ============================================================
-- 触发器：预约状态变更时自动记录审批时间
-- ============================================================
-- 创建新触发器
DELIMITER //

CREATE TRIGGER trg_reservation_before_update
BEFORE UPDATE ON reservation
FOR EACH ROW
BEGIN
    -- 当状态从 pending 变为 approved 时，自动设置 approved_at
    IF OLD.status = 'pending' AND NEW.status = 'approved' AND NEW.approved_at IS NULL THEN
        SET NEW.approved_at = NOW();
    END IF;
END //

DELIMITER ;
```



测试触发器

```sql
-- 查看测试前的数据
SELECT reservation_id, status, approved_at FROM reservation WHERE reservation_id = 1;

-- 执行审批更新
UPDATE reservation SET status = 'approved' WHERE reservation_id = 1;

-- 查看测试后的数据（验证 approved_at 是否被自动设置）
SELECT reservation_id, status, approved_at FROM reservation WHERE reservation_id = 1;
```



### 管理触发器
```sql
-- ============================================================
-- 查看触发器
-- ============================================================
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE, ACTION_STATEMENT
FROM INFORMATION_SCHEMA.TRIGGERS
WHERE TRIGGER_SCHEMA = DATABASE();

-- ============================================================
-- 删除触发器
-- ============================================================
DROP TRIGGER IF EXISTS 触发器名称;
```





### 实操任务
任务1：分析小组选题项目中的业务，选择合适的业务，然后使用触发器实现该业务



## 事务
### 教学内容
+ ACID特性
+ 事务控制语句
+ 隔离级别与并发问题
+ MySQL事件调度器（定时任务）



### 使用事务
事务：完整的预约 + 签到流程

```sql
-- ============================================================
-- 事务示例：完整的预约 + 签到流程
-- ============================================================
DELIMITER $$

CREATE PROCEDURE sp_create_approved_reservation(
    IN p_employee_id VARCHAR(20),
    IN p_room_id INT,
    IN p_subject VARCHAR(100),
    IN p_start_time DATETIME,
    IN p_end_time DATETIME,
    IN p_actual_attendees INT,
    OUT p_result_code INT,
    OUT p_reservation_id INT,
    OUT p_result_msg VARCHAR(200)
)
sp_main:  -- ✅ 添加块标签，用于LEAVE语句
BEGIN
    -- 声明变量
    DECLARE v_room_status VARCHAR(20);
    DECLARE v_conflict_count INT;
    DECLARE v_reservation_id INT DEFAULT 0;
    DECLARE v_now DATETIME;
    
    -- 声明异常退出处理器
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_result_code = -1;
        SET p_reservation_id = NULL;
        SET p_result_msg = '系统异常，操作失败';
    END;

    SET v_now = NOW();
    
    -- ============================================================
    -- 业务规则：预约创建时签到的前提条件（可选，如不需要可删除）
    -- ============================================================
    -- 会议必须已开始（允许提前30分钟签到）
    IF p_start_time > DATE_ADD(v_now, INTERVAL 30 MINUTE) THEN
        SET p_result_code = 4;
        SET p_reservation_id = NULL;
        SET p_result_msg = '会议尚未开始，请在会议开始前30分钟内签到';
        LEAVE sp_main;  -- ✅ 指定标签名
    END IF;
    
    -- 会议不能已结束（允许会议开始后1小时内签到）
    IF p_end_time < DATE_SUB(v_now, INTERVAL 1 HOUR) THEN
        SET p_result_code = 5;
        SET p_reservation_id = NULL;
        SET p_result_msg = '会议已结束，无法签到';
        LEAVE sp_main;  -- ✅ 指定标签名
    END IF;

    -- 开启事务
    START TRANSACTION;

    -- 1. 校验会议室存在且可用
    SELECT STATUS INTO v_room_status 
    FROM meeting_room 
    WHERE room_id = p_room_id;
    
    IF v_room_status IS NULL THEN
        SET p_result_code = 1;
        SET p_reservation_id = NULL;
        SET p_result_msg = '会议室不存在';
        ROLLBACK;
        LEAVE sp_main;  -- ✅ 指定标签名
    END IF;
    
    IF v_room_status != 'available' THEN
        SET p_result_code = 2;
        SET p_reservation_id = NULL;
        SET p_result_msg = CONCAT('会议室不可用，状态：', v_room_status);
        ROLLBACK;
        LEAVE sp_main;  -- ✅ 指定标签名
    END IF;
    
    -- 2. 检测时间冲突
    SELECT COUNT(*) INTO v_conflict_count
    FROM reservation
    WHERE room_id = p_room_id
      AND STATUS IN ('pending', 'approved')
      AND start_time < p_end_time
      AND end_time > p_start_time;
    
    IF v_conflict_count > 0 THEN
        SET p_result_code = 3;
        SET p_reservation_id = NULL;
        SET p_result_msg = CONCAT('时间段被占用，冲突数：', v_conflict_count);
        ROLLBACK;
        LEAVE sp_main;  -- ✅ 指定标签名
    END IF;
    
    -- 3. 插入预约（已审批）
    INSERT INTO reservation (
        employee_id, 
        room_id, 
        SUBJECT, 
        start_time, 
        end_time, 
        STATUS,
        approved_at
    ) VALUES (
        p_employee_id, 
        p_room_id, 
        p_subject, 
        p_start_time, 
        p_end_time, 
        'approved',
        NOW()
    );

    -- 4. 获取刚插入的预约自增 ID
    SET v_reservation_id = LAST_INSERT_ID();

    -- 5. 插入签到记录
    INSERT INTO check_in (
        reservation_id, 
        check_in_time, 
        actual_attendees
    ) VALUES (
        v_reservation_id, 
        NOW(),
        p_actual_attendees
    );

    -- 6. 提交事务
    COMMIT;

    -- 设置成功返回信息
    SET p_result_code = 0;
    SET p_reservation_id = v_reservation_id;
    SET p_result_msg = '预约及签到创建成功';

END sp_main  -- ✅ 可选：结束标签
$$

DELIMITER ;
```



测试正常场景（会议即将开始）

```sql
CALL sp_create_approved_reservation(
    'EMP001',
    1,
    '紧急会议',
    DATE_ADD(NOW(), INTERVAL 10 MINUTE),  -- 10分钟后开始
    DATE_ADD(NOW(), INTERVAL 1 HOUR),
    8,
    @code,
    @id,
    @msg
);

SELECT @code AS code, @id AS id, @msg AS msg;
```



测试冲突场景（同一会议室相同时间段）

```sql
CALL sp_create_approved_reservation(
    'EMP002',
    1,
    '另一个会议',
    DATE_ADD(NOW(), INTERVAL 10 MINUTE),
    DATE_ADD(NOW(), INTERVAL 1 HOUR),
    5,
    @code2,
    @id2,
    @msg2
);

SELECT @code2, @id2, @msg2;  -- 应该返回错误码3
```



查看预约和签到记录

```sql
SELECT 
    r.reservation_id,
    r.subject,
    r.start_time,
    r.end_time,
    r.status,
    r.approved_at,
    c.check_in_time,
    c.actual_attendees
FROM reservation r
LEFT JOIN check_in c ON r.reservation_id = c.reservation_id
ORDER BY r.reservation_id DESC
LIMIT 5;
```



### 实操任务
任务1：分析小组选题项目中的业务，找出需要在事务中执行的业务

任务2：编写程序，实现在事务中执行业务。



## 事件
### 教学内容
+ MySQL事件调度器（定时任务）



### 定义事件
MySQL事件调度器：每日凌晨自动更新过期预约状态

```sql
-- ============================================================
-- MySQL事件调度器：每日凌晨自动更新过期预约状态
-- ============================================================
CREATE EVENT ev_auto_complete_reservations
ON SCHEDULE EVERY 1 DAY
STARTS '2026-06-23 00:00:00'
DO
    UPDATE reservation 
    SET status = 'completed' 
    WHERE status IN ('approved') 
      AND end_time < NOW();
```



### 管理事件
```sql
-- 查看事件调度器是否开启
SHOW VARIABLES LIKE 'event_scheduler';

-- 开启事件调度器
SET GLOBAL event_scheduler = ON;

-- 查看当前数据库下的所有事件
SHOW EVENTS;

-- 如果当前数据库没有事件，可以用 FROM 指定要查看的数据库
SHOW EVENTS FROM 数据库名;

-- 删除事件
DROP EVENT [IF EXISTS] 事件名;
```



### 实操任务
任务1：分析小组选题项目中的业务，选择有定时特征的业务，然后事件实现定时任务。
