WITH UpdatedRows AS (
    SELECT 
        CT.SYS_CHANGE_VERSION,
        CT.SYS_CHANGE_CREATION_VERSION,
        CT.SYS_CHANGE_COLUMNS, 
        CT.SYS_CHANGE_CONTEXT,
        T.*
    FROM 
        CHANGETABLE(CHANGES TableName, 100) AS CT
    JOIN 
        TableName AS T
    ON 
        CT.PrimaryKeyColumn = T.PrimaryKeyColumn
    WHERE
        CT.SYS_CHANGE_OPERATION = 'U' -- Filter for updates only
)
SELECT 
    UR.SYS_CHANGE_VERSION,
    UR.PrimaryKeyColumn,
    -- Add custom logic to interpret SYS_CHANGE_COLUMNS bitmap
    -- Example for common columns (adjust column names as needed):
    CASE WHEN CHANGE_TRACKING_IS_COLUMN_IN_MASK(COLUMNPROPERTY(OBJECT_ID('TableName'), 'Column1', 'ColumnId'), UR.SYS_CHANGE_COLUMNS) = 1 
         THEN 'Changed' ELSE 'Unchanged' END AS Column1_Status,
    CASE WHEN CHANGE_TRACKING_IS_COLUMN_IN_MASK(COLUMNPROPERTY(OBJECT_ID('TableName'), 'Column2', 'ColumnId'), UR.SYS_CHANGE_COLUMNS) = 1 
         THEN 'Changed' ELSE 'Unchanged' END AS Column2_Status,
    CASE WHEN CHANGE_TRACKING_IS_COLUMN_IN_MASK(COLUMNPROPERTY(OBJECT_ID('TableName'), 'Column3', 'ColumnId'), UR.SYS_CHANGE_COLUMNS) = 1 
         THEN 'Changed' ELSE 'Unchanged' END AS Column3_Status,
    -- Include the current values (after the update)
    UR.Column1,
    UR.Column2,
    UR.Column3
FROM 
    UpdatedRows AS UR
ORDER BY 
    UR.SYS_CHANGE_VERSION;
