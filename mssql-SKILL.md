---
name: mssql
description: Use this skill whenever the user wants to write, optimize, debug, or work with Microsoft SQL Server databases. Triggers include any mention of 'SQL Server', 'MSSQL', 'T-SQL', 'Transact-SQL', 'SSMS', 'stored procedures', 'SQL Agent', or requests involving relational database design, query performance tuning, indexing strategies, database migrations, or data warehousing on SQL Server. Also use when creating tables, views, functions, triggers, CTEs, window functions, or when working with execution plans, deadlocks, tempdb, or SQL Server-specific features like MERGE, CROSS APPLY, STRING_AGG, or JSON support. Do NOT use for PostgreSQL, MySQL, SQLite, Oracle, or other non-SQL-Server databases unless the user is migrating to/from SQL Server.
---

# Microsoft SQL Server Skill

This skill guides the creation of professional, performant, and maintainable T-SQL code and SQL Server database designs following Microsoft's recommended practices and real-world production patterns.

The user provides database requirements: schema design, queries, stored procedures, performance tuning, migrations, reporting queries, or ETL logic. They may include context about their SQL Server version, data volumes, or performance constraints.

---

## Request Tracking — Two IDs Across Every Layer

Every request flowing through the system carries **two tracking IDs**:

| ID | Purpose | Lifecycle | Format | Example |
|----|---------|-----------|--------|---------|
| **CorrelationId** | Tracks an entire user flow / session across multiple requests | Generated once per user session or flow; passed across all related API calls | `UNIQUEIDENTIFIER` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| **RequestId** | Tracks a single individual request end-to-end | Generated fresh per API call; unique for every request | `UNIQUEIDENTIFIER` | `f9e8d7c6-b5a4-3210-fedc-ba0987654321` |

**Why two IDs?**
- **CorrelationId** lets support trace a user's entire journey (e.g., "create order → add 3 items → confirm" = 5 calls, 1 CorrelationId)
- **RequestId** pinpoints the exact call that failed (e.g., "the 2nd item insert" = 1 specific RequestId)

**Flow**: UI generates CorrelationId → HTTP headers → API generates RequestId → both passed to SP → SP logs to `log.RequestLog` → returns both in result set → API returns in response headers + body → UI shows RequestId on error screens.

**RULE: Every stored procedure MUST accept `@CorrelationId` and `@RequestId` as the first two parameters. No exceptions.**

---

## Environment Setup

When executing SQL locally, use the `sqlcmd` CLI or a Python/Node script with appropriate drivers. If SQL Server is not available locally, produce `.sql` script files the user can run in SSMS or Azure Data Studio.

```bash
# Check if sqlcmd is available
sqlcmd -? 2>/dev/null && echo "sqlcmd available" || echo "sqlcmd not found"

# Alternatively, use Python with pyodbc
pip install pyodbc --break-system-packages
```

Target **SQL Server 2019+** unless the user specifies an older version. Note features that require specific versions (e.g., `STRING_AGG` requires 2017+, `GENERATE_SERIES` requires 2022+).

---

## Request Log Table (MANDATORY)

**Every database MUST have this table.** Both tracking IDs are logged at START and updated at SUCCESS or ERROR.

```sql
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'log')
    EXEC('CREATE SCHEMA log');
GO

IF NOT EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'log' AND TABLE_NAME = 'RequestLog')
BEGIN
    CREATE TABLE log.RequestLog
    (
        Id              BIGINT              NOT NULL IDENTITY(1,1),
        CorrelationId   UNIQUEIDENTIFIER    NOT NULL,
        RequestId       UNIQUEIDENTIFIER    NOT NULL,
        SpName          NVARCHAR(200)       NOT NULL,
        Action          VARCHAR(20)         NULL,
        Parameters      NVARCHAR(MAX)       NULL,       -- optional JSON of input params for debugging
        UserId          INT                 NULL,
        Status          VARCHAR(20)         NOT NULL,    -- 'START' | 'SUCCESS' | 'ERROR'
        ErrorMessage    NVARCHAR(4000)      NULL,
        ErrorNumber     INT                 NULL,
        DurationMs      INT                 NULL,
        CreatedAt       DATETIME2(3)        NOT NULL CONSTRAINT DF_RequestLog_CreatedAt DEFAULT SYSUTCDATETIME(),

        CONSTRAINT PK_RequestLog PRIMARY KEY CLUSTERED (Id)
    );
END;
GO

CREATE NONCLUSTERED INDEX IX_RequestLog_CorrelationId ON log.RequestLog (CorrelationId) INCLUDE (RequestId, SpName, Status, CreatedAt);
CREATE NONCLUSTERED INDEX IX_RequestLog_RequestId     ON log.RequestLog (RequestId) INCLUDE (CorrelationId, SpName, Status);
CREATE NONCLUSTERED INDEX IX_RequestLog_CreatedAt     ON log.RequestLog (CreatedAt DESC) INCLUDE (CorrelationId, RequestId, Status);
GO
```

### Troubleshooting Queries

```sql
-- Customer gives you the RequestId from their error screen
SELECT * FROM log.RequestLog WHERE RequestId = 'f9e8d7c6-b5a4-3210-fedc-ba0987654321';

-- See the full flow (all requests in that session)
SELECT * FROM log.RequestLog WHERE CorrelationId = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890' ORDER BY CreatedAt;

-- All errors in last hour
SELECT * FROM log.RequestLog WHERE Status = 'ERROR' AND CreatedAt >= DATEADD(HOUR, -1, SYSUTCDATETIME()) ORDER BY CreatedAt DESC;

-- Slowest requests today
SELECT TOP 20 * FROM log.RequestLog WHERE CreatedAt >= CAST(SYSUTCDATETIME() AS DATE) ORDER BY DurationMs DESC;
```

---

## Database Design Principles

### Table Design — Standard Template

Every table MUST follow this structure with audit columns and soft-delete:

```sql
IF NOT EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'app' AND TABLE_NAME = 'Users')
BEGIN
    CREATE TABLE app.Users
    (
        Id          INT             NOT NULL IDENTITY(1,1),
        FirstName   NVARCHAR(100)   NOT NULL,
        LastName    NVARCHAR(100)   NOT NULL,
        Email       NVARCHAR(256)   NOT NULL,
        Phone       NVARCHAR(20)    NULL,
        IsActive    BIT             NOT NULL CONSTRAINT DF_Users_IsActive DEFAULT 1,
        CreatedBy   INT             NULL,
        CreatedAt   DATETIME2(3)    NOT NULL CONSTRAINT DF_Users_CreatedAt DEFAULT SYSUTCDATETIME(),
        ModifiedBy  INT             NULL,
        ModifiedAt  DATETIME2(3)    NULL,

        CONSTRAINT PK_Users PRIMARY KEY CLUSTERED (Id),
        CONSTRAINT UQ_Users_Email UNIQUE NONCLUSTERED (Email)
    );
END;
GO
```

### Design Rules

- **Always use schemas** — never put objects in `dbo` without intent; use schemas to organize by domain (e.g., `app.Orders`, `log.RequestLog`)
- **Always name constraints** explicitly (PK_, FK_, UQ_, DF_, CK_ prefixes)
- **Use DATETIME2** over DATETIME — better precision, range, and storage
- **Use NVARCHAR** for user-facing text, **VARCHAR** for system/ASCII-only data
- **Use DATETIME2(3)** for timestamps (millisecond precision, 7 bytes vs 8 for DATETIME)
- **Store dates in UTC** — use `SYSUTCDATETIME()`, not `GETDATE()`
- **Avoid NULLable columns** unless the business domain requires it
- **Audit columns on every table** — `CreatedBy`, `CreatedAt`, `ModifiedBy`, `ModifiedAt`
- **Soft-delete pattern** — `IsActive BIT DEFAULT 1`, never physically delete records
- **Use IDENTITY or SEQUENCE** for surrogate keys; consider GUIDs only when distributed generation is needed (use `NEWSEQUENTIALID()` to avoid page splits)
- **All DDL must be idempotent** — wrap in `IF NOT EXISTS` checks
- **Tracking IDs** — use `UNIQUEIDENTIFIER` type (not VARCHAR(36))

### Data Types — Choose Correctly

| Need                     | Use                  | Avoid                     |
|--------------------------|----------------------|---------------------------|
| Whole numbers            | INT / BIGINT         | FLOAT for IDs             |
| Money / currency         | DECIMAL(19,4)        | MONEY / FLOAT             |
| Short strings (<= 8000)  | VARCHAR(n) / NVARCHAR(n) | VARCHAR(MAX) unnecessarily |
| Large text               | NVARCHAR(MAX)        | TEXT (deprecated)         |
| Dates                    | DATE / DATETIME2     | DATETIME / SMALLDATETIME  |
| Booleans                 | BIT                  | INT / CHAR(1)             |
| Binary data              | VARBINARY(MAX)       | IMAGE (deprecated)        |
| Tracking IDs             | UNIQUEIDENTIFIER     | VARCHAR(36)               |

### Foreign Keys & Relationships

```sql
CREATE TABLE app.Orders
(
    Id          INT             NOT NULL IDENTITY(1,1),
    UserId      INT             NOT NULL,
    OrderDate   DATETIME2(3)    NOT NULL CONSTRAINT DF_Orders_OrderDate DEFAULT SYSUTCDATETIME(),
    TotalAmount DECIMAL(19,4)   NOT NULL CONSTRAINT DF_Orders_TotalAmt DEFAULT 0,
    -- ... audit columns ...

    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (Id),
    CONSTRAINT FK_Orders_Users FOREIGN KEY (UserId) REFERENCES app.Users(Id)
);

-- Always index foreign key columns
CREATE NONCLUSTERED INDEX IX_Orders_UserId ON app.Orders (UserId) INCLUDE (OrderDate, TotalAmount);
```

---

## Stored Procedure Naming Convention & Pattern

**Every table gets exactly 2 procedures:**

| Procedure | Purpose | @Action Values |
|-----------|---------|----------------|
| `{TableName}_Manage` | INSERT / UPDATE / DELETE | `'INSERT'`, `'UPDATE'`, `'DELETE'` |
| `{TableName}_Get` | Retrieve records | `'GETBYID'`, `'GETALL'`, `'GETFILTER'` |

Procedures are organized **table-wise** (one file per table) or **functionality-wise** (all _Manage together, all _Get together), depending on project size.

---

## _Manage Procedure — Full Pattern

The `_Manage` procedure handles all write operations via `@Action VARCHAR(10)`:

```sql
CREATE OR ALTER PROCEDURE app.Users_Manage
    -- ===== TRACKING (MANDATORY — always first two params) =====
    @CorrelationId  UNIQUEIDENTIFIER,
    @RequestId      UNIQUEIDENTIFIER,

    -- Control
    @Action         VARCHAR(10),              -- 'INSERT' | 'UPDATE' | 'DELETE'

    -- Primary key
    @Id             INT             = NULL,

    -- Table columns (all nullable with defaults for partial update)
    @FirstName      NVARCHAR(100)   = NULL,
    @LastName       NVARCHAR(100)   = NULL,
    @Email          NVARCHAR(256)   = NULL,
    @Phone          NVARCHAR(20)    = NULL,
    @IsActive       BIT             = NULL,

    -- Audit
    @UserId         INT             = NULL,   -- logged-in user performing the action
    @OutputId       INT             = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @StartTime DATETIME2(3) = SYSUTCDATETIME();
    DECLARE @SpName    NVARCHAR(200) = OBJECT_SCHEMA_NAME(@@PROCID) + '.' + OBJECT_NAME(@@PROCID);

    -- ===== LOG START =====
    INSERT INTO log.RequestLog (CorrelationId, RequestId, SpName, Action, UserId, Status)
    VALUES (@CorrelationId, @RequestId, @SpName, @Action, @UserId, 'START');

    BEGIN TRY

        --------------------------------------------------------------------------
        -- 1. VALIDATION (always first)
        --------------------------------------------------------------------------
        IF @Action IS NULL OR @Action NOT IN ('INSERT', 'UPDATE', 'DELETE')
        BEGIN
            THROW 50001, 'Invalid @Action. Allowed: INSERT, UPDATE, DELETE.', 1;
            RETURN;
        END;

        IF @Action IN ('UPDATE', 'DELETE') AND @Id IS NULL
        BEGIN
            THROW 50002, '@Id is required for UPDATE and DELETE.', 1;
            RETURN;
        END;

        -- INSERT-specific: required fields
        IF @Action = 'INSERT'
        BEGIN
            IF @FirstName IS NULL OR @LastName IS NULL OR @Email IS NULL
            BEGIN
                THROW 50003, '@FirstName, @LastName, and @Email are required for INSERT.', 1;
                RETURN;
            END;

            -- Duplicate check on business key
            IF EXISTS (SELECT 1 FROM app.Users WHERE Email = @Email AND IsActive = 1)
            BEGIN
                THROW 50004, 'A user with this email already exists.', 1;
                RETURN;
            END;
        END;

        -- UPDATE-specific: duplicate check if unique field is being changed
        IF @Action = 'UPDATE' AND @Email IS NOT NULL
        BEGIN
            IF EXISTS (SELECT 1 FROM app.Users WHERE Email = @Email AND Id <> @Id AND IsActive = 1)
            BEGIN
                THROW 50004, 'Another user with this email already exists.', 1;
                RETURN;
            END;
        END;

        --------------------------------------------------------------------------
        -- 2. INSERT
        --------------------------------------------------------------------------
        IF @Action = 'INSERT'
        BEGIN
            INSERT INTO app.Users (FirstName, LastName, Email, Phone, IsActive, CreatedBy, CreatedAt)
            VALUES (@FirstName, @LastName, @Email, @Phone, ISNULL(@IsActive, 1), @UserId, SYSUTCDATETIME());

            SET @OutputId = SCOPE_IDENTITY();
        END;

        --------------------------------------------------------------------------
        -- 3. UPDATE — ISNULL pattern: only overwrites columns that are passed
        --------------------------------------------------------------------------
        ELSE IF @Action = 'UPDATE'
        BEGIN
            UPDATE app.Users
            SET FirstName   = ISNULL(@FirstName, FirstName),
                LastName    = ISNULL(@LastName, LastName),
                Email       = ISNULL(@Email, Email),
                Phone       = ISNULL(@Phone, Phone),
                IsActive    = ISNULL(@IsActive, IsActive),
                ModifiedBy  = @UserId,
                ModifiedAt  = SYSUTCDATETIME()
            WHERE Id = @Id;

            SET @OutputId = @Id;
        END;

        --------------------------------------------------------------------------
        -- 4. DELETE — Soft Delete (set IsActive = 0)
        --------------------------------------------------------------------------
        ELSE IF @Action = 'DELETE'
        BEGIN
            UPDATE app.Users
            SET IsActive    = 0,
                ModifiedBy  = @UserId,
                ModifiedAt  = SYSUTCDATETIME()
            WHERE Id = @Id;

            SET @OutputId = @Id;
        END;

        -- ===== LOG SUCCESS =====
        UPDATE log.RequestLog
        SET Status = 'SUCCESS', DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

        --------------------------------------------------------------------------
        -- 5. RETURN the affected record (always) + tracking IDs
        --------------------------------------------------------------------------
        SELECT  u.Id, u.FirstName, u.LastName, u.Email, u.Phone,
                u.IsActive, u.CreatedBy, u.CreatedAt, u.ModifiedBy, u.ModifiedAt,
                @CorrelationId AS CorrelationId, @RequestId AS RequestId
        FROM    app.Users u
        WHERE   u.Id = @OutputId;

    END TRY
    BEGIN CATCH
        -- ===== LOG ERROR =====
        UPDATE log.RequestLog
        SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(), ErrorNumber = ERROR_NUMBER(),
            DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

        THROW;
    END CATCH;
END;
GO
```

### _Manage — Key Rules

- **@CorrelationId + @RequestId always first two params** — every SP, no exceptions
- **LOG START → TRY → DML → LOG SUCCESS → return record** — CATCH → LOG ERROR → THROW
- **Return tracking IDs in every result set**: `@CorrelationId AS CorrelationId, @RequestId AS RequestId`
- **@Action VARCHAR(10)** — always `'INSERT'`, `'UPDATE'`, or `'DELETE'`
- **Validate first** — check @Action, check @Id for UPDATE/DELETE, check required fields for INSERT
- **Duplicate check** — always check business key (Email, SKU, Code) before INSERT and UPDATE
- **ISNULL update pattern** — `SET Col = ISNULL(@Col, Col)` means only passed params overwrite
- **Soft delete** — never physically delete; set `IsActive = 0`
- **@UserId** — always accept the logged-in user for audit trail
- **@OutputId OUTPUT** — always return the affected Id
- **Always return the record** — SELECT the affected row at the end
- **Child table cascade** — when deleting a parent, soft-delete children too inside a transaction

### _Manage — Parent-Child Delete Pattern (e.g., Orders → OrderDetails)

```sql
-- Inside the DELETE block of a parent _Manage proc:
ELSE IF @Action = 'DELETE'
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;

        -- Soft-delete children first
        UPDATE app.OrderDetails
        SET    IsActive = 0, ModifiedBy = @UserId, ModifiedAt = SYSUTCDATETIME()
        WHERE  OrderId = @Id AND IsActive = 1;

        -- Soft-delete parent
        UPDATE app.Orders
        SET    IsActive = 0, ModifiedBy = @UserId, ModifiedAt = SYSUTCDATETIME()
        WHERE  Id = @Id;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        -- Log error with tracking IDs
        UPDATE log.RequestLog
        SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(), ErrorNumber = ERROR_NUMBER(),
            DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

        THROW;
    END CATCH;

    SET @OutputId = @Id;
END;
```

### _Manage — Child with Auto-Recalculate Pattern (e.g., OrderDetails recalcs Order.TotalAmount)

```sql
-- After INSERT/UPDATE/DELETE of a child row, recalculate the parent aggregate:
BEGIN TRY
    BEGIN TRANSACTION;

    -- ... INSERT/UPDATE/DELETE the child row ...

    -- Recalculate parent total
    UPDATE app.Orders
    SET TotalAmount = ISNULL((
            SELECT SUM(od.Quantity * od.UnitPrice)
            FROM   app.OrderDetails od
            WHERE  od.OrderId = @ResolvedOrderId AND od.IsActive = 1
        ), 0),
        ModifiedBy  = @UserId,
        ModifiedAt  = SYSUTCDATETIME()
    WHERE Id = @ResolvedOrderId;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

    UPDATE log.RequestLog
    SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(), ErrorNumber = ERROR_NUMBER(),
        DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
    WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

    THROW;
END CATCH;
```

---

## _Get Procedure — Full Pattern

The `_Get` procedure handles all read operations via `@Action VARCHAR(20)`:

```sql
CREATE OR ALTER PROCEDURE app.Users_Get
    -- ===== TRACKING (MANDATORY) =====
    @CorrelationId  UNIQUEIDENTIFIER,
    @RequestId      UNIQUEIDENTIFIER,

    -- Control
    @Action         VARCHAR(20)     = 'GETALL',   -- 'GETBYID' | 'GETALL' | 'GETFILTER'
    @Id             INT             = NULL,

    -- Filters
    @SearchText     NVARCHAR(200)   = NULL,        -- free-text search
    @IsActive       BIT             = 1,           -- default: active only

    -- Sorting
    @SortColumn     VARCHAR(50)     = 'Id',
    @SortDirection  VARCHAR(4)      = 'ASC',

    -- Pagination
    @PageNumber     INT             = 1,
    @PageSize       INT             = 20
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartTime DATETIME2(3) = SYSUTCDATETIME();
    DECLARE @SpName    NVARCHAR(200) = OBJECT_SCHEMA_NAME(@@PROCID) + '.' + OBJECT_NAME(@@PROCID);

    -- ===== LOG START =====
    INSERT INTO log.RequestLog (CorrelationId, RequestId, SpName, Action, Status)
    VALUES (@CorrelationId, @RequestId, @SpName, @Action, 'START');

    BEGIN TRY

        --------------------------------------------------------------------------
        -- GETBYID — single record
        --------------------------------------------------------------------------
        IF @Action = 'GETBYID'
        BEGIN
            IF @Id IS NULL
            BEGIN
                THROW 50001, '@Id is required for GETBYID.', 1;
                RETURN;
            END;

            SELECT  u.Id, u.FirstName, u.LastName, u.Email, u.Phone,
                    u.IsActive, u.CreatedBy, u.CreatedAt, u.ModifiedBy, u.ModifiedAt,
                    @CorrelationId AS CorrelationId, @RequestId AS RequestId
            FROM    app.Users u
            WHERE   u.Id = @Id;

            UPDATE log.RequestLog SET Status = 'SUCCESS',
                   DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
            WHERE  RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

            RETURN;
        END;

        --------------------------------------------------------------------------
        -- GETALL / GETFILTER — list with search + pagination
        --------------------------------------------------------------------------

        -- Sanitize pagination
        IF @PageNumber < 1 SET @PageNumber = 1;
        IF @PageSize   < 1 SET @PageSize = 20;
        IF @PageSize   > 200 SET @PageSize = 200;

        -- Sanitize sort (whitelist to prevent injection)
        IF @SortColumn NOT IN ('Id', 'FirstName', 'LastName', 'Email', 'CreatedAt')
            SET @SortColumn = 'Id';
        IF @SortDirection NOT IN ('ASC', 'DESC')
            SET @SortDirection = 'ASC';

        -- RESULT SET 1: Total count + tracking IDs
        SELECT  COUNT(*) AS TotalRecords,
                @CorrelationId AS CorrelationId, @RequestId AS RequestId
        FROM    app.Users u
        WHERE   (@IsActive IS NULL OR u.IsActive = @IsActive)
           AND  (@SearchText IS NULL
                OR u.FirstName   LIKE '%' + @SearchText + '%'
                OR u.LastName    LIKE '%' + @SearchText + '%'
                OR u.Email       LIKE '%' + @SearchText + '%');

        -- RESULT SET 2: Paged data with dynamic sort
        SELECT  u.Id, u.FirstName, u.LastName, u.Email, u.Phone,
                u.IsActive, u.CreatedBy, u.CreatedAt, u.ModifiedBy, u.ModifiedAt
        FROM    app.Users u
        WHERE   (@IsActive IS NULL OR u.IsActive = @IsActive)
           AND  (@SearchText IS NULL
                OR u.FirstName   LIKE '%' + @SearchText + '%'
                OR u.LastName    LIKE '%' + @SearchText + '%'
                OR u.Email       LIKE '%' + @SearchText + '%')
        ORDER BY
            CASE WHEN @SortColumn = 'Id'        AND @SortDirection = 'ASC'  THEN u.Id END ASC,
            CASE WHEN @SortColumn = 'Id'        AND @SortDirection = 'DESC' THEN u.Id END DESC,
            CASE WHEN @SortColumn = 'FirstName' AND @SortDirection = 'ASC'  THEN u.FirstName END ASC,
            CASE WHEN @SortColumn = 'FirstName' AND @SortDirection = 'DESC' THEN u.FirstName END DESC,
            CASE WHEN @SortColumn = 'LastName'  AND @SortDirection = 'ASC'  THEN u.LastName END ASC,
            CASE WHEN @SortColumn = 'LastName'  AND @SortDirection = 'DESC' THEN u.LastName END DESC,
            CASE WHEN @SortColumn = 'Email'     AND @SortDirection = 'ASC'  THEN u.Email END ASC,
            CASE WHEN @SortColumn = 'Email'     AND @SortDirection = 'DESC' THEN u.Email END DESC,
            CASE WHEN @SortColumn = 'CreatedAt' AND @SortDirection = 'ASC'  THEN u.CreatedAt END ASC,
            CASE WHEN @SortColumn = 'CreatedAt' AND @SortDirection = 'DESC' THEN u.CreatedAt END DESC
        OFFSET (@PageNumber - 1) * @PageSize ROWS
        FETCH NEXT @PageSize ROWS ONLY;

        UPDATE log.RequestLog SET Status = 'SUCCESS',
               DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE  RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

    END TRY
    BEGIN CATCH
        UPDATE log.RequestLog
        SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(), ErrorNumber = ERROR_NUMBER(),
            DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

        THROW;
    END CATCH;
END;
GO
```

### _Get — Key Rules

- **@CorrelationId + @RequestId always first two params** — identical to _Manage
- **@Action defaults to 'GETALL'** — calling with no params returns page 1 of active records
- **GETBYID returns early** — single record, no pagination, includes tracking IDs in columns
- **GETALL/GETFILTER return 2 result sets** — TotalRecords count (with tracking IDs) + paged data
- **@IsActive defaults to 1** — pass `NULL` to get all records including soft-deleted
- **@SearchText** — LIKE search across key text columns
- **Sort whitelist** — only allow known column names to prevent SQL injection
- **Page size cap** — enforce max 200 to prevent huge result sets
- **CASE-based ORDER BY** — for dynamic sorting without dynamic SQL
- **LOG START/SUCCESS/ERROR** — identical pattern to _Manage

### _Get — Parent with Child Result Sets (e.g., Orders GETBYID returns header + line items)

```sql
IF @Action = 'GETBYID'
BEGIN
    -- Result set 1: Order header with customer info + tracking IDs
    SELECT  o.Id, o.OrderNo, o.UserId,
            u.FirstName + ' ' + u.LastName AS CustomerName,
            u.Email AS CustomerEmail,
            o.OrderDate, o.TotalAmount, o.Status, o.Remarks,
            o.IsActive, o.CreatedBy, o.CreatedAt, o.ModifiedBy, o.ModifiedAt,
            @CorrelationId AS CorrelationId, @RequestId AS RequestId
    FROM    app.Orders o
    INNER JOIN app.Users u ON u.Id = o.UserId
    WHERE   o.Id = @Id;

    -- Result set 2: Line items
    SELECT  od.Id, od.OrderId, od.ProductId,
            p.ProductName, p.SKU,
            od.Quantity, od.UnitPrice, od.LineTotal, od.IsActive
    FROM    app.OrderDetails od
    INNER JOIN app.Products p ON p.Id = od.ProductId
    WHERE   od.OrderId = @Id AND od.IsActive = 1
    ORDER BY od.Id;

    RETURN;
END;
```

### _Get — Custom Filters (e.g., Orders with date range, status, FK filter)

```sql
CREATE OR ALTER PROCEDURE app.Orders_Get
    @CorrelationId  UNIQUEIDENTIFIER,
    @RequestId      UNIQUEIDENTIFIER,
    @Action         VARCHAR(20)     = 'GETALL',
    @Id             INT             = NULL,
    @CustUserId     INT             = NULL,       -- filter by customer FK
    @Status         VARCHAR(20)     = NULL,       -- filter by status
    @SearchText     NVARCHAR(200)   = NULL,
    @FromDate       DATETIME2(3)    = NULL,       -- date range start
    @ToDate         DATETIME2(3)    = NULL,       -- date range end
    @IsActive       BIT             = 1,
    @SortColumn     VARCHAR(50)     = 'Id',
    @SortDirection  VARCHAR(4)      = 'DESC',     -- orders default DESC
    @PageNumber     INT             = 1,
    @PageSize       INT             = 20
AS
-- ... WHERE clause includes:
--   AND (@CustUserId IS NULL OR o.UserId = @CustUserId)
--   AND (@Status IS NULL OR o.Status = @Status)
--   AND (@FromDate IS NULL OR o.OrderDate >= @FromDate)
--   AND (@ToDate IS NULL OR o.OrderDate <= @ToDate)
```

---

## Reusable Template — Copy-Paste for New Tables

Replace `{{placeholders}}` with your actual values:

```
{{SchemaName}}    → your schema       (e.g., app, sales, hr)
{{TableName}}     → your table        (e.g., Employees, Invoices)
{{ProcPrefix}}    → proc prefix       (e.g., Employees, Invoices)
{{UniqueColumn}}  → business key col  (e.g., Email, SKU, Code)
Column1..N        → your actual column names and data types
```

### Template: _Manage

```sql
CREATE OR ALTER PROCEDURE {{SchemaName}}.{{ProcPrefix}}_Manage
    -- ===== TRACKING (MANDATORY) =====
    @CorrelationId  UNIQUEIDENTIFIER,
    @RequestId      UNIQUEIDENTIFIER,

    @Action         VARCHAR(10),
    @Id             INT             = NULL,
    -- === YOUR COLUMNS (all nullable for partial update) ===
    @Column1        NVARCHAR(200)   = NULL,
    @Column2        VARCHAR(50)     = NULL,
    @Column3        NVARCHAR(1000)  = NULL,
    @Column4        DECIMAL(19,4)   = NULL,
    -- === END COLUMNS ===
    @IsActive       BIT             = NULL,
    @UserId         INT             = NULL,
    @OutputId       INT             = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @StartTime DATETIME2(3) = SYSUTCDATETIME();
    DECLARE @SpName NVARCHAR(200) = OBJECT_SCHEMA_NAME(@@PROCID) + '.' + OBJECT_NAME(@@PROCID);

    INSERT INTO log.RequestLog (CorrelationId, RequestId, SpName, Action, UserId, Status)
    VALUES (@CorrelationId, @RequestId, @SpName, @Action, @UserId, 'START');

    BEGIN TRY
        -- VALIDATION
        IF @Action IS NULL OR @Action NOT IN ('INSERT', 'UPDATE', 'DELETE')
        BEGIN THROW 50001, 'Invalid @Action. Allowed: INSERT, UPDATE, DELETE.', 1; RETURN; END;

        IF @Action IN ('UPDATE', 'DELETE') AND @Id IS NULL
        BEGIN THROW 50002, '@Id is required for UPDATE and DELETE.', 1; RETURN; END;

        IF @Action = 'INSERT'
        BEGIN
            IF @Column1 IS NULL OR @Column2 IS NULL
            BEGIN THROW 50003, '@Column1 and @Column2 are required for INSERT.', 1; RETURN; END;

            IF EXISTS (SELECT 1 FROM {{SchemaName}}.{{TableName}} WHERE {{UniqueColumn}} = @Column2 AND IsActive = 1)
            BEGIN THROW 50004, 'A record with this {{UniqueColumn}} already exists.', 1; RETURN; END;
        END;

        IF @Action = 'UPDATE' AND @Column2 IS NOT NULL
        BEGIN
            IF EXISTS (SELECT 1 FROM {{SchemaName}}.{{TableName}} WHERE {{UniqueColumn}} = @Column2 AND Id <> @Id AND IsActive = 1)
            BEGIN THROW 50004, 'Another record with this {{UniqueColumn}} already exists.', 1; RETURN; END;
        END;

        -- INSERT
        IF @Action = 'INSERT'
        BEGIN
            INSERT INTO {{SchemaName}}.{{TableName}} (Column1, Column2, Column3, Column4, IsActive, CreatedBy, CreatedAt)
            VALUES (@Column1, @Column2, @Column3, @Column4, ISNULL(@IsActive, 1), @UserId, SYSUTCDATETIME());
            SET @OutputId = SCOPE_IDENTITY();
        END
        -- UPDATE
        ELSE IF @Action = 'UPDATE'
        BEGIN
            UPDATE {{SchemaName}}.{{TableName}}
            SET Column1 = ISNULL(@Column1, Column1), Column2 = ISNULL(@Column2, Column2),
                Column3 = ISNULL(@Column3, Column3), Column4 = ISNULL(@Column4, Column4),
                IsActive = ISNULL(@IsActive, IsActive), ModifiedBy = @UserId, ModifiedAt = SYSUTCDATETIME()
            WHERE Id = @Id;
            SET @OutputId = @Id;
        END
        -- DELETE (Soft)
        ELSE IF @Action = 'DELETE'
        BEGIN
            UPDATE {{SchemaName}}.{{TableName}}
            SET IsActive = 0, ModifiedBy = @UserId, ModifiedAt = SYSUTCDATETIME()
            WHERE Id = @Id;
            SET @OutputId = @Id;
        END;

        UPDATE log.RequestLog SET Status = 'SUCCESS',
               DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';

        -- Return affected record + tracking IDs
        SELECT *, @CorrelationId AS CorrelationId, @RequestId AS RequestId
        FROM {{SchemaName}}.{{TableName}} WHERE Id = @OutputId;
    END TRY
    BEGIN CATCH
        UPDATE log.RequestLog SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(),
               ErrorNumber = ERROR_NUMBER(), DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';
        THROW;
    END CATCH;
END;
GO
```

### Template: _Get

```sql
CREATE OR ALTER PROCEDURE {{SchemaName}}.{{ProcPrefix}}_Get
    -- ===== TRACKING (MANDATORY) =====
    @CorrelationId  UNIQUEIDENTIFIER,
    @RequestId      UNIQUEIDENTIFIER,

    @Action         VARCHAR(20)     = 'GETALL',
    @Id             INT             = NULL,
    @SearchText     NVARCHAR(200)   = NULL,
    @IsActive       BIT             = 1,
    -- === YOUR CUSTOM FILTERS ===
    -- @Status      VARCHAR(20)     = NULL,
    -- @FromDate    DATETIME2(3)    = NULL,
    -- @ToDate      DATETIME2(3)    = NULL,
    -- @ForeignKeyId INT            = NULL,
    -- === END FILTERS ===
    @SortColumn     VARCHAR(50)     = 'Id',
    @SortDirection  VARCHAR(4)      = 'ASC',
    @PageNumber     INT             = 1,
    @PageSize       INT             = 20
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartTime DATETIME2(3) = SYSUTCDATETIME();
    DECLARE @SpName NVARCHAR(200) = OBJECT_SCHEMA_NAME(@@PROCID) + '.' + OBJECT_NAME(@@PROCID);

    INSERT INTO log.RequestLog (CorrelationId, RequestId, SpName, Action, Status)
    VALUES (@CorrelationId, @RequestId, @SpName, @Action, 'START');

    BEGIN TRY
        -- GETBYID
        IF @Action = 'GETBYID'
        BEGIN
            IF @Id IS NULL BEGIN THROW 50001, '@Id is required for GETBYID.', 1; RETURN; END;
            SELECT *, @CorrelationId AS CorrelationId, @RequestId AS RequestId
            FROM {{SchemaName}}.{{TableName}} WHERE Id = @Id;

            UPDATE log.RequestLog SET Status = 'SUCCESS',
                   DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
            WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';
            RETURN;
        END;

        -- GETALL / GETFILTER
        IF @PageNumber < 1 SET @PageNumber = 1;
        IF @PageSize < 1 SET @PageSize = 20;
        IF @PageSize > 200 SET @PageSize = 200;
        IF @SortColumn NOT IN ('Id', 'Column1', 'Column2', 'CreatedAt') SET @SortColumn = 'Id';
        IF @SortDirection NOT IN ('ASC', 'DESC') SET @SortDirection = 'ASC';

        -- Result set 1: Total count + tracking IDs
        SELECT COUNT(*) AS TotalRecords,
               @CorrelationId AS CorrelationId, @RequestId AS RequestId
        FROM {{SchemaName}}.{{TableName}} t
        WHERE (@IsActive IS NULL OR t.IsActive = @IsActive)
          AND (@SearchText IS NULL
               OR t.Column1 LIKE '%' + @SearchText + '%'
               OR t.Column2 LIKE '%' + @SearchText + '%');

        -- Result set 2: Paged data
        SELECT * FROM {{SchemaName}}.{{TableName}} t
        WHERE (@IsActive IS NULL OR t.IsActive = @IsActive)
          AND (@SearchText IS NULL
               OR t.Column1 LIKE '%' + @SearchText + '%'
               OR t.Column2 LIKE '%' + @SearchText + '%')
        ORDER BY
            CASE WHEN @SortColumn = 'Id'       AND @SortDirection = 'ASC'  THEN t.Id END ASC,
            CASE WHEN @SortColumn = 'Id'       AND @SortDirection = 'DESC' THEN t.Id END DESC,
            CASE WHEN @SortColumn = 'Column1'  AND @SortDirection = 'ASC'  THEN t.Column1 END ASC,
            CASE WHEN @SortColumn = 'Column1'  AND @SortDirection = 'DESC' THEN t.Column1 END DESC,
            CASE WHEN @SortColumn = 'Column2'  AND @SortDirection = 'ASC'  THEN t.Column2 END ASC,
            CASE WHEN @SortColumn = 'Column2'  AND @SortDirection = 'DESC' THEN t.Column2 END DESC,
            CASE WHEN @SortColumn = 'CreatedAt' AND @SortDirection = 'ASC'  THEN t.CreatedAt END ASC,
            CASE WHEN @SortColumn = 'CreatedAt' AND @SortDirection = 'DESC' THEN t.CreatedAt END DESC
        OFFSET (@PageNumber - 1) * @PageSize ROWS FETCH NEXT @PageSize ROWS ONLY;

        UPDATE log.RequestLog SET Status = 'SUCCESS',
               DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';
    END TRY
    BEGIN CATCH
        UPDATE log.RequestLog SET Status = 'ERROR', ErrorMessage = ERROR_MESSAGE(),
               ErrorNumber = ERROR_NUMBER(), DurationMs = DATEDIFF(MILLISECOND, @StartTime, SYSUTCDATETIME())
        WHERE RequestId = @RequestId AND SpName = @SpName AND Status = 'START';
        THROW;
    END CATCH;
END;
GO
```

---

## Usage Examples

```sql
-- INSERT a user
DECLARE @NewId INT;
EXEC app.Users_Manage
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = 'F9E8D7C6-B5A4-3210-FEDC-BA0987654321',
    @Action = 'INSERT', @FirstName = N'John', @LastName = N'Doe',
    @Email = N'john@example.com', @UserId = 1, @OutputId = @NewId OUTPUT;

-- UPDATE only the phone (other fields unchanged)
EXEC app.Users_Manage
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = '11111111-2222-3333-4444-555555555555',
    @Action = 'UPDATE', @Id = 1, @Phone = N'+91-9876543210', @UserId = 1;

-- SOFT DELETE
EXEC app.Users_Manage
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = '22222222-3333-4444-5555-666666666666',
    @Action = 'DELETE', @Id = 1, @UserId = 1;

-- GET by ID
EXEC app.Users_Get
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = '33333333-4444-5555-6666-777777777777',
    @Action = 'GETBYID', @Id = 1;

-- GET filtered with search and pagination
EXEC app.Users_Get
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = '44444444-5555-6666-7777-888888888888',
    @Action = 'GETFILTER', @SearchText = N'john',
    @SortColumn = 'FirstName', @SortDirection = 'ASC',
    @PageNumber = 1, @PageSize = 10;

-- GET including soft-deleted records
EXEC app.Users_Get
    @CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
    @RequestId     = '55555555-6666-7777-8888-999999999999',
    @IsActive = NULL;

-- Troubleshoot: customer gives you RequestId from the error screen
SELECT * FROM log.RequestLog WHERE RequestId = 'F9E8D7C6-B5A4-3210-FEDC-BA0987654321';
-- See the full session
SELECT * FROM log.RequestLog WHERE CorrelationId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890' ORDER BY CreatedAt;
```

---

## Query Writing Standards

### General Rules

- **Always qualify column names** with table aliases in multi-table queries
- **Use meaningful table aliases** — `u` for Users, `o` for Orders; not `a`, `b`, `c`
- **Terminate statements with semicolons**
- **Use SET NOCOUNT ON** at the top of every stored procedure
- **Avoid SELECT \*** in production code — always list explicit columns
- **Use TOP with ORDER BY** — `TOP` without `ORDER BY` gives non-deterministic results
- **Use `EXISTS` over `IN`** for correlated subqueries (better optimization)
- **Use `IS NULL` / `IS NOT NULL`** — never `= NULL`

### Common Table Expressions (CTEs)

Prefer CTEs over subqueries for readability:

```sql
WITH ActiveUsers AS
(
    SELECT u.Id, u.Email, u.FirstName, u.LastName
    FROM app.Users u
    WHERE u.IsActive = 1
),
UserOrderCounts AS
(
    SELECT o.UserId, COUNT(*) AS OrderCount, SUM(o.TotalAmount) AS TotalSpent
    FROM app.Orders o
    GROUP BY o.UserId
)
SELECT
    au.Email, au.FirstName, au.LastName,
    ISNULL(uoc.OrderCount, 0)  AS OrderCount,
    ISNULL(uoc.TotalSpent, 0)  AS TotalSpent
FROM ActiveUsers au
LEFT JOIN UserOrderCounts uoc ON uoc.UserId = au.Id
ORDER BY uoc.TotalSpent DESC;
```

### Window Functions

```sql
SELECT
    o.Id, o.UserId, o.OrderDate, o.TotalAmount,
    ROW_NUMBER() OVER (PARTITION BY o.UserId ORDER BY o.OrderDate DESC)  AS RowNum,
    SUM(o.TotalAmount) OVER (PARTITION BY o.UserId ORDER BY o.OrderDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)               AS RunningTotal,
    LAG(o.TotalAmount) OVER (PARTITION BY o.UserId ORDER BY o.OrderDate) AS PreviousOrderAmount
FROM app.Orders o;
```

### MERGE Statement

```sql
MERGE app.Products AS target
USING @StagingProducts AS source ON target.SKU = source.SKU
WHEN MATCHED THEN
    UPDATE SET target.ProductName = source.ProductName, target.UnitPrice = source.UnitPrice,
               target.ModifiedAt = SYSUTCDATETIME()
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ProductName, SKU, UnitPrice, CreatedAt)
    VALUES (source.ProductName, source.SKU, source.UnitPrice, SYSUTCDATETIME())
OUTPUT $action, inserted.Id, inserted.SKU;
```

---

## Indexing Strategy

### Index Types & When to Use

| Type                    | Use Case                                     |
|-------------------------|----------------------------------------------|
| Clustered               | Primary key (one per table, defines physical order) |
| Nonclustered            | WHERE, JOIN, ORDER BY columns                |
| Covering (INCLUDE)      | Avoid key lookups by including extra columns  |
| Filtered                | Partial index on a subset of rows             |
| Columnstore             | Analytics / aggregation on large tables       |

### Index Design Rules

- **Every foreign key column** should have a nonclustered index
- **Use INCLUDE columns** to create covering indexes
- **Filter indexes** for common WHERE conditions: `WHERE IsActive = 1`
- **Avoid over-indexing** — each index adds write overhead
- **Avoid leading key on low-cardinality columns** (e.g., BIT columns alone)

```sql
-- Covering index
CREATE NONCLUSTERED INDEX IX_Orders_UserId_OrderDate
    ON app.Orders (UserId, OrderDate DESC)
    INCLUDE (TotalAmount);

-- Filtered index
CREATE NONCLUSTERED INDEX IX_Users_Email_Active
    ON app.Users (Email)
    WHERE IsActive = 1;
```

---

## Performance Tuning

### Identifying Problems

```sql
-- Top 10 most expensive queries by CPU
SELECT TOP 10
    qs.total_worker_time / qs.execution_count AS AvgCPU,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset / 2) + 1,
        (CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
         ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2 + 1) AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;

-- Missing indexes
SELECT mid.statement, mid.equality_columns, mid.inequality_columns,
       mid.included_columns, migs.avg_user_impact
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;
```

### Common Performance Fixes

- **SARGability** — avoid functions on columns: `WHERE YEAR(OrderDate) = 2024` → `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`
- **Parameter sniffing** — use `OPTION (RECOMPILE)` or `OPTIMIZE FOR` when plans vary wildly
- **Implicit conversions** — match data types in comparisons (VARCHAR vs NVARCHAR causes scans)
- **Temp tables vs Table variables** — temp tables for large sets (get statistics); table variables for small sets (<100 rows)
- **NOLOCK** — use sparingly; prefer `READ COMMITTED SNAPSHOT ISOLATION`

---

## JSON Support (SQL Server 2016+)

```sql
-- Parse JSON
SELECT JSON_VALUE(j.doc, '$.name') AS Name, JSON_VALUE(j.doc, '$.email') AS Email
FROM app.JsonDocuments j WHERE ISJSON(j.doc) = 1;

-- Generate JSON
SELECT u.Id, u.Email, u.FirstName FROM app.Users u WHERE u.IsActive = 1
FOR JSON PATH, ROOT('users');

-- OPENJSON for arrays
SELECT item.*
FROM app.Orders o
CROSS APPLY OPENJSON(o.LineItemsJson) WITH (
    ProductId INT '$.productId', Quantity INT '$.quantity', UnitPrice DECIMAL(19,4) '$.unitPrice'
) AS item;
```

---

## Script Organization

```
scripts/
├── 00-schemas.sql          # CREATE SCHEMA (app, log)
├── 01-tables.sql           # Business tables + log.RequestLog
├── 02-indexes.sql          # Non-PK indexes
├── 03-SP_Users.sql         # Users_Manage + Users_Get
├── 04-SP_Products.sql      # Products_Manage + Products_Get
├── 05-SP_Orders.sql        # Orders_Manage + Orders_Get
├── 06-SP_OrderDetails.sql  # OrderDetails_Manage + OrderDetails_Get
├── 07-views.sql            # View definitions
├── 08-seed-data.sql        # Reference/lookup data
└── 09-permissions.sql      # GRANT/DENY statements
```

Each script must be **idempotent** — use `IF NOT EXISTS`, `CREATE OR ALTER`, and `GO` batch separators.

---

## Key Design Decisions Summary

| Feature | Implementation |
|---------|---------------|
| **Request Tracking** | `@CorrelationId` + `@RequestId` as first 2 params of every SP, logged to `log.RequestLog` |
| **Delete strategy** | Soft delete (`IsActive = 0`) — data is never physically removed |
| **Audit columns** | `CreatedBy`, `CreatedAt`, `ModifiedBy`, `ModifiedAt` on every table |
| **Update pattern** | `ISNULL(@Param, ExistingValue)` — only overwrites columns you pass |
| **Pagination** | `OFFSET/FETCH` with max page size cap (200) |
| **Sort injection** | Whitelisted column names + CASE-based ORDER BY |
| **Search** | `LIKE '%text%'` on key text columns |
| **Return pattern** | Every `_Manage` call returns the affected record + tracking IDs |
| **Total count** | `_Get` returns TotalRecords + tracking IDs as first result set |
| **Child tables** | `OrderDetails_Manage` auto-recalculates parent `TotalAmount` |
| **Order GETBYID** | Returns 2 result sets: header + line items |
| **Error logging** | TRY/CATCH logs ERROR with message + error number to `log.RequestLog` |

---

## Output Delivery

1. Write all `.sql` files under `/home/claude/`
2. **Always include `log` schema + `log.RequestLog` table** in schema scripts
3. **Every SP must accept `@CorrelationId` + `@RequestId`** as first two params
4. **Every SP must log START/SUCCESS/ERROR** to `log.RequestLog`
5. **Every result set must include `CorrelationId` and `RequestId` columns**
6. Validate SQL syntax where possible (basic static checks)
7. Copy final scripts to `/mnt/user-data/outputs/`
8. For multi-file deliverables, organize by execution order and include a `README.md`
9. Always include comments explaining complex logic and `GO` batch separators
