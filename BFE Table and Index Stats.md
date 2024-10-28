# SQL Query: Index Fragmentation and Table Statistics

**Purpose**: This query retrieves statistics on all indexes within the given database, including fragmentation percentage, space allocated, space used, and other helpful metrics.

---

## Explanation of Metrics

- **TableName**: The name of the table the index belongs to.
- **IndexName**: The name of the index.
- **IndexType**: Describes the index type (e.g., clustered, non-clustered).
- **AllocationType**: Shows whether the index is stored on `IN_ROW_DATA` or `LOB_DATA` (useful for large object data types).
- **SpaceUsedKB / SpaceUsedMB / SpaceUsedGB**: The space used by the index, provided in kilobytes, megabytes, and gigabytes.
- **RecordCount**: Total number of records in the index.
- **PageCount**: Number of pages used by the index.
- **FragmentationPercent**: Percentage of index fragmentation.
- **FragmentationStatus**: Classification based on fragmentation level to help prioritize maintenance (e.g., reorganizing or rebuilding indexes).

---

## Tips for Reducing Fragmentation

- **Low fragmentation**: No action necessary.
- **Moderate fragmentation** (10-30%): Use `ALTER INDEX … REORGANIZE`.
- **High fragmentation** (>30%): Use `ALTER INDEX … REBUILD`.

---

## Query

```sql
WITH IndexStats AS (
    SELECT
        OBJECT_NAME(ps.[object_id]) AS TableName,
        i.name AS IndexName,
        i.index_id AS IndexID,
        i.type_desc AS IndexType,
        ps.[alloc_unit_type_desc] AS AllocationType,
        ps.[page_count] * 8 AS SpaceUsedKB, -- Page count in KB
        (ps.[page_count] * 8) / 1024 AS SpaceUsedMB,
        ps.[page_count] * 8 / 1024 / 1024 AS SpaceUsedGB,
        ps.[record_count] AS RecordCount,
        ps.[page_count] AS PageCount,
        ps.[avg_fragmentation_in_percent] AS FragmentationPercent
    FROM 
        sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ps
    INNER JOIN 
        sys.indexes i ON ps.[object_id] = i.[object_id] AND ps.[index_id] = i.[index_id]
    WHERE 
        i.is_primary_key = 0 -- Exclude primary key indexes if not needed
        AND i.is_unique_constraint = 0 -- Exclude unique constraint indexes if not needed
)

SELECT 
    i.TableName,
    i.IndexName,
    i.IndexType,
    i.AllocationType,
    i.SpaceUsedKB,
    i.SpaceUsedMB,
    i.SpaceUsedGB,
    i.RecordCount,
    i.PageCount,
    i.FragmentationPercent,
    CASE 
        WHEN i.FragmentationPercent >= 30 THEN 'Highly Fragmented'
        WHEN i.FragmentationPercent BETWEEN 10 AND 30 THEN 'Moderately Fragmented'
        ELSE 'Low Fragmentation'
    END AS FragmentationStatus
FROM 
    IndexStats i
ORDER BY 
    i.FragmentationPercent DESC, i.SpaceUsedMB DESC;
```

---

## Example Results

| TableName         | IndexName                                      | IndexType      | AllocationType | SpaceUsedKB | SpaceUsedMB | SpaceUsedGB | RecordCount | PageCount | FragmentationPercent | FragmentationStatus    |
|-------------------|----------------------------------------------- |----------------|----------------|-------------|-------------|-------------|-------------|-----------|----------------------|------------------------|
| COMPUTER_SITES    | WEBUI_COMPSITES_SiteID                         | NONCLUSTERED   | IN_ROW_DATA    | 32          | 0           | 0           | 586         | 4         | 75                   | Highly Fragmented      |
| ACTIONS           | WEBUI_ACT_Retry                                | NONCLUSTERED   | IN_ROW_DATA    | 88          | 0           | 0           | 5833        | 11        | 27.27                | Moderately Fragmented  |
| ACTIONRESULTS     | IX_Sequence                                    | CLUSTERED      | IN_ROW_DATA    | 712         | 0           | 0           | 3781        | 89        | 24.72                | Moderately Fragmented  |
| QUESTIONRESULTS   | IX_CID                                         | NONCLUSTERED   | IN_ROW_DATA    | 168         | 0           | 0           | 4492        | 21        | 23.81                | Moderately Fragmented  |
| FIXLETRESULTS     | WEBUI_FR_IsRelevant_SiteID_Type                | NONCLUSTERED   | IN_ROW_DATA    | 608         | 0           | 0           | 24315       | 76        | 23.68                | Moderately Fragmented  |
| ACTION_DEFS       | INDEX_ACTION_DEFS_ROWVERSION                   | NONCLUSTERED   | IN_ROW_DATA    | 352         | 0           | 0           | 8955        | 44        | 22.73                | Moderately Fragmented  |
| ACTION_FLAGS      | IX_ISDELETED                                   | NONCLUSTERED   | IN_ROW_DATA    | 112         | 0           | 0           | 5833        | 14        | 21.43                | Moderately Fragmented  |
| QUESTIONRESULTS   | WEBUI_QR_AnalysisID_PropertyID_IsFailure_WebuiSiteID | NONCLUSTERED   | IN_ROW_DATA    | 168         | 0           | 0           | 4492        | 21        | 19.05                | Moderately Fragmented  |
| EXTERNAL_FIXLET_ACTIONS | ix_sequence                             | NONCLUSTERED   | IN_ROW_DATA    | 41120       | 40          | 0           | 755736      | 5140      | 3.39                 | Low Fragmentation      |
| EXTERNAL_FIXLET_FIELDS  | ix_sequence                             | NONCLUSTERED   | IN_ROW_DATA    | 149128      | 145         | 0           | 1726400     | 18641     | 4.35                 | Low Fragmentation      |
| FIXLETRESULTS     | IX_Sequence                                    | NONCLUSTERED   | IN_ROW_DATA    | 968         | 0           | 0           | 24315       | 121       | 4.96                 | Low Fragmentation      |
| ACTIONS           | WEBUI_ACT_IsOffer                              | NONCLUSTERED   | IN_ROW_DATA    | 72          | 0           | 0           | 5833        | 9         | 11.11                | Moderately Fragmented  |
| EXTERNAL_FIXLET_RELEVANCE | ix_sequence                           | NONCLUSTERED   | IN_ROW_DATA    | 59368       | 57          | 0           | 1434126     | 7421      | 2.13                 | Low Fragmentation      |
| ACTIONRESULTS     | IX_AID_CID                                     | NONCLUSTERED   | IN_ROW_DATA    | 136         | 0           | 0           | 3781        | 17        | 11.76                | Moderately Fragmented  |
| EXTERNAL_ANALYSES | ix_sequence                                    | NONCLUSTERED   | IN_ROW_DATA    | 816         | 0           | 0           | 22449       | 102       | 1.96                 | Low Fragmentation      |

> **Note**: The above table shows only a subset of results as an example.

