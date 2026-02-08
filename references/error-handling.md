# CloudKit Error Handling

Complete reference for CKError codes and handling strategies.

## Table of Contents
1. [Error Structure](#error-structure)
2. [Transient Errors (Auto-Retry)](#transient-errors-auto-retry)
3. [Conflict Errors](#conflict-errors)
4. [Catastrophic Errors (Halt)](#catastrophic-errors-halt)
5. [Quota & Limit Errors](#quota--limit-errors)
6. [Partial Failures](#partial-failures)
7. [Retry Strategy](#retry-strategy)

## Error Structure

```swift
do {
    try await database.save(record)
} catch let error as CKError {
    // Access error properties
    let code = error.code              // CKError.Code enum
    let userInfo = error.userInfo      // Additional context
    let retryAfter = error.retryAfterSeconds  // Suggested wait time
    let serverRecord = error.serverRecord     // For conflicts
    let partialErrors = error.partialErrorsByItemID  // For batch ops
}
```

## Transient Errors (Auto-Retry)

These errors are temporary. CKSyncEngine handles them automatically. For direct API usage, retry with backoff.

| Code | Description | Action |
|------|-------------|--------|
| `.networkFailure` | Network connection failed | Retry after connectivity |
| `.networkUnavailable` | No network available | Wait for connectivity |
| `.serviceUnavailable` | CloudKit server down (503) | Retry with backoff |
| `.zoneBusy` | Zone temporarily locked | Retry after delay |
| `.requestRateLimited` | Too many requests | Use `retryAfterSeconds` |
| `.operationCancelled` | Operation was cancelled | Retry if appropriate |

```swift
case .networkFailure, .networkUnavailable, .serviceUnavailable,
     .zoneBusy, .requestRateLimited:
    let delay = error.retryAfterSeconds ?? 30
    try await Task.sleep(for: .seconds(delay))
    // Retry operation
```

### Rate Limiting Details

Apple uses HTTP 503 (not 429) for rate limiting. Key mitigation strategies:

1. **One operation at a time** — Don't run concurrent CloudKit operations
2. **Minimum 2s between operations** — Even after success
3. **Exponential backoff on failure** — Double delay after each error
4. **Gradual recovery** — Decrease delay by 33% after success

```swift
var throttleDelay: TimeInterval = 2.0

func executeWithThrottle() async {
    try await Task.sleep(for: .seconds(throttleDelay))
    
    do {
        try await performOperation()
        throttleDelay = max(2.0, throttleDelay * 0.67)  // Reduce on success
    } catch {
        throttleDelay = min(300, throttleDelay * 2)  // Increase on failure
        throw error
    }
}
```

## Conflict Errors

| Code | Description | Action |
|------|-------------|--------|
| `.serverRecordChanged` | Record modified on server | Merge and retry |

```swift
case .serverRecordChanged:
    guard let serverRecord = error.serverRecord else {
        // Rare - refetch and retry
        return
    }
    
    // Server record has current change tag
    // Your options:
    
    // 1. Server wins (safest)
    updateLocalStore(with: serverRecord)
    
    // 2. Local wins (overwrite server)
    // Copy server's system fields to your record
    let merged = mergeSystemFields(from: serverRecord, to: localRecord)
    try await database.save(merged)
    
    // 3. Field-level merge
    let merged = mergeFields(local: localRecord, server: serverRecord)
    try await database.save(merged)
```

## Catastrophic Errors (Halt)

Stop syncing and alert user. These require intervention.

| Code | Description | Action |
|------|-------------|--------|
| `.notAuthenticated` | Not signed into iCloud | Prompt sign-in |
| `.badDatabase` | Database configuration error | Check entitlements |
| `.badContainer` | Container doesn't exist | Check container ID |
| `.internalError` | CloudKit internal error | Contact Apple |
| `.serverRejectedRequest` | Invalid request | Check record data |
| `.incompatibleVersion` | API version mismatch | Update app |
| `.permissionFailure` | No permission | Check sharing/capabilities |
| `.managedAccountRestricted` | MDM restriction | Inform user |

```swift
case .notAuthenticated:
    showAlert("Please sign into iCloud in Settings")
    suspendSync()

case .badDatabase, .badContainer:
    log.error("CloudKit configuration error: \(error)")
    suspendSync()
    // This is a developer error - check entitlements
```

## Quota & Limit Errors

| Code | Description | Action |
|------|-------------|--------|
| `.quotaExceeded` | Out of iCloud storage | Notify user |
| `.limitExceeded` | Too many records in batch | Split into chunks of 400 |
| `.assetFileModified` | Asset changed during upload | Re-create asset |
| `.assetFileNotFound` | Asset file doesn't exist | Skip or recreate |
| `.assetNotAvailable` | Asset not downloaded | Download first |

```swift
case .quotaExceeded:
    showAlert("iCloud storage full. Free up space or upgrade.")
    // Re-queue for later retry
    engine.state.add(pendingRecordZoneChanges: [.saveRecord(recordID)])

case .limitExceeded:
    // Split records into batches of 400
    let batches = records.chunked(into: 400)
    for batch in batches {
        try await saveRecords(batch)
    }
```

## Zone Errors

| Code | Description | Action |
|------|-------------|--------|
| `.zoneNotFound` | Zone doesn't exist | Create zone first |
| `.userDeletedZone` | User deleted zone | Recreate or stop |
| `.changeTokenExpired` | Token too old | Full re-sync |

```swift
case .zoneNotFound:
    // Create the zone, then retry
    let zone = CKRecordZone(zoneID: zoneID)
    try await database.save(zone)
    // Retry original operation

case .changeTokenExpired:
    // Clear cached token and do full fetch
    clearCachedToken()
    try await fetchAllRecords()
```

## Record Errors

| Code | Description | Action |
|------|-------------|--------|
| `.unknownItem` | Record doesn't exist | Remove from local store |
| `.constraintViolation` | Unique constraint violated | Handle duplicate |
| `.referenceViolation` | Referenced record missing | Create reference first |
| `.invalidArguments` | Bad record data | Fix record values |

```swift
case .unknownItem:
    // Record was deleted server-side
    deleteFromLocalStore(recordID)

case .referenceViolation:
    // Parent record must exist first
    // Save parent, then child
```

## Partial Failures

Batch operations can partially succeed. Check individual errors:

```swift
if let partialErrors = error.partialErrorsByItemID {
    for (itemID, itemError) in partialErrors {
        guard let recordError = itemError as? CKError else { continue }
        
        switch recordError.code {
        case .serverRecordChanged:
            handleConflict(for: itemID, error: recordError)
        case .unknownItem:
            removeFromLocal(itemID)
        default:
            requeueForRetry(itemID)
        }
    }
}
```

## Retry Strategy

```swift
class RetryManager {
    private var retryCount = 0
    private let maxRetries = 5
    
    func execute<T>(_ operation: () async throws -> T) async throws -> T {
        while retryCount < maxRetries {
            do {
                let result = try await operation()
                retryCount = 0  // Reset on success
                return result
            } catch let error as CKError {
                guard isRetryable(error) else {
                    throw error  // Catastrophic - don't retry
                }
                
                retryCount += 1
                let delay = calculateDelay(for: error)
                try await Task.sleep(for: .seconds(delay))
            }
        }
        throw CloudKitError.maxRetriesExceeded
    }
    
    private func isRetryable(_ error: CKError) -> Bool {
        switch error.code {
        case .networkFailure, .networkUnavailable, .serviceUnavailable,
             .zoneBusy, .requestRateLimited, .serverRecordChanged:
            return true
        default:
            return false
        }
    }
    
    private func calculateDelay(for error: CKError) -> TimeInterval {
        if let retryAfter = error.retryAfterSeconds {
            return retryAfter
        }
        // Exponential backoff: 2, 4, 8, 16, 32 seconds
        return pow(2.0, Double(retryCount))
    }
}
```

## Error Logging Best Practices

```swift
func logCloudKitError(_ error: CKError, operation: String) {
    let info: [String: Any] = [
        "operation": operation,
        "code": error.code.rawValue,
        "description": error.localizedDescription,
        "retryAfter": error.retryAfterSeconds ?? "none",
        "partialCount": error.partialErrorsByItemID?.count ?? 0
    ]
    
    switch error.code {
    case .notAuthenticated, .badDatabase, .badContainer:
        log.critical("CloudKit fatal: \(info)")
    case .quotaExceeded, .limitExceeded:
        log.warning("CloudKit limit: \(info)")
    default:
        log.info("CloudKit error: \(info)")
    }
}
```
