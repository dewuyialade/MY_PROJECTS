# OLTP to Staging: Store Dimension ETL Package

![SSIS store staging control flow](../Screenshot%202026-09-15%20173245.png)

## Overview

This documentation describes an SQL Server Integration Services (SSIS) package that extracts store data from an operational online transaction processing (OLTP) system and loads it into a staging layer.

The screenshots show the package `stgstore.dtsx` in the  project. The package is an example of a repeatable staging pattern: clear the previous staging data, count the source rows, extract and load the current OLTP data, count the destination rows, and record execution metrics.

## End-to-end process

```text
TESCA_OLTP
    |
    |  Extract store data with joins and a load timestamp
    v
Load Staging (Store)
    ^
    |  Full-refresh staging table is cleared first
    |
TESCA_STAGING

Source Count -> Load Staging (Store) -> Destination Count -> Metrics
```

The green precedence constraints in the control-flow screenshot indicate that each task runs after the preceding task succeeds.

## Control flow

![Store staging control flow](../Screenshot%202026-09-15%20173245.png)

The package contains five visible steps:

1. **Truncate Staging (Store)** clears the existing store staging data. This suggests a full-refresh load for the staging subject area.
2. **Source Count** captures the number of source rows expected from the OLTP query.
3. **Load Staging (Store)** runs the SSIS data flow that extracts and inserts the store records.
4. **Des Count** captures the number of rows loaded into the destination. The task name appears to mean destination count.
5. **Metrics** stores execution statistics and updates package-control information.

The package is one member of a broader staging project that also contains packages for customers, employees, products, promotions, vendors, sales facts, purchase facts, and other business subjects.

## Data flow

![Store data flow](../Screenshot%202026-09-15%20174328.png)

The data-flow view shows two components:

- **Extract**, which reads from the OLTP system.
- **Load Store**, which writes the extracted rows into the staging destination.

The simple design makes the movement of data easy to inspect and provides a clean location for adding transformations, data-quality checks, derived columns, or error outputs as the pipeline matures.

## OLTP source query

![OLTP source configuration](../Screenshot%202026-09-15%20174349.png)

The source uses the `TESCA_OLTP` OLE DB connection manager and the `SQL command` access mode:
```sql
SELECT
    s.StoreID,
    s.StoreName,
    s.StreetAddress,
    c.CityName AS City,
    st.State,
    GETDATE() AS LoadDate
FROM Store AS s
INNER JOIN City AS c
    ON s.CityID = c.CityID
INNER JOIN State AS st
    ON st.StateID = c.StateID;
```

This query demonstrates several useful ETL decisions:

- The source is the OLTP database rather than the warehouse.
- Store data is denormalized and enriched with city and state attributes during extraction.
- An ETL load timestamp is added to support traceability.
- Inner joins restrict the result to stores with matching city and state reference records.


## Metrics and audit SQL

![Metrics task SQL](../Screenshot%202026-09-15%20174424.png)

The final Execute SQL Task uses package parameters or variables for the package identifier, source count, and destination count as shown below:

```sql
DECLARE @PackageID INT = ?;
DECLARE @StgSourceCount INT = ?;
DECLARE @StgDestCount INT = ?;

BEGIN
    INSERT INTO control.metrics
        (PackageID, StgSourceCount, StgDestCount, LoadDate)
    SELECT
        @PackageID,
        @StgSourceCount,
        @StgDestCount,
        GETDATE();

    UPDATE control.package
    SET LastUpdate = GETDATE()
    WHERE PackageID = @PackageID;
END;
```

The question marks represent SSIS parameter placeholders. Their order must match the parameter mapping configured in the Execute SQL Task.

This step provides two forms of operational visibility:

- A historical metric record containing source count, destination count, package ID, and load date.
- A current `control.package` update showing when the package last ran.

## Connection architecture

The SSIS project displays four project-level connection managers:

- `TESCA_OLTP`: operational source system.
- `TESCA_STAGING`: staging destination.
- `TESCA_CONTROL`: package metadata, metrics, and operational control tables.
- `TESCA_EDW`: enterprise data warehouse connection for later pipeline stages.

This separation reflects a layered data architecture in which transactional data is extracted from OLTP, standardized in staging, audited through control tables, and subsequently made available for warehouse loading.

## What this design communicates 

The screenshots demonstrate the following engineering principles:

- A clear boundary between transactional source data and analytical processing.
- Subject-area-oriented SSIS package design.
- Task sequencing using precedence constraints.

- Source-to-destination reconciliation through row counts.
- Operational auditability through metrics and package-control records.
- Early enrichment of store data using reference-table joins.
- Addition of a load timestamp for data lineage and troubleshooting.

The design is intentionally understandable: a reviewer can trace the data from the OLTP query through the destination load and then into the metrics tables.
