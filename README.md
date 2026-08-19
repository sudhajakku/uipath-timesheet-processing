# UiPath Timesheet Processing

UiPath technical assignment implementing automated processing of structured `.txt` timesheet files using a **Dispatcher–Performer** architecture.

The solution reads incoming timesheets, extracts and validates employee data, writes only valid records to Excel, logs every processed record, moves source files into Valid/Invalid folders, and uses UiPath Orchestrator Queue transaction status for business/system exception tracking and daily reporting.

## Architecture

![Solution Architecture](Documentation/Architecture.png)

**Implemented flow**

`Input Folder -> Dispatcher -> Orchestrator Queue -> REFramework Performer -> Excel / Logs / Processed Folders`

### Dispatcher
A lightweight UiPath process that:

- Loads settings from `Config.xlsx`
- Finds `.txt` files in the configured Input folder
- Reads the source file content
- Adds one Orchestrator Queue item per file
- Stores `FileName`, `FilePath`, and `RawContent` in the queue item
- Uses the file name as the queue reference
- Handles duplicate queue references without stopping the entire batch

### Performer
A UiPath **Robotic Enterprise Framework (REFramework)** process that:

- Retrieves queue transactions
- Extracts timesheet fields from the queued raw text
- Validates required fields and business rules
- Writes valid records to `Timesheet_Data.xlsx`
- Logs valid and invalid records to `Timesheet_Log.xlsx`
- Moves source files to `Processed/Valid` or `Processed/Invalid`
- Throws `BusinessRuleException` for validation failures
- Allows unexpected technical exceptions to follow the REFramework System Exception path
- Generates an end-of-run system error report and daily processing summary

## Input Format

Expected structured text format:

```text
Employee Name: John Doe
Employee ID: 12345
Week Starting: 01-Apr-2025
Hours Worked: 38
Department: IT
```

## Validation Rules

The Performer validates the following:

- Employee Name must not be empty
- Employee ID must not be empty
- Week Starting must not be empty
- Week Starting must match `dd-MMM-yyyy`
- Hours Worked must not be empty
- Hours Worked must be numeric
- Hours Worked must be between `0` and the configured maximum (`60` by default)
- Department must not be empty

Regex extraction is restricted to the current line so an empty field cannot accidentally capture the next field.

## Output Files

### `Timesheet_Data.xlsx`
Contains **valid records only**.

Columns:

- Employee Name
- Employee ID
- Week Starting
- Hours Worked
- Department

The Performer checks existing data before appending to reduce duplicate business writes during retries.

### `Timesheet_Log.xlsx`
Audit log for **all valid and invalid processed timesheets**.

Includes:

- File Name
- Employee Name
- Employee ID
- Week Starting
- Hours Worked
- Department
- Status
- Validation Message
- Timestamp

### `Error_Log.xlsx`
Generated from Orchestrator Queue transaction information for unexpected Application/System failures.

Includes:

- File Name
- Error Message
- Timestamp

## File Routing

After processing:

```text
Processed/
├── Valid/
└── Invalid/
```

- Valid records are moved to `Processed/Valid`
- Business-invalid records are moved to `Processed/Invalid`
- File movement includes retry-safe handling for previously moved files

## Exception Handling

### Business Exceptions
Validation failures are treated as business exceptions.

Flow:

`Validation Failure -> Log Invalid -> Move to Processed/Invalid -> BusinessRuleException -> Queue Failed (Business)`

Examples:

- Missing required field
- Invalid date
- Nonnumeric hours
- Hours below 0 or above 60

Business exceptions are not treated as transient technical failures.

### System Exceptions
Unexpected technical failures follow the standard REFramework System Exception path.

Examples:

- Unreadable/missing source file
- Excel/workbook access error
- File movement conflict
- Permission or infrastructure failure

REFramework handles logging, retry behavior, and final queue transaction status. Final Application/System failures are included in `Error_Log.xlsx`.

## End-of-Run Reporting

The Performer uses `GetQueueItemsPaginated.xaml` to retrieve queue transactions in batches of 100.

This supports:

### Daily Summary
Logs counts of:

- Valid transactions
- Invalid / Business Exception transactions
- System / Application Exception transactions
- Total processed transactions

### System Error Report
Writes Application/System failures from the queue into `Error_Log.xlsx`.

Reporting is executed once from the REFramework End Process state and is isolated from standard application cleanup.

## Configuration

Configuration is stored in each process `Config.xlsx`.

### Dispatcher configuration
Typical settings:

| Setting | Purpose |
|---|---|
| `OrchestratorQueueName` | Queue used for timesheet transactions |
| `OrchestratorQueueFolder` | Orchestrator folder/workspace |
| `InputFolder` | Incoming `.txt` folder |
| `FilePattern` | File search pattern, e.g. `*.txt` |

### Performer configuration
Typical settings:

| Setting | Purpose |
|---|---|
| `OrchestratorQueueName` | Queue used for timesheet transactions |
| `OrchestratorQueueFolder` | Orchestrator folder/workspace |
| `MaxHours` | Maximum permitted hours, default `60` |
| `TimesheetDataFilePath` | Valid-record Excel output |
| `TimesheetDataSheetName` | Valid-record worksheet |
| `TimesheetLogFilePath` | Valid/invalid audit log |
| `TimesheetLogSheetName` | Audit-log worksheet |
| `ErrorLogFilePath` | System error report |
| `ErrorLogSheetName` | Error-log worksheet |
| `ProcessedValidFolder` | Destination for valid files |
| `ProcessedInvalidFolder` | Destination for invalid files |

No credentials are stored in Config files.

## Queue Payload Design

Each queue item contains:

- `FileName`
- `FilePath`
- `RawContent`

For this assignment, the input is a small structured text file, so capturing `RawContent` in the queue creates a stable transaction payload and avoids rereading mutable file contents during processing.

For a production solution involving large documents, the preferred approach would be immutable shared/object storage with the queue containing only identifiers, location/version metadata, and an integrity reference such as a hash.

## Project Structure

```text
uipath-timesheet-processing/
│
├── Timesheet_Dispatcher/
│   ├── Main.xaml
│   ├── Data/
│   │   └── Config.xlsx
│   └── Framework/
│
├── Timesheet_Performer/
│   ├── Main.xaml
│   ├── Process.xaml
│   ├── Data/
│   │   └── Config.xlsx
│   └── Workflows/
│       ├── ExtractTimesheetData.xaml
│       ├── ValidateTimesheet.xaml
│       ├── WriteTimesheetData.xaml
│       ├── LogTimesheet.xaml
│       ├── MoveProcessedFile.xaml
│       ├── Queue/
│       │   └── GetQueueItemsPaginated.xaml
│       └── Reporting/
│           ├── GenerateErrorLog.xaml
│           └── GenerateDailySummary.xaml
│
├── Documentation/
│   ├── SolutionArchitecture.png
        ExceptionHandlingArchitecture.png
        
│
└── README.md
```

> Folder names may vary slightly depending on the UiPath Studio project layout.

## Orchestrator Setup

Create an Orchestrator Queue named:

```text
TimesheetProcessing
```

Recommended:

- Enable unique references
- Configure queue retry behavior as required
- Ensure the robot/user has access to the queue in the configured folder/workspace

## How to Run

1. Configure Dispatcher and Performer `Config.xlsx`.
2. Create/configure the `TimesheetProcessing` Orchestrator Queue.
3. Place timesheet `.txt` files in the Dispatcher Input folder.
4. Run `Timesheet_Dispatcher`.
5. Confirm queue items were created.
6. Run `Timesheet_Performer`.
7. Review:
   - `Timesheet_Data.xlsx`
   - `Timesheet_Log.xlsx`
   - `Error_Log.xlsx` when system errors exist
   - `Processed/Valid`
   - `Processed/Invalid`
   - Orchestrator Queue transaction statuses and logs

## Testing

Testing covered both clean and invalid input scenarios, including:

- Standard valid record
- `0` hours boundary
- `60` hours boundary
- Decimal hours
- Missing Employee Name
- Missing Employee ID
- Missing Week Starting
- Missing Hours Worked
- Missing Department
- Whitespace-only fields
- Nonnumeric hours
- Negative hours
- Hours greater than 60
- Invalid date format
- Impossible date

A 30-file regression test pack was used with:

```text
15 expected Valid
15 expected Invalid
```

The expected clean-run result is:

```text
Successful:         15
Business Invalid:   15
System Errors:       0
```

Only the 15 valid records should appear in `Timesheet_Data.xlsx`.

## Design Decisions

- **Dispatcher–Performer:** separates file discovery/queue creation from transactional business processing.
- **REFramework Performer:** provides robust transaction lifecycle, retries, exception handling, and queue status management.
- **RawContent in Queue:** stabilizes the small assignment payload at dispatch time.
- **Business vs System Exceptions:** validation failures are not retried as technical failures.
- **Workbook Activities:** used for Excel persistence without requiring Excel UI automation.
- **Queue-Based Reporting:** Orchestrator is the transaction-status source of truth.
- **Pagination:** reporting supports more than 100 queue records.
- **Retry-Safe Output:** valid data writes include a duplicate check and file movement handles already-moved files safely.

## Assumptions

- Incoming timesheets follow the expected label-based text structure.
- The Input folder contains timesheet `.txt` files only.
- Employee ID is treated as text to preserve leading zeros.
- The configured date format is `dd-MMM-yyyy`.
- Runtime output files and temporary UiPath artifacts are not intended to be committed to source control.

## Assignment Coverage

The solution implements:

- `.txt` timesheet ingestion
- Field extraction
- Required-field validation
- Hours validation from 0–60
- Valid/invalid Excel audit logging
- Valid-record Excel output
- Valid/Invalid source-file routing
- Config-driven paths/settings
- Business and System exception handling
- Unexpected-error reporting
- Daily processing summary
- Dispatcher–Performer architecture
- High-level implemented-solution architecture diagram
