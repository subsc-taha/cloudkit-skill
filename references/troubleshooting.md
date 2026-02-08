# CloudKit Troubleshooting Guide

Common issues, debugging techniques, and solutions.

## Table of Contents
1. [Setup Issues](#setup-issues)
2. [Authentication Issues](#authentication-issues)
3. [Sync Issues](#sync-issues)
4. [Crash Issues](#crash-issues)
5. [Performance Issues](#performance-issues)
6. [Debugging Techniques](#debugging-techniques)
7. [CloudKit Dashboard](#cloudkit-dashboard)

## Setup Issues

### "Container Not Found" / badContainer

**Symptoms:** CKError with code `.badContainer`

**Causes & Fixes:**
1. **Container ID mismatch**
   ```swift
   // Wrong
   CKContainer(identifier: "iCloud.myapp")
   // Right - must match entitlements exactly
   CKContainer(identifier: "iCloud.com.yourcompany.appname")
   ```

2. **Missing entitlements**
   - Open target → Signing & Capabilities
   - Add iCloud capability
   - Check CloudKit
   - Select your container

3. **Container not created**
   - Go to CloudKit Dashboard
   - Create container with exact ID
   - Wait a few minutes for propagation

### "Bad Database" Error

**Symptoms:** CKError with code `.badDatabase`

**Causes:**
- Using wrong database type for operation
- Zone operations on public database (not supported)
- Container ID in code doesn't match entitlements

### App Not Receiving Push Notifications

**Check:**
1. Background Modes capability → Remote notifications ✓
2. Push Notifications capability added ✓
3. Running on physical device (not simulator)
4. iCloud signed in on device
5. Subscriptions created successfully

```swift
// Verify subscription exists
let subscriptions = try await database.allSubscriptions()
print("Subscriptions: \(subscriptions.map { $0.subscriptionID })")
```

## Authentication Issues

### "Not Authenticated" Error

**Check iCloud status:**
```swift
let container = CKContainer.default()
let status = try await container.accountStatus()

switch status {
case .available:
    print("iCloud available")
case .noAccount:
    print("No iCloud account - prompt user to sign in")
case .restricted:
    print("iCloud restricted (parental controls/MDM)")
case .couldNotDetermine:
    print("Unknown - check network")
case .temporarilyUnavailable:
    print("Try again later")
@unknown default:
    break
}
```

**User not signed in:**
- Cannot programmatically sign user in
- Show alert directing to Settings → iCloud
- Listen for account changes

### Account Changed Mid-Session

```swift
// Listen for account changes
NotificationCenter.default.addObserver(
    forName: .CKAccountChanged,
    object: nil,
    queue: .main
) { _ in
    Task {
        await self.handleAccountChange()
    }
}

func handleAccountChange() async {
    let status = try? await CKContainer.default().accountStatus()
    if status != .available {
        // Clear local data or disable sync
        clearSyncState()
    }
}
```

## Sync Issues

### Changes Not Syncing

**Debugging steps:**
1. Check network connectivity
2. Verify iCloud account status
3. Check if changes are queued
   ```swift
   print("Pending changes: \(engine.state.pendingRecordZoneChanges.count)")
   ```
4. Check for errors in `handleEvent`
5. Verify state token is being saved

### Duplicate Records

**Causes:**
- Creating records without checking existence
- Not using unique record IDs
- State token not persisted (re-fetching everything)

**Fix:**
```swift
// Always check for existing before inserting
func processIncoming(_ record: CKRecord) {
    let id = record.recordID.recordName
    if let existing = localStore.find(id: id) {
        existing.update(from: record)
    } else {
        localStore.insert(Model(from: record))
    }
}
```

### Data Appears Then Disappears

**Cause:** Conflict resolution choosing wrong version

**Fix:** Review conflict resolution logic
```swift
case .serverRecordChanged:
    // Don't blindly delete - check what you're overwriting
    let serverRecord = error.serverRecord!
    if shouldKeepLocal(local: localRecord, server: serverRecord) {
        // Re-queue with server's change tag
        let merged = copySystemFields(from: serverRecord, to: localRecord)
        engine.state.add(pendingRecordZoneChanges: [.saveRecord(merged.recordID)])
    } else {
        updateLocal(with: serverRecord)
    }
```

### Deleted Data Reappears

**Cause:** Deletion not properly tracked

**Fix:** Track deleted IDs permanently
```swift
// Maintain set of deleted IDs
var deletedIDs: Set<String> = loadDeletedIDs()

func handleIncoming(_ record: CKRecord) {
    let id = record.recordID.recordName
    if deletedIDs.contains(id) {
        // User deleted this - ignore incoming
        return
    }
    // Process normally
}

func deleteRecord(_ id: String) {
    localStore.delete(id)
    deletedIDs.insert(id)
    saveDeletedIDs()
}
```

### changeTokenExpired Error

**Cause:** Token too old (data expired on server)

**Fix:** Do full re-fetch
```swift
case .changeTokenExpired:
    clearCachedToken()
    try await engine.fetchChanges()
```

## Crash Issues

### iOS 26+ Container Initialization Crash (FB18043319)

**Symptom:** `EXC_BREAKPOINT` in CloudKit during container creation

**Cause:** iOS 26 requires network subsystem init on main thread

**Fix:**
```swift
import Network

@MainActor
func preloadCloudKit() {
    if #available(iOS 26.0, *) {
        _ = nw_tls_create_options()
    }
}

// Call early in app launch, before any CloudKit operations
```

### Crash in Background Thread

**Symptom:** Crash with unclear CloudKit stack trace

**Cause:** Thread safety issues with CKRecord

**Fix:** Don't share CKRecords across threads
```swift
// Extract data on receipt, don't pass CKRecord around
struct RecordData {
    let id: String
    let fields: [String: Any]
    let systemData: Data
    
    init(from record: CKRecord) {
        self.id = record.recordID.recordName
        self.fields = // extract fields
        self.systemData = record.systemFieldsData()
    }
}
```

## Performance Issues

### Sync Taking Too Long

**Causes & Fixes:**

1. **Too many records per batch**
   ```swift
   // Limit to 400 records per operation
   let batches = records.chunked(into: 400)
   ```

2. **Large assets**
   - Compress images before upload
   - Use appropriate quality settings

3. **Too many concurrent operations**
   - Use one operation at a time
   - Queue operations serially

### Rate Limiting / 503 Errors

**Signs:** Frequent `serviceUnavailable` errors

**Fixes:**
```swift
// Minimum 2 seconds between operations
var lastOperationTime = Date.distantPast
let minInterval: TimeInterval = 2.0

func executeOperation() async {
    let elapsed = Date().timeIntervalSince(lastOperationTime)
    if elapsed < minInterval {
        try await Task.sleep(for: .seconds(minInterval - elapsed))
    }
    
    // Execute operation
    lastOperationTime = Date()
}
```

### App Slow After Large Sync

**Cause:** Processing too much on main thread

**Fix:**
```swift
// Process in background
func processFetchedChanges(_ changes: ...) async {
    await Task.detached(priority: .userInitiated) {
        // Heavy processing off main thread
        for record in changes.modifications {
            processRecord(record)
        }
    }.value
    
    // Update UI on main thread
    await MainActor.run {
        tableView.reloadData()
    }
}
```

## Debugging Techniques

### Enable CloudKit Logging

```swift
// In AppDelegate or early startup
UserDefaults.standard.set(true, forKey: "com.apple.CloudKit.Debug")
```

### Log All CKSyncEngine Events

```swift
func handleEvent(_ event: CKSyncEngine.Event, syncEngine: CKSyncEngine) async {
    print("☁️ Event: \(type(of: event))")
    
    switch event {
    case .stateUpdate(let update):
        print("☁️ State update - hasPendingUploads: \(update.hasPendingUploads)")
    case .fetchedRecordZoneChanges(let changes):
        print("☁️ Fetched \(changes.modifications.count) modifications, \(changes.deletions.count) deletions")
    // ... log all cases
    }
}
```

### Track Sync Status

```swift
struct SyncStatus: CustomStringConvertible {
    var lastFetchDate: Date?
    var lastSendDate: Date?
    var pendingUploads: Int
    var pendingDeletes: Int
    var isAuthenticated: Bool
    var lastError: Error?
    
    var description: String {
        """
        Sync Status:
        - Authenticated: \(isAuthenticated)
        - Last Fetch: \(lastFetchDate?.description ?? "never")
        - Last Send: \(lastSendDate?.description ?? "never")
        - Pending: \(pendingUploads) uploads, \(pendingDeletes) deletes
        - Error: \(lastError?.localizedDescription ?? "none")
        """
    }
}
```

## CloudKit Dashboard

Access: https://icloud.developer.apple.com

### Useful Actions

1. **View Records**
   - Database → Records → Query
   - See actual data stored

2. **Check Zones**
   - Database → Zones
   - Verify zones exist

3. **View Subscriptions**
   - Database → Subscriptions
   - Confirm push setup

4. **Schema Issues**
   - Schema → Record Types
   - Check field indexes
   - Verify queryable fields

5. **Deploy to Production**
   - Schema → Deploy to Production
   - Required before App Store release

### Development vs Production

| Development | Production |
|-------------|------------|
| Auto-creates schema | Schema locked |
| Resets allowed | No resets |
| Debug data | Real user data |
| Test containers | Live containers |

**Switch environments:**
```swift
// In entitlements file
com.apple.developer.icloud-container-environment = Development
// or
com.apple.developer.icloud-container-environment = Production
```

### Common Dashboard Tasks

**Delete test data:**
1. Database → Records → Query
2. Select records
3. Delete

**Reset schema (Development only):**
1. Schema → Reset Development Environment
2. ⚠️ This deletes ALL development data

**Check quotas:**
1. Usage → View metrics
2. Monitor storage and request limits
