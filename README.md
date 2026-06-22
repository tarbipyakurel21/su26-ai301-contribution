# Contribution 1: Classify Iceberg CommitFailedException as TRANSACTION_CONFLICT
 #16770

**Contribution Number:** 1  
**Student:** Tarbi Pyakurel
**Issue:** (https://github.com/trinodb/trino/issues/16770) 
**Status:** Phase II Complete

---

## Why I Chose This Issue
I chose this issue because this is an error handling being generic instead of specific. It would really help to figure out issues clear with proper error handling plus it matches my skills level as I am pointed in a direction to wrap the generic errors as a certain specific execption class named Trino Execption. I hope to learn error handling and database engines from this issue.

---

## Understanding the Issue

### Problem Description
When multiple writers try to commit to the same Iceberg table at the same time, Iceberg throws a CommitFailedException. Trino catches it but maps it to the wrong error code (ICEBERG_COMMIT_ERROR), or in some places doesn't catch it at all, causing a generic GENERIC_INTERNAL_ERROR.

### Expected Behavior
CommitFailedException should produce a TrinoException with error code TRANSACTION_CONFLICT, which tells the client there was a concurrent write conflict.

### Current Behavior
The error surfaces as ICEBERG_COMMIT_ERROR (an external error) with no hint that it was caused by concurrent writers.

### Affected Components
plugin/trino-iceberg — IcebergMetadata.java and AbstractTrinoCatalog.java.

---

## Reproduction Process

### Environment Setup
Java 25, Maven, macOS
Cloned my fork and no external services was needed
The in-memory Iceberg catalog is enough to trigger the conflict locally however I wrote an additional assert function in the test method to verify the issue more properly.

### Steps to Reproduce

1.
Add an error code assertion to testConcurrentOverlappingUpdate in TestIcebergLocalConcurrentWrites.java (assert the caught exception has error code TRANSACTION_CONFLICT)

2.
Run from plugin/trino-iceberg/:
mvn test -Dtest="TestIcebergLocalConcurrentWrites#testConcurrentOverlappingUpdate" -DfailIfNoTests=false

3.
Observed: 3/3 runs fail — exception has code ICEBERG_COMMIT_ERROR, not TRANSACTION_CONFLICT

### Reproduction Evidence

- **Commit showing reproduction:** 
- **Screenshots/logs:** <img width="1470" height="956" alt="Screenshot 2026-06-14 at 11 48 39 PM" src="https://github.com/user-attachments/assets/87baf29a-7bd6-472a-bd2a-081a8f57b984" />

- **My findings:** :
Error message: "Failed to commit the transaction during write: Found conflicting files..."
Thrown from: IcebergMetadata.commitTransaction(IcebergMetadata.java:2215)

All catalog implementations (Glue, HMS, File, Nessie) only throw CommitFailedException for actual write conflicts — infrastructure failures go to CommitStateUnknownException instead. So CommitFailedException is always a TRANSACTION_CONFLICT.

---

## Solution Approach

### Analysis

In IcebergMetadata.commitTransaction() (line 2209), CommitFailedException is grouped in a multi-catch with infrastructure errors (UncheckedIOException, CommitStateUnknownException, etc.) and all mapped to ICEBERG_COMMIT_ERROR. It needs its own catch block with TRANSACTION_CONFLICT. Two callsites in AbstractTrinoCatalog.java call transaction.commitTransaction() with no exception handling at all.

### Proposed Solution

Split CommitFailedException into its own catch block in IcebergMetadata.java and map it to TRANSACTION_CONFLICT. Add try-catch wrapping to the two unguarded callsites in AbstractTrinoCatalog.java.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** CommitFailedException means a concurrent write conflict, not an infrastructure failure. It's currently treated the same as infrastructure errors, giving users a misleading error code.

**Match:** TRANSACTION_CONFLICT already exists in StandardErrorCode.java (line 41) and is used elsewhere in Trino for optimistic concurrency failures.

**Plan:** 
1.
IcebergMetadata.java — extract CommitFailedException from the multi-catch in commitTransaction() and commitUpdate(), throw TrinoException(TRANSACTION_CONFLICT, ...). Add import static io.trino.spi.StandardErrorCode.TRANSACTION_CONFLICT.
2.
AbstractTrinoCatalog.java — wrap bare transaction.commitTransaction() in createMaterializedViewStorageTable() (line 365) and add a catch (CommitFailedException) to the try-finally in createOrReplaceMaterializedView() (line 554).
3.
TestIcebergLocalConcurrentWrites.java — already updated with error code assertion.


**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
IcebergMetadata — commitUpdate() and commitTransaction()
AbstractTrinoCatalog — createMaterializedViewStorageTable() and replaceMaterializedViewStorageTable()
MigrationUtils — addFiles()
MigrateProcedure — migrate()

- **Key commits:**
https://github.com/trinodb/trino/pull/29982/changes/64efe199e72238839b629bb4fb03ecfcdae7d74a
https://github.com/trinodb/trino/pull/29982/changes/a20479557451ce642d47d864564199925fb4fe77

- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** https://github.com/trinodb/trino/pull/29982

**PR Description:** Map CommitFailedException to TRANSACTION_CONFLICT instead of ICEBERG_COMMIT_ERROR across all Iceberg commit calls. CommitFailedException is thrown exclusively for concurrent write conflicts — infrastructure failures use CommitStateUnknownException. Using TRANSACTION_CONFLICT (a user-error type) allows clients to distinguish a retriable conflict from a real infrastructure failure.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
