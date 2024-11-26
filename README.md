# 資料庫測驗

## 題目一

1. 使用Limit和Join來解決

```mysql
SELECT bnb.id             AS bnb_id,
       bnb.name           AS bnb_name,
       SUM(orders.amount) AS may_amount
FROM orders
         JOIN
     rooms ON orders.room_id = rooms.id
         JOIN
     bnbs AS bnb ON rooms.bnb_id = bnb.id
WHERE orders.currency = 'TWD'
  AND orders.check_in_date >= '2023-05-01'
  AND orders.check_out_date <= '2023-05-31'
GROUP BY bnb.id, bnb.name
ORDER BY may_amount DESC
LIMIT 10;
```

### 優點

- 容易理解 使用`JOIN`和`GroupBy`以及`OrderBy`，相對較容易理解。
- 可擴展 可以輕鬆調整查詢條件（例如改變月份或貨幣類型）。

### 缺點

- 效能較差 由於`ORDER BY`和`LIMIT`是在`GROUP BY`之後執行，因此會導致效能較差。
- 依賴索引 查詢效率相當依賴於適合的索引設置，否則可能執行速度較慢。

### 適合場景

- 數據量較小且查詢頻率不高的報表查詢。
- 已有建立良好的索引，且數據量規模不大的情況下。

2. 使用Window Function來解決

```mysql
WITH RankedData AS (SELECT bnb.id                                               AS bnb_id,
                           bnb.name                                             AS bnb_name,
                           SUM(orders.amount)                                   AS may_amount,
                           ROW_NUMBER() OVER (ORDER BY SUM(orders.amount) DESC) AS row_num
                    FROM orders
                             JOIN
                         rooms ON orders.room_id = rooms.id
                             JOIN
                         bnbs AS bnb ON rooms.bnb_id = bnb.id
                    WHERE orders.currency = 'TWD'
                      AND orders.check_in_date >= '2023-05-01'
                      AND orders.check_out_date <= '2023-05-31'
                    GROUP BY bnb.id, bnb.name)
SELECT bnb_id,
       bnb_name,
       may_amount
FROM RankedData
WHERE row_num <= 10;
```

### 優點

- 彈性更高
    - 可以在不影響主查詢結構的情況下計算聚合數據。
    - 不強制分組，因此可以同時返回明細和聚合數據。
- 性能優化
    - 視窗函數僅在計算分區內的數據時生效，能避免整體表的多次掃描（依賴於數據庫優化器）。
    - 如果有適當的索引（如 bnb_id 或日期篩選），效率更高。
- 易於擴展
    - 可以輕鬆加入其他窗口函數，例如排序型的（`RANK()`或`ROW_NUMBER()`）或其他統計數據（如`AVG`或 `Count`以及`Max`等等）。

### 缺點

- 索引依賴性 與一般查詢時會有依賴索引的問題
- 執行效率 對於非常大規模的數據集（如數億行），Window Function 的計算開銷可能會比較大，特別是在沒有分區表或索引輔助的情況下。
- 學習成本 Window Function相較於普通聚合的語法複雜度略高，學習門檻較為陡峭。

### 適合場景

- 分組排序的需求 例如Top N的查詢。
- 詳細查詢的需求 例如同時返回明細（如同時查看每個訂單和總金額）和聚合數據。
- 即席分析(Ad Hoc queries)的需求 適用於開發或分析環境，方便快速提取數據進行統計。

3. 使用Subquery來解決

```mysql
SELECT b.id   AS bnb_id,
       b.name AS bnb_name,
       o.may_amount
FROM bnbs b
         JOIN (SELECT bnb_id,
                      SUM(amount) AS may_amount
               FROM orders
               WHERE currency = 'TWD'
                 AND check_in_date >= '2023-05-01'
                 AND check_out_date < '2023-06-01'
               GROUP BY bnb_id) o ON b.id = o.bnb_id
ORDER BY o.may_amount DESC
LIMIT 10;

```

### 優點

- 簡單易懂 將篩選條件和計算邏輯集中在子查詢，讓主查詢只負責最終的資料連接和排序。
- 執行效率較佳 如果子查詢結果小於原始表規模，可以減少 JOIN 時的數據處理量。

### 缺點

- 可讀性較差 子查詢的結果不容易理解，對於複雜的查詢可能會增加閱讀和維護的難度。
- 可擴展性較差 子查詢的結果無法直接用於其他查詢，如果需要在其他地方使用，可能需要重複定義子查詢。
- 效能問題 如果子查詢的結果集過大，可能會導致效能問題。
- 不適用頻繁查詢 不適用於頻繁查詢的場景，因為每次查詢都需要重新計算子查詢。

### 適合場景

- 輕量級報表查詢 對於輕量級的報表查詢，不需要太多的數據處理和計算的場景。
- 中小型數據集 對於中小型數據集，子查詢結果集不會過大的場景。
- 單次查詢 使用一次性查詢的場景，不需要在其他地方重複使用子查詢結果。

4. 使用temporary table來解決
   ``建立臨時表``

```mysql
CREATE TEMPORARY TABLE temp_may_orders AS
SELECT bnb_id,
       SUM(amount) AS may_amount
FROM orders
WHERE currency = 'TWD'
  AND check_in_date >= '2023-05-01'
  AND check_out_date < '2023-06-01'
GROUP BY bnb_id;
```

``查詢時透過Limit來限制``

```mysql
SELECT b.id   AS bnb_id,
       b.name AS bnb_name,
       t.may_amount
FROM bnbs b
         JOIN
     temp_may_orders t ON b.id = t.bnb_id
ORDER BY t.may_amount DESC
LIMIT 10;
```

### 優點

- 減少重複計算 臨時表只需創建一次，後續可多次使用。
- 靈活性高 適合臨時的數據處理需求，無需永久改變數據庫結構。

### 缺點

- 生命週期較為短暫 臨時表僅在當前連線中有效。
- 額外存儲 需要額外的存儲空間或內存來保存臨時表的數據。

### 適合場景

- 臨時分析需求 例如即席查詢或一次性報表生成。
- 快速數據處理 適用於批量數據處理或中間結果緩存。

5. 使用分表來解決

``建立分表``

```mysql
CREATE TABLE orders
(
    id             INT NOT NULL,
    bnb_id         INT,
    room_id        INT,
    currency       CHAR(3),
    amount         INT,
    check_in_date  DATE,
    check_out_date DATE,
    created_at     TIMESTAMP
) PARTITION BY RANGE
(
    YEAR
(
    check_in_date
), MONTH
(
    check_in_date
))
(
    PARTITION
    p2023_05
    VALUES
    LESS
    THAN
(
    2023,
    6
),
    PARTITION p2023_06 VALUES LESS THAN
(
    2023,
    7
),
    PARTITION p_max VALUES LESS THAN MAXVALUE
    );
```

``查詢用法``

```mysql
SELECT b.id          AS bnb_id,
       b.name        AS bnb_name,
       SUM(o.amount) AS may_amount
FROM bnbs b
         JOIN
     orders o ON b.id = o.bnb_id
WHERE o.currency = 'TWD'
  AND o.check_in_date >= '2023-05-01'
  AND o.check_out_date < '2023-06-01'
GROUP BY b.id, b.name
ORDER BY may_amount DESC
LIMIT 10;
```

### 優點

- 高效篩選 查詢僅針對符合條件的分區進行處理，顯著提升篩選速度。
- 適合大數據 有效降低大表的查詢和維護成本

### 缺點

- 管理成本高 需要規劃分區策略，另外分表需要定期維護，包括分區管理、數據遷移等。
- 存儲成本增加 分區表會增加存儲空間的使用，特別是對於小數據量的表格，可能會浪費空間。

### 適合場景

- 大數據量 例如每日交易記錄、歷史日誌查詢。
- 篩選條件明確 例如按日期、地區等明確篩選條件的查詢。

6. 採用物化檢視（Materialized View）來解決

``建立物化檢視``

```mysql
CREATE
MATERIALIZED VIEW bnb_monthly_revenue AS
SELECT bnb_id,
       SUM(amount)                         AS may_amount,
       DATE_FORMAT(check_in_date, '%Y-%m') AS month
FROM orders
WHERE currency = 'TWD'
GROUP BY bnb_id, DATE_FORMAT(check_in_date, '%Y-%m');
```

``查詢物化檢視``

```mysql
SELECT b.id   AS bnb_id,
       b.name AS bnb_name,
       m.may_amount
FROM bnbs b
         JOIN
     bnb_monthly_revenue m ON b.id = m.bnb_id
WHERE m.month = '2023-05'
ORDER BY m.may_amount DESC
LIMIT 10;
```

### 優點

- 高效查詢 物化檢視將計算結果存儲在表中，提高查詢效率。
- 易於維護 物化檢視的更新頻率可以靈活設定（例如每日或按需更新）。

### 缺點

- 存儲需求：需要額外的磁碟空間存儲檢視結果。
- 更新成本：需要考慮更新機制，對於頻繁變化的數據可能增加系統負擔。

### 適合場景

- 頻繁查詢 適用於固定的報表查詢，且查詢需求不頻繁變更。
- 歷史數據查詢 如每月營收、年度報表。
