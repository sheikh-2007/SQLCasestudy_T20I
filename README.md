# 🏏 T20I Cricket Match & Venue Performance Analytics (SQL Case Study)

## 📌 Executive Summary
This project performs an in-depth exploratory data analysis (EDA) of International Twenty20 (T20I) cricket match records using **Microsoft SQL Server (T-SQL)**. Beyond sports analytics, this case study demonstrates how to clean and extract numerical metrics from unstructured text columns (e.g., `'45 runs'`, `'7 wickets'`), normalize multi-column entity records (`Team1`, `Team2`) using `UNION ALL`, and calculate win conversion percentages.

---

## 📁 Repository Files
* 📄 **`T20I_InsertData (1).sql`** — Database table creation and dataset insertion script.
* 📄 **`T20I_DataExploration.sql`** — Complete analytical SQL script answering the 10 business/performance questions below.

---

## 🛠️ Key SQL Techniques Demonstrated
* **String Parsing & Type Casting:** Extracting numerical margins from text strings using `SUBSTRING`, `CHARINDEX`, and `CAST(... AS INT)` to compute averages.
* **Data Normalization with `UNION ALL` & CTEs:** Combining `Team1` and `Team2` columns into a unified structure to accurately calculate total appearances and win percentages (`DECIMAL(5,2)`).
* **Window Functions:** Applying `DENSE_RANK()` and `RANK() OVER (PARTITION BY Ground ...)` to rank team dominance globally and by specific stadium.
* **Parameterized Queries:** Using T-SQL variables (`DECLARE @TeamA`, `@TeamB`) for dynamic head-to-head comparisons.

---

## 📊 Analytical Questions & SQL Solutions

### Q1. Head-to-head match log between two specific teams in 2024 (India vs. South Africa)
```sql
SELECT * 
FROM T20I
WHERE ((Team1 = 'South Africa' AND Team2 = 'India') 
    OR (Team2 = 'South Africa' AND Team1 = 'India'))
  AND YEAR(MatchDate) = 2024;
```

### Q2. Identify the team with the highest number of wins in 2024
```sql
SELECT TOP 1 
    Winner, 
    COUNT(*) AS [Number of Wins]
FROM T20I
WHERE YEAR(MatchDate) = 2024 
  AND Winner NOT IN ('tied', 'no result')
GROUP BY Winner
ORDER BY [Number of Wins] DESC;
```

### Q3. Rank all teams based on total wins in 2024 (handling ties with `DENSE_RANK`)
```sql
SELECT 
    Winner, 
    COUNT(*) AS [Number of Wins],
    DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) AS Rank_Assigned
FROM T20I
WHERE YEAR(MatchDate) = 2024 
  AND Winner NOT IN ('tied', 'no result')
GROUP BY Winner;
```

### Q4. Data Cleaning & Aggregation: Highest average winning margin in Runs & Wickets
```sql
-- 4A: Highest average winning margin (in runs)
SELECT TOP 1 
    Winner, 
    AVG(CAST(SUBSTRING(Margin, 1, CHARINDEX(' ', Margin) - 1) AS INT)) AS Avg_Run_Margin
FROM T20I
WHERE Margin LIKE '%runs'
GROUP BY Winner
ORDER BY Avg_Run_Margin DESC;

-- 4B: Highest average winning margin (in wickets)
SELECT TOP 1 
    Winner, 
    AVG(CAST(SUBSTRING(Margin, 1, CHARINDEX(' ', Margin) - 1) AS INT)) AS Avg_Wicket_Margin
FROM T20I
WHERE Margin LIKE '%wickets'
GROUP BY Winner
ORDER BY Avg_Wicket_Margin DESC;
```

### Q5. Benchmark Analysis: Matches where the winning run margin exceeded the global average
```sql
WITH CTE_AvgMargin AS (
    SELECT AVG(CAST(SUBSTRING(Margin, 1, CHARINDEX(' ', Margin) - 1) AS INT)) AS Avg_OverAllMargin
    FROM T20I
    WHERE Margin LIKE '%runs'
)
SELECT 
    T.Team1, 
    T.Team2, 
    T.Winner, 
    T.Margin
FROM T20I T
CROSS JOIN CTE_AvgMargin A
WHERE T.Margin LIKE '%runs'
  AND CAST(SUBSTRING(T.Margin, 1, CHARINDEX(' ', T.Margin) - 1) AS INT) > A.Avg_OverAllMargin;
```

### Q6. Identify the most successful team when chasing a target (wins by wickets)
```sql
SELECT Winner, WinWhileChasing
FROM (
    SELECT 
        Winner, 
        COUNT(*) AS WinWhileChasing,
        RANK() OVER (ORDER BY COUNT(*) DESC) AS rk
    FROM T20I
    WHERE Margin LIKE '%wickets'
      AND Winner NOT IN ('tied', 'no result')
    GROUP BY Winner
) t
WHERE rk = 1;
```

### Q7. Dynamic Head-to-Head Summary using T-SQL Variables
```sql
DECLARE @TeamA VARCHAR(25) = 'India';
DECLARE @TeamB VARCHAR(25) = 'South Africa';

SELECT 
    Winner, 
    COUNT(*) AS MatchesWon
FROM T20I
WHERE (Team1 = @TeamA AND Team2 = @TeamB) 
   OR (Team1 = @TeamB AND Team2 = @TeamA)
GROUP BY Winner;
```

### Q8. Seasonality Analysis: Identify peak months for T20I cricket in 2024
```sql
SELECT 
    YEAR(MatchDate) AS YearPlayed,
    DATENAME(MONTH, MatchDate) AS MonthName,
    COUNT(*) AS MatchesPlayed
FROM T20I
WHERE YEAR(MatchDate) = 2024
GROUP BY YEAR(MatchDate), MONTH(MatchDate), DATENAME(MONTH, MatchDate)
ORDER BY MatchesPlayed DESC;
```

### Q9. Win Conversion Rate: Total matches played vs. Win Percentage per team in 2024
```sql
WITH CTE_MatchesPlayed AS (
    SELECT Team, COUNT(*) AS MatchesPlayed
    FROM (
        SELECT Team1 AS Team FROM T20I WHERE YEAR(MatchDate) = 2024
        UNION ALL
        SELECT Team2 AS Team FROM T20I WHERE YEAR(MatchDate) = 2024
    ) t
    GROUP BY Team
),
CTE_Wins AS (
    SELECT Winner AS Team, COUNT(*) AS Wins
    FROM T20I
    WHERE YEAR(MatchDate) = 2024 
      AND Winner NOT IN ('tied', 'no result')
    GROUP BY Winner
)
SELECT 
    m.Team, 
    m.MatchesPlayed, 
    ISNULL(w.Wins, 0) AS Wins,
    CAST(ISNULL(w.Wins, 0) * 100.0 / m.MatchesPlayed AS DECIMAL(5,2)) AS WinPercentage
FROM CTE_MatchesPlayed m
LEFT JOIN CTE_Wins w ON m.Team = w.Team
ORDER BY WinPercentage DESC;
```

### Q10. Venue Dominance: Identify the most successful team at each cricket ground
```sql
WITH CTE_WinsPerGround AS (
    SELECT 
        Ground, 
        Winner, 
        Wins, 
        RANK() OVER (PARTITION BY Ground ORDER BY Wins DESC) AS rn
    FROM (
        SELECT Ground, Winner, COUNT(*) AS Wins
        FROM T20I 
        WHERE Winner NOT IN ('tied', 'no result')
        GROUP BY Ground, Winner
    ) t
)
SELECT 
    Ground, 
    Winner AS MostSuccessfulTeam, 
    Wins
FROM CTE_WinsPerGround
WHERE rn = 1
ORDER BY Ground;
```
