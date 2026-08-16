# uipath-timesheet-processing
UiPath RPA solution for automated timesheet processing using a lightweight Dispatcher and an REFramework-based Performer.
## Architecture Decisions

### Queue Payload Strategy

For this assignment, the Dispatcher reads each timesheet file and includes the raw
text content in the Orchestrator Queue transaction. This ensures that the Performer
processes the exact content captured at dispatch time and avoids dependency on a
mutable local file path.

This approach is appropriate because the assignment uses small structured text
timesheets.

For larger production documents, the solution would store the document in immutable shared storage or object storage and place only the document identifier, storage location, version/hash, and required metadata in the Orchestrator Queue.