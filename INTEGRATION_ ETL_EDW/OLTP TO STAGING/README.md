# OLTP to Staging: Store Dimension ETL Package

![SSIS store staging control flow](/INTEGRATION_%20ETL_EDW/lmages/Screenshot%202026-09-15%20173245.png)

## Overview

This report documents the design and execution of a SQL Server Integration Services (SSIS) package used to extract store-dimension records from the operational OLTP source system and load them into the staging layer. The package follows a structured extract-load-audit pattern that enforces data freshness, supports row-count reconciliation, and provides operational traceability for staging processes.

The implementation demonstrates a disciplined ETL approach in which the staging table is cleared before each full refresh, source and destination row totals are measured, and execution results are recorded in control tables. This design ensures that the staging environment reflects the current state of the source data while maintaining a clear and auditable trail for operational review.

## Business and Technical Context

The package is designed to support the movement of store data from the transactional source environment into a staging repository that serves downstream analytical processing. In the broader data architecture, staging acts as the intermediary layer between operational systems and the enterprise warehouse, enabling transformations, validation, and controlled loading without directly modifying the OLTP source.

The process is structured around a repeatable pattern that includes:

- Truncating prior staged data to maintain a clean state.
- Capturing the expected source volume before extraction.
- Loading the current set of store records into the staging table.
- Reconciling the destination records against the source volume.
- Recording execution metrics for monitoring and audit purposes.

This package is one component in a wider staging framework that includes subject-area packages for customers, employees, products, promotions, vendors, sales facts, purchase facts, and other business domains.

## End-to-End Process Flow

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

The control-flow configuration reflects a sequential execution model, with green precedence constraints indicating that each task runs only after the preceding task completes successfully. This sequencing ensures that the package maintains integrity from source extraction through destination load and final audit logging.

## Control Flow Design

![Store staging control flow](/INTEGRATION_%20ETL_EDW/lmages/Screenshot%202026-09-15%20173245.png)

The package contains five operational steps, each serving a distinct role in the ETL lifecycle:

1. **Truncate Staging** clears the current staging table to support a full-refresh load of the store subject area.
2. **Source Count** calculates the number of records expected from the OLTP source query.
3. **Load Staging (Store)** executes the SSIS data flow responsible for extracting and inserting the store records.
4. **Des Count (Destination Count)** captures the total number of rows written to the staging destination.
5. **Metrics** records execution statistics and updates the package-control metadata used for operational monitoring.

The control-flow structure is designed to allow a reviewer validate the execution sequence and confirm each task contributes to the staging objective.

## Data Flow Analysis

![Store data flow](/INTEGRATION_%20ETL_EDW/lmages/Screenshot%202026-09-15%20174328.png)

The data-flow design includes two primary components:

- **Extract**, which reads the relevant source records from the OLTP database.
- **Load Store**, which writes the extracted rows to the staging destination table.

This arrangement supports a clear inspection path from source extraction to final storage. It also provides a suitable point for future enhancements such as data-quality checks, derived columns, validation rules, or error handling logic as the solution evolves.

## Source Extraction Logic

![OLTP source configuration](/INTEGRATION_%20ETL_EDW/lmages/Screenshot%202026-09-15%20174349.png)

The package uses the `TESCA_OLTP` OLE DB connection manager and the `SQL command` access mode to retrieve the store dataset with the following query:

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

This query reflects several key ETL design decisions:

- The source is the OLTP operational database rather than a flat-file extract.
- Store data is denormalized and enriched with city and state attributes during extraction.
- A load timestamp is added to support lineage, auditing, and troubleshooting.
- Inner joins restrict the output to stores with valid matching city and state reference records.

These decisions improve the usability of the staging data and ensure that downstream processes receive a consistent and business-readable representation of each store record.

## Metrics and Audit Mechanism

![Metrics task SQL](/INTEGRATION_%20ETL_EDW/lmages/Screenshot%202026-09-15%20174424.png)

The final Execute SQL Task records package execution results by using variables for the package identifier, source count, and destination count. The SQL logic is as follows:

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

The parameter markers denote SSIS placeholders, and their order must align with the parameter mapping configured in the Execute SQL Task. This step provides operational transparency in two ways:

- A historical metric record captures source-count, destination-count, package ID, and load timestamp data.
- The `control.package` table reflects the most recent execution time for the package.

This audit mechanism is essential for monitoring ETL reliability, validating load completeness, and tracing package performance over time.

## Connection Architecture

The SSIS project uses four project-level connection managers:

- `TESCA_OLTP`: transaction source system.
- `TESCA_STAGING`: staging destination for analytical preparation.
- `TESCA_CONTROL`: metadata and control tables used for package auditing and monitoring.
- `TESCA_EDW`: warehouse connection for subsequent downstream loading stages.

This separation reflects a layered data architecture in which transactional data is extracted from OLTP, standardized in staging, audited through the control layer, and then made available for enterprise warehouse processing.

## What this Design Communicates

The design demonstrates several engineering principles that are well suited to ETL implementation in a structured data warehouse environment:

- Clear separation between transactional source data and analytical preparation.
- Subject-area-oriented package design for maintainability and ownership.
- Sequential task execution supported by precedence constraints.
- Source-to-destination reconciliation through row-count validation.
- Operational traceability through metrics and package-control records.
- Early data enrichment through reference-table joins.
- Load timestamp tracking for lineage and issue investigation.

Overall, the package is effective as a staging pattern because it is easy to review, easy to troubleshoot, and well aligned with standard warehouse-loading practices. A reviewer can clearly trace the data from the OLTP query through the staging load and into the monitoring tables, confirming both process integrity and operational accountability.
