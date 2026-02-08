# CloudKit Operations

Complete reference for CKOperation classes and batch operations.

## Table of Contents
1. [Operation Hierarchy](#operation-hierarchy)
2. [CKDatabaseOperation](#ckdatabaseoperation)
3. [Record Operations](#record-operations)
4. [Zone Operations](#zone-operations)
5. [Subscription Operations](#subscription-operations)
6. [Fetch Operations](#fetch-operations)
7. [Notification Operations](#notification-operations)
8. [Operation Groups](#operation-groups)
9. [Configuration & QoS](#configuration--qos)
10. [Best Practices](#best-practices)

## Operation Hierarchy

```
CKOperation (base class)
├── CKDatabaseOperation
│   ├── CKModifyRecordsOperation
│   ├── CKFetchRecordsOperation
│   ├── CKQueryOperation
│   ├── CKModifyRecordZonesOperation
│   ├── CKFetchRecordZonesOperation
│   ├── CKFetchRecordZoneChangesOperation
│   ├── CKFetchDatabaseChangesOperation
│   ├── CKModifySubscriptionsOperation
│   └── CKFetchSubscriptionsOperation
├── CKFetchShareMetadataOperation
├── CKFetchShareParticipantsOperation
├── CKAcceptSharesOperation
├── CKDiscoverUserIdentitiesOperation
└── CKFetchNotificationChangesOperation
```

## CKDatabaseOperation

Base class for database operations:

```swift
let operation = CKModifyRecordsOperation(...)

// Common properties
operation.database = container.privateCloudDatabase
operation.qualityOfService = .userInitiated
operation.configuration.timeoutIntervalForRequest = 30
operation.configuration.timeoutIntervalForResource = 60

// Add to database (auto-sets database property)
database.add(operation)

// Or add to container for non-database operations
container.add(operation)
```

## Record Operations

### CKModifyRecordsOperation

Save and delete multiple records:

```swift
let recordsToSave = [record1, record2, record3]
let recordIDsToDelete = [oldRecordID1, oldRecordID2]

let operation = CKModifyRecordsOperation(
    recordsToSave: recordsToSave,
    recordIDsToDelete: recordIDsToDelete
)

// Save policy
operation.savePolicy = .changedKeys  // Only modified fields

// Atomic - all or nothing
operation.isAtomic = true

// Per-record callbacks
operation.perRecordSaveBlock = { recordID, result in
    switch result {
    case .success(let record):
        print("Saved: \(recordID)")
    case .failure(let error):
        print("Failed \(recordID): \(error)")
    }
}

operation.perRecordDeleteBlock = { recordID, result in
    switch result {
    case .success:
        print("Deleted: \(recordID)")
    case .failure(let error):
        print("Delete failed \(recordID): \(error)")
    }
}

// Completion
operation.modifyRecordsResultBlock = { result in
    switch result {
    case .success:
        print("All operations completed")
    case .failure(let error):
        print("Batch failed: \(error)")
    }
}

database.add(operation)
```

### Save Policies

```swift
// Only send fields that changed (safest for concurrent edits)
operation.savePolicy = .changedKeys

// Send all fields (overwrites everything)
operation.savePolicy = .allKeys

// Fail if server record changed (default)
operation.savePolicy = .ifServerRecordUnchanged
```

### CKFetchRecordsOperation

Fetch multiple records by ID:

```swift
let recordIDs = [id1, id2, id3]
let operation = CKFetchRecordsOperation(recordIDs: recordIDs)

// Only fetch specific keys (optimization)
operation.desiredKeys = ["title", "content", "modifiedAt"]

// Per-record callback
operation.perRecordResultBlock = { recordID, result in
    switch result {
    case .success(let record):
        processRecord(record)
    case .failure(let error):
        handleError(error, for: recordID)
    }
}

// Completion
operation.fetchRecordsResultBlock = { result in
    switch result {
    case .success:
        print("Fetch completed")
    case .failure(let error):
        print("Fetch failed: \(error)")
    }
}

database.add(operation)
```

### CKQueryOperation

Query records with predicates:

```swift
let predicate = NSPredicate(format: "category == %@ AND createdAt > %@", "work", oneWeekAgo)
let query = CKQuery(recordType: "Note", predicate: predicate)
query.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

let operation = CKQueryOperation(query: query)
operation.zoneID = myZoneID  // Optional: specific zone
operation.desiredKeys = ["title", "content"]
operation.resultsLimit = 50  // Pagination

// Collect results
var fetchedRecords: [CKRecord] = []

operation.recordMatchedBlock = { recordID, result in
    if case .success(let record) = result {
        fetchedRecords.append(record)
    }
}

operation.queryResultBlock = { result in
    switch result {
    case .success(let cursor):
        if let cursor = cursor {
            // More results available - fetch next page
            fetchNextPage(cursor: cursor)
        } else {
            // All results fetched
            processResults(fetchedRecords)
        }
    case .failure(let error):
        print("Query failed: \(error)")
    }
}

database.add(operation)
```

### Cursor-based Pagination

```swift
func fetchAllRecords(cursor: CKQueryOperation.Cursor? = nil) {
    let operation: CKQueryOperation
    
    if let cursor = cursor {
        operation = CKQueryOperation(cursor: cursor)
    } else {
        let query = CKQuery(recordType: "Note", predicate: NSPredicate(value: true))
        operation = CKQueryOperation(query: query)
    }
    
    operation.resultsLimit = 100
    
    operation.recordMatchedBlock = { _, result in
        if case .success(let record) = result {
            allRecords.append(record)
        }
    }
    
    operation.queryResultBlock = { result in
        if case .success(let nextCursor) = result {
            if let nextCursor = nextCursor {
                fetchAllRecords(cursor: nextCursor)  // Recursive
            } else {
                print("Fetched all \(allRecords.count) records")
            }
        }
    }
    
    database.add(operation)
}
```

## Zone Operations

### CKModifyRecordZonesOperation

Create or delete zones:

```swift
let zoneID = CKRecordZone.ID(zoneName: "MyZone", ownerName: CKCurrentUserDefaultName)
let zone = CKRecordZone(zoneID: zoneID)

let operation = CKModifyRecordZonesOperation(
    recordZonesToSave: [zone],
    recordZoneIDsToDelete: nil
)

operation.perRecordZoneSaveBlock = { zoneID, result in
    switch result {
    case .success(let zone):
        print("Zone created: \(zone.zoneID)")
    case .failure(let error):
        print("Zone creation failed: \(error)")
    }
}

operation.modifyRecordZonesResultBlock = { result in
    // Overall completion
}

database.add(operation)
```

### CKFetchRecordZonesOperation

Fetch zone information:

```swift
// Fetch specific zones
let operation = CKFetchRecordZonesOperation(recordZoneIDs: [zoneID1, zoneID2])

// Or fetch all zones
let allZonesOperation = CKFetchRecordZonesOperation.fetchAllRecordZonesOperation()

operation.perRecordZoneResultBlock = { zoneID, result in
    if case .success(let zone) = result {
        print("Zone: \(zone.zoneID.zoneName)")
    }
}

database.add(operation)
```

### CKFetchRecordZoneChangesOperation

Incremental sync - fetch only changes since last sync:

```swift
var serverChangeTokens: [CKRecordZone.ID: CKServerChangeToken] = loadTokens()

let zoneIDs = [zone1ID, zone2ID]
var configurations: [CKRecordZone.ID: CKFetchRecordZoneChangesOperation.ZoneConfiguration] = [:]

for zoneID in zoneIDs {
    let config = CKFetchRecordZoneChangesOperation.ZoneConfiguration()
    config.previousServerChangeToken = serverChangeTokens[zoneID]
    configurations[zoneID] = config
}

let operation = CKFetchRecordZoneChangesOperation(
    recordZoneIDs: zoneIDs,
    configurationsByRecordZoneID: configurations
)

// New/modified records
operation.recordWasChangedBlock = { recordID, result in
    if case .success(let record) = result {
        saveLocally(record)
    }
}

// Deleted records
operation.recordWithIDWasDeletedBlock = { recordID, recordType in
    deleteLocally(recordID)
}

// Zone completion - save new token
operation.recordZoneFetchResultBlock = { zoneID, result in
    switch result {
    case .success((let serverChangeToken, _, _)):
        serverChangeTokens[zoneID] = serverChangeToken
        saveTokens(serverChangeTokens)
    case .failure(let error):
        if let ckError = error as? CKError, ckError.code == .changeTokenExpired {
            // Token expired - do full fetch
            serverChangeTokens[zoneID] = nil
        }
    }
}

database.add(operation)
```

### CKFetchDatabaseChangesOperation

Fetch database-level changes (zone creations/deletions):

```swift
var databaseChangeToken: CKServerChangeToken? = loadDatabaseToken()

let operation = CKFetchDatabaseChangesOperation(previousServerChangeToken: databaseChangeToken)

operation.recordZoneWithIDWasDeletedBlock = { zoneID in
    print("Zone deleted: \(zoneID)")
    deleteLocalZoneData(zoneID)
}

operation.recordZoneWithIDChangedBlock = { zoneID in
    print("Zone changed: \(zoneID)")
    zonesToFetch.append(zoneID)
}

operation.fetchDatabaseChangesResultBlock = { result in
    switch result {
    case .success((let newToken, _)):
        databaseChangeToken = newToken
        saveDatabaseToken(newToken)
        // Now fetch record changes for modified zones
        fetchZoneChanges(for: zonesToFetch)
    case .failure(let error):
        print("Database changes fetch failed: \(error)")
    }
}

database.add(operation)
```

## Subscription Operations

### CKModifySubscriptionsOperation

Create or delete subscriptions:

```swift
// Create zone subscription
let zoneSubscription = CKRecordZoneSubscription(zoneID: zoneID)
zoneSubscription.notificationInfo = CKSubscription.NotificationInfo()
zoneSubscription.notificationInfo?.shouldSendContentAvailable = true

// Create query subscription
let predicate = NSPredicate(format: "priority > %d", 5)
let querySubscription = CKQuerySubscription(
    recordType: "Task",
    predicate: predicate,
    subscriptionID: "high-priority-tasks",
    options: [.firesOnRecordCreation, .firesOnRecordUpdate]
)

let operation = CKModifySubscriptionsOperation(
    subscriptionsToSave: [zoneSubscription, querySubscription],
    subscriptionIDsToDelete: nil
)

operation.perSubscriptionSaveBlock = { subscriptionID, result in
    switch result {
    case .success(let subscription):
        print("Subscription saved: \(subscriptionID)")
    case .failure(let error):
        print("Subscription failed: \(error)")
    }
}

database.add(operation)
```

### CKFetchSubscriptionsOperation

Fetch existing subscriptions:

```swift
// Fetch specific subscriptions
let operation = CKFetchSubscriptionsOperation(subscriptionIDs: ["sub1", "sub2"])

// Or fetch all
let allOperation = CKFetchSubscriptionsOperation.fetchAllSubscriptionsOperation()

operation.perSubscriptionResultBlock = { subscriptionID, result in
    if case .success(let subscription) = result {
        print("Found subscription: \(subscription)")
    }
}

database.add(operation)
```

## Notification Operations

### CKFetchNotificationChangesOperation

Fetch push notification history:

```swift
var notificationChangeToken: CKServerChangeToken? = loadNotificationToken()

let operation = CKFetchNotificationChangesOperation(
    previousServerChangeToken: notificationChangeToken
)

operation.notificationChangedBlock = { notification in
    guard let queryNotification = notification as? CKQueryNotification else { return }
    
    if let recordID = queryNotification.recordID {
        print("Record changed: \(recordID)")
    }
}

operation.fetchNotificationChangesResultBlock = { result in
    switch result {
    case .success((let newToken, _)):
        notificationChangeToken = newToken
        saveNotificationToken(newToken)
    case .failure(let error):
        print("Failed to fetch notifications: \(error)")
    }
}

container.add(operation)
```

## Operation Groups

Group related operations:

```swift
let group = CKOperationGroup()
group.expectedSendSize = .megabytes
group.expectedReceiveSize = .megabytes
group.name = "Initial Sync"

let fetchOp = CKFetchRecordZoneChangesOperation(...)
fetchOp.group = group

let modifyOp = CKModifyRecordsOperation(...)
modifyOp.group = group

// Operations in same group share network resources
database.add(fetchOp)
database.add(modifyOp)
```

## Configuration & QoS

### Quality of Service

```swift
operation.qualityOfService = .userInitiated  // User waiting
operation.qualityOfService = .utility        // Background sync
operation.qualityOfService = .background     // Low priority

// Higher QoS = faster scheduling, more resources
```

### Timeouts

```swift
operation.configuration.timeoutIntervalForRequest = 30   // Per-request
operation.configuration.timeoutIntervalForResource = 300 // Total operation
```

### Long-Lived Operations

For operations that should survive app termination:

```swift
operation.configuration.isLongLived = true
operation.configuration.longLivedOperationWasPersistedBlock = {
    print("Operation persisted - will resume on next launch")
}

// On next app launch, fetch persisted operations
container.fetchAllLongLivedOperationIDs { operationIDs, error in
    guard let ids = operationIDs else { return }
    for id in ids {
        container.fetchLongLivedOperation(withID: id) { operation, error in
            if let op = operation {
                // Resume operation
                container.add(op)
            }
        }
    }
}
```

## Best Practices

### 1. Batch Size Limits

```swift
// Max 400 records per operation
let batches = records.chunked(into: 400)
for batch in batches {
    let op = CKModifyRecordsOperation(recordsToSave: batch)
    database.add(op)
    try await Task.sleep(for: .seconds(2))  // Rate limit
}
```

### 2. Error Recovery in Batches

```swift
operation.perRecordSaveBlock = { recordID, result in
    switch result {
    case .success:
        markAsUploaded(recordID)
    case .failure(let error):
        if let ckError = error as? CKError {
            switch ckError.code {
            case .serverRecordChanged:
                handleConflict(recordID, error: ckError)
            case .limitExceeded:
                requeueForSmallerBatch(recordID)
            default:
                requeueForRetry(recordID)
            }
        }
    }
}
```

### 3. Desynchronize Operations

```swift
// Add random delay between operations to avoid server throttling
func executeWithJitter(_ operation: CKDatabaseOperation) async {
    let jitter = Double.random(in: 0...2)
    try? await Task.sleep(for: .seconds(jitter))
    database.add(operation)
}
```

### 4. Cancel Operations

```swift
// Cancel single operation
operation.cancel()

// Cancel all operations
container.cancelAllOperations()

// Check if cancelled
if operation.isCancelled {
    return
}
```

### 5. Dependencies

```swift
let fetchZonesOp = CKFetchRecordZonesOperation.fetchAllRecordZonesOperation()
let fetchRecordsOp = CKFetchRecordZoneChangesOperation(...)

// Fetch records only after zones are fetched
fetchRecordsOp.addDependency(fetchZonesOp)

database.add(fetchZonesOp)
database.add(fetchRecordsOp)
```
