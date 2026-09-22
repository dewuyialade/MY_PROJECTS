# Flat File to Staging: Overtime Fact ETL

![Overtime fact control flow](/INTEGRATION_%20ETL_EDW/lmages/1.png)

## Overview

The attached screenshots explain an SSIS package associated with an overtime-fact staging process. The images show a pipeline that reads overtime data from a flat-file source, applies date-based routing, counts and prepares the rows, and loads them into a SQL Server staging table.

Unlike a conventional OLTP extraction, the source is a flat file represented by the package connection manager `OVERTIME_FACT`. 

## Processing order

### 1. Control flow: package orchestration

![Overtime fact control flow](/INTEGRATION_%20ETL_EDW/lmages/1.png)

The image above image shows the package-level control flow:

1. **Truncate Staging** clears the existing staging data to avoid violation of primary key constraints and possible data duplication .
2. **EDW Count** obtains a count of the data previously existing in the EDW before the main data flow runs. 
3. **Load Overtime** runs the SSIS data flow.
4. **Des Count** records the number of rows loaded or available at the destination.
5. **Metrics** records execution statistics for monitoring and audit.


### 2. Data flow: extract, route, standardize, and load

![Overtime fact data flow](/INTEGRATION_%20ETL_EDW/lmages/2.png)

The second image expands the **Load Overtime** data-flow task. 

1. **Extract** reads the overtime rows from the flat-file source.
2. A conditional routing step evaluates the overtime start date to determine how much data needs to be extracted.
3. The flow branches into **Beginning to N-1** and **N-1** paths.
4. **Union All** combines the routed streams into one output.
5. **Source Count** captures the row count after the streams are combined.
6. **Load Date** adds or derives an ETL load timestamp.
7. **Balanced Data Distributor** divides the rows across multiple load paths for faster execution.



### 3. Conditional split: current or fetchable overtime rows

![FetchData conditional split](/INTEGRATION_%20ETL_EDW/lmages/3.png)

The third image shows a conditional split output named **FetchData** with the condition:

```text
(DT_DATE)StartOvertime <= DATEADD("day", -1, GETDATE())
```

In practical terms, rows whose `StartOvertime` date is on or before yesterday are routed to `FetchData`. This prevents future-dated overtime records from being extracted. This is important because we do not want to interfere with the day's business by  trying to extract data during active business hours. This may cause a problem at the sales front and it needs to be totally avoided.

The default output is named **FutureData**, which provides a route for rows that do not satisfy the condition. This is useful because unmatched rows are not silently lost, they are explicitly identified as future-dated or otherwise outside the main condition.

### 4. Conditional split: beginning of the N-1 boundary

![Begin to N-1 conditional split](/INTEGRATION_%20ETL_EDW/lmages/4.png)

The fourth image shows an output named **Begin to N-1** with the condition:

```text
(DT_DATE)StartOvertime <= DATEADD("day", -1, GETDATE())
```

This captures all records from the beginning up until the day before the current date. This would be the output if the count of the EDW from one of the previous steps is zero.


### 5. Conditional split: N-1 boundary

![N-1 conditional split](/INTEGRATION_%20ETL_EDW/lmages/5.png)

The fifth image shows an output named **N-1** with the condition:

```text
(DT_DATE)StartOvertime == DATEADD("day", -1, GETDATE())
```

This routes records whose start date is exactly yesterday into a separate stream. Keeping the N-1 stream distinct can support a rolling-load strategy, separate reconciliation, or special handling for the most recently completed business day. This would be the output returned if the count of the EDW is not null.



### 6. OLE DB destination: staging table

![Overtime staging destination](/INTEGRATION_%20ETL_EDW/lmages/6.png)

The final image shows the OLE DB Destination Editor configured with:

- **Connection manager:** `TESCA_STAGING`
- **Data access mode:** `Table or view - fast load`
- **Destination table:** `[hr].[stgovertime]`


The fast-load mode is intended for efficient bulk insertion.  while constraint checking helps protect the staging table's structural rules.



## Why this is a flat-file pipeline

This process differs from an OLTP-to-staging package in several important ways:

- The source is a file-based extract rather than normalized transactional tables.
- There is no visible source-side SQL join or database query in the screenshots.
- File availability, encoding, duplicate files, and file archival become part of pipeline reliability.
- Data validation must compensate for the weaker constraints commonly present in flat files.


The `OVERTIME_FACT` package connection manager is the strongest visual evidence of the file-based source. 

## What the design communicates to a data-expert panel

The screenshots demonstrate an attempt to build a controlled and auditable ingestion process with:

- A clearly separated flat-file source and staging database destination.
- A repeatable full-refresh staging pattern.
- Explicit date-based business rules for current, prior-day, and future records.
- A Union All step that brings routed records back into a common loading stream.
- Source and destination row-count instrumentation.
- A derived load date for lineage and troubleshooting.
- Parallelized or balanced destination loading for performance.
- Fast-load insertion with constraint validation.
- Control and metrics tasks for operational monitoring.

The architectural idea is the separation of concerns: file ingestion, business-date routing, row-count tracking, load-date stamping, and database loading are visible as distinct steps that can be tested independently.

