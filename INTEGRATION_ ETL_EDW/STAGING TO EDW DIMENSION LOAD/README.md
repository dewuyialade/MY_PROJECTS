# Staging to EDW: Employeee Dimension Pipeline (Slowly Changing Dimensions)

![Employee dimension control flow](/INTEGRATION_%20ETL_EDW/lmages/7.png)

## Project Report

This report documents an SQL Server Integration Services (SSIS) package that loads employee dimension data from the staging layer into the enterprise data warehouse (EDW). The design applies Slowly Changing Dimension principles before the records are then loaded into the warehouse.

The screenshots provide evidence of a three-part implementation:

1. A control flow that measures the dimension before and after processing.
2. A data flow that applies Slowly Changing Dimension (SCD) logic.
3. A metrics step that records the result of the load for audit and monitoring.

## Objective

The purpose of this package is to maintain an employee dimension while preserving the history of attributes that have business value over time. The data flow distinguishes between:

- New employee records.
- Changes that should update the current dimension row.
- Historical changes that should create a new version of the employee record.

This is a warehouse-loading pattern rather than a simple insert-only staging process. It allows downstream reporting to use the employee attributes that were valid at the relevant point in time.

## Evidence and Processing Order

### 1. Control flow orchestration

![Employee dimension control flow](/INTEGRATION_%20ETL_EDW/lmages/7.png)

The control-flow screenshot shows the package executing the following tasks in sequence:

```text
PreCount -> Load EDW (Employee) -> PostCount -> Metrics
```

The green precedence constraints indicate a successful-completion dependency between the tasks:

1. **PreCount** captures the employee-dimension count before the load begins.
2. **Load EDW (Employee)** runs the data flow that applies the SCD rules and writes to the EDW.
3. **PostCount** captures the count after the load completes.
4. **Metrics** persists the counts and other package execution details for operational review.

The package uses the project-level connection managers shown in the screenshot: `TESCA_CONTROL`, `TESCA_EDW`, and `TESCA_STAGING`. This demonstrates separation between control metadata, warehouse targets, and staged source data.

### 2. Employee dimension data flow

![Employee dimension data flow](/INTEGRATION_%20ETL_EDW/lmages/8.png)

The data-flow task is named **Load EDW (Employee)**. Its visible processing sequence is:

```text
Extract
    |
    v
Slowly Changing Dimension
    |-- Historical Attribute Inserts Output -> Type2Count -> Derived Column -> OLE DB Command
    |-- New Output                         -> CurrentCount --------------------------+
    |-- Changing Attribute Updates Output -> Type1Count -> OLE DB Command 1         |
                                                                                   v
                                                                             Union All
                                                                                   |
                                                                            Derived Column 1
                                                                                   |
                                                                            Insert Destination
```

#### Extract

The **Extract** component reads employee records from the upstream data layer (the staging environment) . 

#### Slowly Changing Dimension transformation

The **Slowly Changing Dimension** component compares incoming employee records with existing dimension records and routes them according to their change type. This is the central dimensional-modelling technique demonstrated by the package.

| Output | Meaning in the load | Downstream handling |
| --- | --- | --- |
| **Historical Attribute (Inserts Output)** | A Type 2 change where the previous version must remain available for historical reporting. | Counted by `Type2Count`, enriched by `Derived Column`, and sent to an `OLE DB Command` before being combined for insertion. |
| **New Output** | An employee record not currently represented in the dimension. | Counted by `CurrentCount` and sent to `Union All` for the destination load. |
| **Changing Attribute (Updates Output)** | A Type 1 change where the current dimension row is updated/overwritten without creating a new historical version. | Counted by `Type1Count` and processed by `OLE DB Command 1`. |

#### Type 1 and Type 2 behavior

The package demonstrates two complementary history-management strategies:

- **Type 1 processing** updates the existing warehouse row when a change does not require historical preservation. The `Changing Attribute` branch terminates at an OLE DB Command, which is consistent with an in-place update operation.
- **Type 2 processing** preserves the previous employee version and prepares a new version for insertion. The `Historical Attribute` branch uses a derived-column step and ultimately joins the common insert stream through `Union All`.



#### Union and final destination

`Union All` consolidates the new records and the prepared historical inserts into a common output. `Derived Column 1` performs final row preparation before **Insert Destination** writes the combined set to the employee dimension target in the EDW.

The Type 1 update branch remains separate because its purpose is to modify existing rows, while the unioned branches represent records that need to be inserted.

### 3. Metrics and audit logging

![Employee dimension metrics task](/INTEGRATION_%20ETL_EDW/lmages/9.png)

After the warehouse load, the **Metrics** task opens an Execute SQL Task configuration containing package parameters for:

- `PackageID`
- `PreCount`
- `CurrentCount`
- `Type1Count`
- `Type2Count`
- `PostCount`

The SQL follows this pattern:

```sql
DECLARE @PackageID INT = ?;
DECLARE @PreCount INT = ?;
DECLARE @CurrentCount INT = ?;
DECLARE @Type1Count INT = ?;
DECLARE @Type2Count INT = ?;
DECLARE @PostCount INT = ?;

BEGIN
    INSERT INTO control.metrics
        (PackageID, PreCount, CurrentCount, Type1Count, Type2Count, PostCount, LoadDate)
    SELECT
        @PackageID,
        @PreCount,
        @CurrentCount,
        @Type1Count,
        @Type2Count,
        @PostCount,
        GETDATE();

    UPDATE control.package
    SET LastUpdate = GETDATE()
    WHERE PackageID = @PackageID;
END;
```

The question marks represent SSIS parameter placeholders. Their order must match the parameter mapping configured in the Execute SQL Task.

This design provides two forms of operational evidence:

- `control.metrics` stores a historical record of the package counts and load timestamp.
- `control.package.LastUpdate` records the latest successful package activity for the identified package.

The separate Type 1 and Type 2 counts make the load more diagnosable than a single destination count. An engineer can easily identify whether a run primarily inserted new employees, updated current attributes, or created new historical versions.

## Reconciliation and Data-Quality Considerations

The control flow supports a basic before-and-after reconciliation using `PreCount` and `PostCount`. The additional branch counts provide context for interpreting the result:

```text
PreCount + Current Count (New inserts) + Historical inserts = PostCount
```

This expression is a conceptual audit relationship, not a claim that all values are mathematically additive in the implementation. Type 1 updates change existing rows, while new and Type 2 records increase the number of rows.



## Data Engineering Concepts Demonstrated

This implementation provides evidence of the following concepts:

- Layered ETL architecture across staging, control, and EDW databases.
- SSIS control-flow orchestration using precedence constraints.
- Slowly Changing Dimension processing for both Type 1 and Type 2 attributes.
- Separation of insert and update paths based on business meaning.
- Use of `Union All` to consolidate compatible insert streams.
- Derived-column transformations for final warehouse-row preparation.
- OLE DB Command operations for targeted dimension updates.
- Pre-load and post-load row-count reconciliation.
- Branch-level metrics for operational observability.
- Control-table auditing with package identifiers and load timestamps.

## Conclusion

 This package represents a structured and auditable EDW dimension-load process that complements the upstream staging workflows in this project.

The package demonstrates a practical transition from raw or staged employee data to a historized warehouse dimension. Its strongest design feature is the explicit separation of new records, current-row updates, and historical inserts. That separation makes the business rules visible in the SSIS canvas and gives the metrics task enough detail to support post-run investigation.

 