# 407630465

# 資料庫系統期末作業成果展示
> 題目：進銷存管理系統
* 姓名：詹翊辰
* 學號：407630465
* 系級：資管四Ａ

# 系統功能構想
本系統主題為「訂單 / 進銷存系統」，主要功能包含：

* 商品管理：維護商品資料（商品目錄）、商品分類及品牌
* 庫存管理：管理各平台（蝦皮、賣貨便、面交等）庫存量及總庫存
* 進貨管理：紀錄商品進貨明細，包括數量、成本、日期及幣值
* 出貨管理：紀錄商品出貨明細，扣減相應平台或總庫存，記錄訂單編號
* 報表統計：依據銷售、進貨、庫存資料生成業績與庫存報表
* 資料一致性：使用交易機制確保進銷存資料正確，避免庫存錯誤
* 自動更新：透過觸發器自動同步庫存數量與成本等關鍵欄位

# ER圖
![image](https://hackmd.io/_uploads/SJKZl8LNeg.png)
本系統實體包含：

* product（商品目錄）：商品基本資料、品牌與分類
* product_category（商品分類）：商品種類名稱
* brand（品牌廠商）：品牌名稱
* platform（平台）：蝦皮、賣貨便、面交等
* inventory_by_platform（平台庫存）：紀錄各商品在各平台的庫存數量
* purchase（進貨）：商品進貨明細，包括成本與幣值
* sales（出貨）：商品出貨明細，包含平台與訂單資料

實體間關係：

* product 與 product_category、brand 是多對一關係
* inventory_by_platform 連結 product 與 platform（多對多）
* purchase 與 product 是多對一
* sales 與 product 及 platform 是多對一

# 正規化

* 第一正規化（1NF）：所有資料欄位原子性，無重複群組，均分散在不同欄位
* 第二正規化（2NF）：非主鍵欄位完全依賴主鍵，避免部分依賴。inventory_by_platform 使用複合主鍵 (product_id, platform_id)，避免重複紀錄
* 第三正規化（3NF）：非主鍵欄位不依賴其他非主鍵欄位，例如 product 表中品牌及分類使用外鍵指向獨立表格，避免冗餘資訊

> 主鍵外鍵設計

* product:
    * 主鍵：product_id (varchar)
    * 外鍵：category_id → product_category(category_id)
    * 外鍵：brand_id → brand(brand_id)
* inventory_by_platform:
    * 複合主鍵：(product_id, platform_id)
    * 外鍵：product_id → product(product_id)
    * 外鍵：platform_id → platform(platform_id)
* purchase:
    * 主鍵：purchase_id (自增整數)
    * 外鍵：product_id → product(product_id)
* sales:
    * 主鍵：sale_id (自增整數)
    * 外鍵：product_id → product(product_id)
    * 外鍵：platform_id → platform(platform_id)

# 2. 資料表建立與基本操作

## Schema 建立與初始資料插入

系統主要資料表以 MariaDB 為例，包含：

- `product`（商品目錄）
- `product_category`（商品分類）
- `brand`（品牌廠商）
- `platform`（平台）
- `inventory_by_platform`（平台庫存）
- `purchase`（進貨）
- `sales`（出貨）

> Schema 建立

```sql
CREATE TABLE product_category (
  category_id INT PRIMARY KEY AUTO_INCREMENT,
  category_name VARCHAR(100) NOT NULL
);

CREATE TABLE brand (
  brand_id INT PRIMARY KEY AUTO_INCREMENT,
  brand_name VARCHAR(100) NOT NULL
);

CREATE TABLE platform (
  platform_id INT PRIMARY KEY AUTO_INCREMENT,
  platform_name VARCHAR(100) NOT NULL
);

CREATE TABLE product (
  product_id VARCHAR(50) PRIMARY KEY,
  category_id INT NOT NULL,
  brand_id INT NOT NULL,
  product_name VARCHAR(255) NOT NULL,
  product_spec VARCHAR(255),
  price_shopee DECIMAL(12,2),
  price_myship DECIMAL(12,2),
  average_cost DECIMAL(12,2),
  total_stock INT DEFAULT 0,
  remark TEXT,
  FOREIGN KEY (category_id) REFERENCES product_category(category_id),
  FOREIGN KEY (brand_id) REFERENCES brand(brand_id)
);

CREATE TABLE inventory_by_platform (
  product_id VARCHAR(50),
  platform_id INT,
  quantity INT DEFAULT 0,
  PRIMARY KEY (product_id, platform_id),
  FOREIGN KEY (product_id) REFERENCES product(product_id),
  FOREIGN KEY (platform_id) REFERENCES platform(platform_id)
);

CREATE TABLE purchase (
  purchase_id INT PRIMARY KEY AUTO_INCREMENT,
  purchase_date DATE NOT NULL,
  product_id VARCHAR(50) NOT NULL,
  quantity INT NOT NULL,
  cost DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  exchange_rate DECIMAL(6,4) NOT NULL,
  note TEXT,
  FOREIGN KEY (product_id) REFERENCES product(product_id)
);

CREATE TABLE sales (
  sale_id INT PRIMARY KEY AUTO_INCREMENT,
  sale_date DATE NOT NULL,
  platform_id INT NOT NULL,
  order_number VARCHAR(100),
  product_id VARCHAR(50) NOT NULL,
  quantity INT NOT NULL,
  amount DECIMAL(12,2) NOT NULL,
  note TEXT,
  FOREIGN KEY (platform_id) REFERENCES platform(platform_id),
  FOREIGN KEY (product_id) REFERENCES product(product_id)
);
```
> 初始資料插入
```sql
INSERT INTO product_category (category_name) VALUES ('鍵帽'), ('軸體'), ('工具');

INSERT INTO brand (brand_name) VALUES ('Aihey'), ('豆小牛工作室'), ('QvQ Studio');

INSERT INTO platform (platform_name) VALUES ('Shopee'), ('Myship'), ('Face-to-Face');

INSERT INTO product (product_id, category_id, brand_id, product_name, price_shopee, price_myship)
VALUES ('AUTO_GEN_001', 1, 2, '草莓餅乾個性鍵帽', 250, 220);

```

> 索引與效能考量

針對常用查詢欄位建立索引：
* product 表的 category_id、brand_id 已設索引
* inventory_by_platform 表的複合主鍵 (product_id, platform_id)
* purchase 表的 product_id 及 purchase_date 建立索引
* sales 表的 product_id、platform_id 及 sale_date 建立索引

> 交易 (Transaction) 機制

本系統在資料異動過程中（如出貨同時扣除庫存並新增訂單記錄）會使用交易機制（Transaction）來確保資料一致性。若任一步驟失敗，則所有變更會回復，避免資料不一致或庫存錯誤。

以下為一個實際範例：在出貨時同時更新平台庫存與新增出貨紀錄。

```sql
START TRANSACTION;

-- 步驟1：扣除平台庫存
UPDATE inventory_by_platform
SET quantity = quantity - 1
WHERE product_id = 'AUTO_GEN_001' AND platform_id = 1;

-- 步驟2：新增一筆出貨紀錄
INSERT INTO sales (
  sale_date, platform_id, order_number, product_id, quantity, amount, note
) VALUES (
  '2025-06-21', 1, 'SP20250621-008', 'AUTO_GEN_001', 1, 250.00, '模擬銷售'
);

-- 若以上兩步驟皆執行成功，則提交交易
COMMIT;
```

若在過程中發生錯誤（例如商品庫存不足或平台ID錯誤），則可使用下列語法回復整筆交易：
```sql
ROLLBACK;
```
此機制確保多個資料表更新操作能夠「全部成功或全部失敗」，大幅降低資料不一致的風險。

# 進階 SQL 功能應用
> 複雜查詢與子查詢

為了實作進銷存統計、平台銷售分析與商品狀況查詢，本系統設計以下查詢範例：

### 1. 每個平台的銷售總額與總出貨數量

```sql
SELECT 
  p.platform_name,
  COUNT(s.sale_id) AS total_orders,
  SUM(s.quantity) AS total_quantity,
  SUM(s.amount) AS total_revenue
FROM sales s
JOIN platform p ON s.platform_id = p.platform_id
GROUP BY s.platform_id
HAVING total_quantity > 0;
```

### 2. 查詢平均單價超過 200 元的商品
```sql
SELECT 
  product_name, 
  price_shopee, 
  price_myship, 
  (price_shopee + price_myship) / 2 AS avg_price
FROM product
WHERE (price_shopee + price_myship) / 2 > 200;
```

### 3. 建立視圖 View：各商品總銷售量
```sql
CREATE VIEW view_product_sales_summary AS
SELECT 
  p.product_id,
  p.product_name,
  SUM(s.quantity) AS total_sold,
  SUM(s.amount) AS total_sales_amount
FROM sales s
JOIN product p ON s.product_id = p.product_id
GROUP BY s.product_id;
```

### Stored Procedure / Function
本系統透過 Stored Procedure 處理複雜邏輯，例如：計算商品平均成本並更新至商品目錄。

```sql
DELIMITER $$

CREATE PROCEDURE CalculateAverageCost(
  IN p_product_id VARCHAR(50), 
  OUT avg_cost DECIMAL(12,2)
)
BEGIN
  SELECT 
    SUM(quantity * cost * exchange_rate) / SUM(quantity)
  INTO avg_cost
  FROM purchase
  WHERE product_id = p_product_id;
END $$

DELIMITER ;
```

### Trigger（觸發器）
本系統使用觸發器自動維護資料一致性，例如自動計算商品總庫存。
> AFTER INSERT Trigger：新增平台庫存時更新商品總庫存

```sql    
CREATE TRIGGER trg_update_total_stock_after_insert
AFTER INSERT ON inventory_by_platform
FOR EACH ROW
UPDATE product
SET total_stock = (
    SELECT SUM(quantity)
    FROM inventory_by_platform
    WHERE product_id = NEW.product_id
)
WHERE product_id = NEW.product_id;
```

> AFTER UPDATE Trigger：修改平台庫存時更新商品總庫存

```sql 
CREATE TRIGGER trg_update_total_stock_after_update
AFTER UPDATE ON inventory_by_platform
FOR EACH ROW
UPDATE product
SET total_stock = (
    SELECT SUM(quantity)
    FROM inventory_by_platform
    WHERE product_id = NEW.product_id
)
WHERE product_id = NEW.product_id;
```

>AFTER DELETE Trigger：刪除平台庫存時更新商品總庫存
```sql 
CREATE TRIGGER trg_update_total_stock_after_delete
AFTER DELETE ON inventory_by_platform
FOR EACH ROW
UPDATE product
SET total_stock = (
    SELECT IFNULL(SUM(quantity), 0)
    FROM inventory_by_platform
    WHERE product_id = OLD.product_id
)
WHERE product_id = OLD.product_id;
```

# 功能測試



### 1. 查詢目前所有商品總庫存

```sql
SELECT 
  product_id,
  product_name,
  total_stock
FROM product
ORDER BY total_stock DESC;
```
### 2. 查詢各平台目前的商品庫存狀況
```sql
SELECT 
  p.product_id,
  pr.product_name,
  pf.platform_name,
  p.quantity
FROM inventory_by_platform p
JOIN product pr ON p.product_id = pr.product_id
JOIN platform pf ON p.platform_id = pf.platform_id
ORDER BY pr.product_name, pf.platform_name;
```
### 3. 查詢每日銷售總額（近一週）
```sql
SELECT 
  sale_date,
  SUM(amount) AS daily_sales
FROM sales
WHERE sale_date >= CURDATE() - INTERVAL 7 DAY
GROUP BY sale_date
ORDER BY sale_date DESC;
```
### 4. 查詢某商品的進貨歷史紀錄與平均成本
```sql
SELECT 
  product_id,
  SUM(quantity) AS total_quantity,
  ROUND(SUM(quantity * cost * exchange_rate) / SUM(quantity), 2) AS avg_cost
FROM purchase
WHERE product_id = 'AUTO_GEN_001'
GROUP BY product_id;
```

# 資料一致性與異常情況測試

### 1. 外鍵約束測試
測試若插入不存在的 product_id 到 sales 表：

-- 預期會失敗（外鍵違反）
```sql
INSERT INTO sales (sale_date, platform_id, product_id, quantity, amount)
VALUES ('2025-06-22', 1, 'NOT_EXISTING_ID', 1, 100);
```
結果：觸發外鍵錯誤，驗證成功。
![image](https://hackmd.io/_uploads/r1hbAL84xg.png)


### 2. Transaction Rollback 測試
模擬出貨失敗情境（扣庫存後插入出貨資料錯誤）：

```sql
START TRANSACTION;

UPDATE inventory_by_platform
SET quantity = quantity - 1
WHERE product_id = 'AUTO_GEN_001' AND platform_id = 1;

-- 模擬錯誤（平台 ID 為不存在）
INSERT INTO sales (sale_date, platform_id, product_id, quantity, amount)
VALUES ('2025-06-22', 999, 'AUTO_GEN_001', 1, 250);

ROLLBACK;
```
✅ 結果：ROLLBACK 成功，庫存未異動，資料一致性維持。

### 效能測試（簡化）

#### 1. 未建立索引查詢時間（模擬）
查詢 sales 表中所有出貨金額合計，若資料量大、無索引：

```sql
SELECT SUM(amount) FROM sales;
```
📌 預期：查詢時間會較長，特別是資料筆數達數萬筆時。

#### 2. 已建立索引查詢時間
查詢有索引的 sale_date 欄位加速區間查詢：
```sql
SELECT COUNT(*) 
FROM sales 
WHERE sale_date BETWEEN '2025-06-01' AND '2025-06-15';
```
✅ 成效：查詢時間明顯下降，證實索引有效提升搜尋效能。
