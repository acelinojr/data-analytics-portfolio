populatedimtimelyhour: padronização do tempo (em até 6 meses no futuro) para garantir consistência temporal e confiabilidade dos resultados.

<details>

  ```
CREATE DEFINER=`Acelino`@`%` PROCEDURE `PopulateDimTimeHourly`()
BEGIN
    DECLARE start_dt DATETIME DEFAULT '2023-01-01 00:00:00';
    DECLARE end_dt DATETIME DEFAULT DATE_ADD(NOW(), INTERVAL 6 MONTH);
    DECLARE total_hours INT DEFAULT 0;

    SET total_hours = TIMESTAMPDIFF(HOUR, start_dt, end_dt);

    /* 
      Gera 0..total_hours usando cross-join de 5 dígitos (10^5 = 100000 valores).
      Ajuste número de níveis se precisar de >100k horas.
    */
    INSERT IGNORE INTO dim_time_hourly (
        hour_id, full_date, year, quarter, month, day, day_of_week, hour,
        is_weekend, is_month_end, is_quarter_end
    )
    SELECT
        CAST(DATE_FORMAT(DATE_ADD(start_dt, INTERVAL n HOUR), '%Y%m%d%H') AS UNSIGNED) AS hour_id,
        DATE(DATE_ADD(start_dt, INTERVAL n HOUR)) AS full_date,
        YEAR(DATE_ADD(start_dt, INTERVAL n HOUR)) AS year,
        QUARTER(DATE_ADD(start_dt, INTERVAL n HOUR)) AS quarter,
        MONTH(DATE_ADD(start_dt, INTERVAL n HOUR)) AS month,
        DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) AS day,
        (WEEKDAY(DATE_ADD(start_dt, INTERVAL n HOUR)) + 1) AS day_of_week,  -- 1=Mon .. 7=Sun
        HOUR(DATE_ADD(start_dt, INTERVAL n HOUR)) AS hour,
        ((WEEKDAY(DATE_ADD(start_dt, INTERVAL n HOUR)) + 1) IN (6,7)) AS is_weekend, -- 6=Sat,7=Sun
        (DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) = DAY(LAST_DAY(DATE_ADD(start_dt, INTERVAL n HOUR)))) AS is_month_end,
        (MONTH(DATE_ADD(start_dt, INTERVAL n HOUR)) IN (3,6,9,12) AND DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) = DAY(LAST_DAY(DATE_ADD(start_dt, INTERVAL n HOUR)))) AS is_quarter_end
    FROM (
        SELECT (a.n + b.n*10 + c.n*100 + d.n*1000 + e.n*10000) AS n
        FROM (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) a,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) b,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) c,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) d,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) e
    ) nums
    WHERE n BETWEEN 0 AND total_hours
    ORDER BY n;
END
```
</details>


Criação da tabela de staging: fact_market_hourly_stg

<details>

  ```
CREATE TABLE IF NOT EXISTS fact_market_hourly_stg (
  hour_ts DATETIME NOT NULL,
  hour_id BIGINT DEFAULT NULL,
  asset_symbol VARCHAR(64) NOT NULL,
  cnt_obs INT,
  avg_price DECIMAL(30,10),
  avg_change_24h_percent DOUBLE,
  total_volume DOUBLE,
  price_stddev DOUBLE,
  pct_valid DECIMAL(6,4),
  ingest_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (hour_ts, asset_symbol)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
</details>
