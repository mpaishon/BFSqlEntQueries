# SQL Query: Top 20 Largest Properties by Data Size

**Description**: This query retrieves details of the largest 20 properties in the system, ranked by data size, and indicates whether the data is sourced from the `QUESTIONRESULTS` (QR) or `LONGQUESTIONRESULTS` (LQR) table.

---

## Column Explanation

- **SiteID**: The unique identifier for the site associated with the property.
- **AnalysisID**: The unique identifier for the analysis to which the property belongs.
- **PropertyID**: The unique identifier for the property within the analysis.
- **DataSizeBytes**: The size of the property data in bytes.
- **DataSizeKB**: The size of the property data in kilobytes.
- **DataSizeMB**: The size of the property data in megabytes.
- **DataSizeGB**: The size of the property data in gigabytes.
- **DataSizePercentage**: The size of the property data as a percentage of the total size of the `QUESTIONRESULTS` and `LONGQUESTIONRESULTS` tables.
- **SourceTable**: Indicates whether the data is stored in the `QUESTIONRESULTS` table (QR) or the `LONGQUESTIONRESULTS` table (LQR).

---

## Query

```sql
-- Calculate total data size of QUESTIONRESULTS and LONGQUESTIONRESULTS tables
WITH TotalSize AS (
    SELECT 
        SUM(CASE 
                WHEN QR.ResultsText IS NOT NULL THEN CAST(DATALENGTH(QR.ResultsText) AS BIGINT) 
                ELSE CAST(DATALENGTH(LQR.ResultsText) AS BIGINT) 
            END
        ) AS TotalDataSize
    FROM 
        BFEnterprise.dbo.QUESTIONRESULTS QR WITH (NOLOCK)
    LEFT OUTER JOIN 
        BFEnterprise.dbo.LONGQUESTIONRESULTS LQR WITH (NOLOCK) 
        ON QR.AnalysisID = LQR.AnalysisID 
        AND QR.SiteID = LQR.SiteID 
        AND QR.PropertyID = LQR.PropertyID 
        AND QR.ComputerID = LQR.ComputerID
),
TopProperties AS (
    SELECT 
        QR.SiteID, 
        QR.AnalysisID, 
        QR.PropertyID, 
        SUM(
            CASE 
                WHEN QR.ResultsText IS NOT NULL THEN CAST(DATALENGTH(QR.ResultsText) AS BIGINT) 
                ELSE CAST(DATALENGTH(LQR.ResultsText) AS BIGINT) 
            END
        ) AS DataSizeBytes,
        CASE 
            WHEN QR.ResultsText IS NOT NULL THEN 'QR' 
            ELSE 'LQR' 
        END AS SourceTable
    FROM 
        BFEnterprise.dbo.QUESTIONRESULTS QR WITH (NOLOCK)
    LEFT OUTER JOIN 
        BFEnterprise.dbo.LONGQUESTIONRESULTS LQR WITH (NOLOCK) 
        ON QR.AnalysisID = LQR.AnalysisID 
        AND QR.SiteID = LQR.SiteID 
        AND QR.PropertyID = LQR.PropertyID 
        AND QR.ComputerID = LQR.ComputerID 
    GROUP BY 
        QR.SiteID, QR.AnalysisID, QR.PropertyID,
        CASE 
            WHEN QR.ResultsText IS NOT NULL THEN 'QR' 
            ELSE 'LQR' 
        END
)

-- Select top 20 properties with calculated size percentage and source table
SELECT TOP 20 
    TP.SiteID,
    TP.AnalysisID,
    TP.PropertyID,
    TP.DataSizeBytes,
    TP.DataSizeBytes / 1024 AS DataSizeKB,
    TP.DataSizeBytes / 1024 / 1024 AS DataSizeMB,
    TP.DataSizeBytes / 1024 / 1024 / 1024 AS DataSizeGB,
    (CAST(TP.DataSizeBytes AS DECIMAL(18, 2)) / TS.TotalDataSize) * 100 AS DataSizePercentage,
    TP.SourceTable
FROM 
    TopProperties TP
CROSS JOIN 
    TotalSize TS
ORDER BY 
    TP.DataSizeBytes DESC;
```

---

## Example Results

| SiteID       | AnalysisID | PropertyID | DataSizeBytes | DataSizeKB | DataSizeMB | DataSizeGB | DataSizePercentage | SourceTable |
|--------------|------------|------------|---------------|------------|------------|------------|---------------------|-------------|
| -1995309915  | 54484      | 1          | 2180470       | 2129       | 2          | 0          | 71.72              | LQR         |
| 3093         | 34         | 2          | 209774        | 204        | 0          | 0          | 6.90               | LQR         |
| -1995309915  | 8041       | 2          | 98364         | 96         | 0          | 0          | 3.24               | LQR         |
| -1995309915  | 8041       | 1          | 98364         | 96         | 0          | 0          | 3.24               | LQR         |
| -1995309915  | 21         | 1          | 61102         | 59         | 0          | 0          | 2.01               | QR          |
| -1995309915  | 18         | 1          | 45136         | 44         | 0          | 0          | 1.48               | QR          |
| 3093         | 35         | 16         | 25072         | 24         | 0          | 0          | 0.82               | QR          |
| 3093         | 34         | 1          | 18532         | 18         | 0          | 0          | 0.61               | QR          |
| 3093         | 35         | 17         | 17092         | 16         | 0          | 0          | 0.56               | QR          |
| 3093         | 34         | 1          | 16642         | 16         | 0          | 0          | 0.55               | LQR         |
| -1995309915  | 21         | 1          | 16214         | 15         | 0          | 0          | 0.53               | LQR         |
| -1995309915  | 54663      | 1          | 15916         | 15         | 0          | 0          | 0.52               | LQR         |
| -1995309915  | 8041       | 1          | 15816         | 15         | 0          | 0          | 0.52               | QR          |
| -1995309915  | 8041       | 2          | 15816         | 15         | 0          | 0          | 0.52               | QR          |
| -1995309915  | 18         | 1          | 11978         | 11         | 0          | 0          | 0.39               | LQR         |
| -1995309915  | 9798       | 2          | 11698         | 11         | 0          | 0          | 0.38               | LQR         |
| 8521         | 1300       | 2          | 11698         | 11         | 0          | 0          | 0.38               | LQR         |
| 8521         | 1006       | 1          | 9778          | 9          | 0          | 0          | 0.32               | QR          |
| 1            | 2814       | 1          | 5064          | 4          | 0          | 0          | 0.17               | QR          |
| -1995309915  | 4470       | 1          | 4434          | 4          | 0          | 0          | 0.15               | QR          |

---

