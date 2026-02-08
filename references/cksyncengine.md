# CKSyncEngine Complete Guide

CKSyncEngine (iOS 17+) is Apple's recommended approach for CloudKit sync. It handles operations, subscriptions, push notifications, batching, and retry logic automatically.

## Table of Contents
1. [Architecture](#architecture)
2. [Initialization](#initialization)
3. [State Serialization](#state-serialization)
4. [Handling Events](#handling-events)
5. [Sending Changes](#sending-changes)
6. [Account Changes](#account-changes)
7. [Conflict Resolution](#conflict-resolution)
8. [Zone Management](#zone-management)
9. [Advanced Topics](#advanced-topics)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Your App                                 │
├──────────────────────┬──────────────────────────────────────┤
│    Local Store       │         CKSyncEngineDelegate         │
│  (SQLite/CoreData)   │  handleEvent() / nextRecordBatch()   │
├──────────────────────┴──────────────────────────────────────┤
│                      CKSyncEngine                            │
│  • State management    • Push handling                       │
│  • Automatic retries   • Batching                            │
│  • Conflict detection  • Rate limiting                       │
├─────────────────────────────────────────────────────────────┤
│                      CloudKit Server                         │
└─────────────────────────────────────────────────────────────┘
```

## Initialization

```swift
import CloudKit

actor SyncEngine {
    private var engine: CKSyncEngine!
    private let container = CKContainer(identifier: "iCloud.com.example.app")
    private let database: CKDatabase
    
    init() async {
        database = container.privateCloudDatabase
        
        // Load previously cached state (nil on first launch)
        let cachedState = loadState()
        
        let config = CKSyncEngine.Configuration(
            database: database,
            stateSerialization: cachedState,
            delegate: self
        )
        
        engine = CKSyncEngine(config)
        
        // Note: CKSyncEngine automatically starts syncing
    }
    
    private func loadState() -> CKSyncEngine.State.Serialization? {
        guard let data = UserDefaults.standard.data(forKey: "SyncEngineState") else {
            return nil
        }
        return try? CKSyncEngine.State.Serialization(from: data)
    }
    
    private func saveState(_ state: CKSyncEngine.State.Serialization) {
        if let data = try? state.data() {
            UserDefaults.standard.set(data, forKey: "SyncEngineState")
        }
    }
}
```

## State Serialization

**Critical**: Always save state updates. Missing this breaks sync.

```swift
case .stateUpdate(let update):
    // Save immediately - this is your checkpoint
    saveState(update.stateSerialization)
```

The state contains:
- Server change tokens (where you are in the sync timeline)
- Pending changes queue (survives app restarts)
- Zone configurations

## Handling Events

The `handleEvent` delegate method is where you react to sync events:

```swift
extension SyncEngine: CKSyncEngineDelegate {
    func handleEvent(_ event: CKSyncEngine.Event, syncEngine: CKSyncEngine) async {
        switch event {
        
        // ALWAYS handle - save your sync checkpoint
        case .stateUpdate(let update):
            saveState(update.stateSerialization)
        
        // User signed in/out/switched accounts
        case .accountChange(let change):
            await handleAccountChange(change)
        
        // Records came from server - update local store
        case .fetchedRecordZoneChanges(let changes):
            await processFetchedChanges(changes)
        
        // Your uploads finished - check for failures
        case .sentRecordZoneChanges(let sent):
            await processSentChanges(sent)
        
        // Database-level changes (zone deletions, etc.)
        case .fetchedDatabaseChanges(let changes):
            await processDatabaseChanges(changes)
        
        // Informational events (can ignore in simple apps)
        case .willFetchChanges, .didFetchChanges,
             .willSendChanges, .didSendChanges,
             .willFetchRecordZoneChanges, .didFetchRecordZoneChanges,
             .sentDatabaseChanges:
            break
        
        @unknown default:
            break
        }
    }
}
```

### Processing Fetched Changes

```swift
func processFetchedChanges(_ changes: CKSyncEngine.Event.FetchedRecordZoneChanges) async {
    // Handle new/modified records
    for modification in changes.modifications {
        let record = modification.record
        let recordID = record.recordID.recordName
        
        if let existingItem = localStore.find(id: recordID) {
            // Update existing
            existingItem.update(from: record)
        } else {
            // Create new
            let newItem = MyModel(from: record)
            localStore.insert(newItem)
        }
    }
    
    // Handle deletions
    for deletion in changes.deletions {
        let recordID = deletion.recordID.recordName
        localStore.delete(id: recordID)
    }
    
    // Persist changes
    localStore.save()
}
```

### Processing Sent Changes

```swift
func processSentChanges(_ sent: CKSyncEngine.Event.SentRecordZoneChanges) async {
    // Update local records with server metadata
    for saved in sent.savedRecords {
        if let item = localStore.find(id: saved.recordID.recordName) {
            item.updateCloudKitMetadata(from: saved)
        }
    }
    
    // Handle failures
    for failure in sent.failedRecordSaves {
        let recordID = failure.record.recordID
        let error = failure.error
        
        switch error.code {
        case .serverRecordChanged:
            // Conflict - resolve and retry
            if let serverRecord = error.serverRecord {
                resolveConflict(local: failure.record, server: serverRecord)
            }
            
        case .zoneNotFound:
            // Zone doesn't exist - create it
            engine.state.add(pendingDatabaseChanges: [.saveZone(zoneID)])
            // Re-queue the record
            engine.state.add(pendingRecordZoneChanges: [.saveRecord(recordID)])
            
        case .unknownItem:
            // Record was deleted server-side
            localStore.delete(id: recordID.recordName)
            
        default:
            print("Failed to save \(recordID): \(error)")
        }
    }
    
    // Confirm deletions
    for deletedID in sent.deletedRecordIDs {
        localStore.confirmDeletion(id: deletedID.recordName)
    }
}
```

## Sending Changes

Queue changes to sync:

```swift
// Queue a save (create or update)
func save(_ item: MyModel) {
    localStore.save(item)
    engine.state.add(pendingRecordZoneChanges: [.saveRecord(item.recordID)])
}

// Queue a deletion
func delete(_ item: MyModel) {
    localStore.delete(item)
    engine.state.add(pendingRecordZoneChanges: [.deleteRecord(item.recordID)])
}
```

Provide records when sync engine requests them:

```swift
func nextRecordZoneChangeBatch(
    _ context: CKSyncEngine.SendChangesContext,
    syncEngine: CKSyncEngine
) async -> CKSyncEngine.RecordZoneChangeBatch? {
    
    let scope = context.options.scope
    let pendingChanges = syncEngine.state.pendingRecordZoneChanges.filter {
        scope.contains($0)
    }
    
    return await CKSyncEngine.RecordZoneChangeBatch(pendingChanges: pendingChanges) { recordID in
        // Look up local item and return its CKRecord
        guard let item = localStore.find(id: recordID.recordName) else {
            // Item was deleted locally - remove from queue
            syncEngine.state.remove(pendingRecordZoneChanges: [.saveRecord(recordID)])
            return nil
        }
        
        // IMPORTANT: Update the CKRecord with current values before returning
        return item.toCKRecord()
    }
}
```

## Account Changes

```swift
func handleAccountChange(_ change: CKSyncEngine.Event.AccountChange) async {
    switch change.changeType {
    case .signedIn:
        // User signed into iCloud
        // Create zone if needed
        engine.state.add(pendingDatabaseChanges: [.saveZone(myZoneID)])
        
    case .signedOut:
        // User signed out - optionally delete local data
        // Re-initialize engine without state
        await reinitializeEngine(clearState: true)
        
    case .switchedAccounts:
        // Different user - must clear local data and reset
        localStore.deleteAll()
        await reinitializeEngine(clearState: true)
        
    @unknown default:
        break
    }
}
```

## Conflict Resolution

When a record conflict occurs:

```swift
case .serverRecordChanged:
    guard let serverRecord = error.serverRecord else { return }
    
    // Option 1: Server wins (safest)
    updateLocalItem(from: serverRecord)
    
    // Option 2: Merge and re-upload
    let mergedRecord = mergeRecords(local: failure.record, server: serverRecord)
    // The merged record now has server's change tag - safe to upload
    engine.state.add(pendingRecordZoneChanges: [.saveRecord(mergedRecord.recordID)])
```

## Zone Management

Create zones when account signs in:

```swift
let zoneID = CKRecordZone.ID(zoneName: "MyAppZone", ownerName: CKCurrentUserDefaultName)

// Queue zone creation
engine.state.add(pendingDatabaseChanges: [.saveZone(zoneID)])
```

Handle zone deletion:

```swift
case .fetchedDatabaseChanges(let changes):
    for deletion in changes.deletions {
        switch deletion.reason {
        case .deleted:
            // Zone was programmatically deleted
            localStore.deleteZoneData(deletion.zoneID)
            
        case .purged:
            // User cleared iCloud data in Settings
            // Delete local data too (user explicitly wanted this)
            localStore.deleteAll()
            clearSyncState()
            
        case .encryptedDataReset:
            // Account recovery - re-upload everything
            clearSyncState()
            reuploadAllData()
            
        @unknown default:
            break
        }
    }
```

## Advanced Topics

### CKRecord Metadata Preservation

Store record metadata locally to avoid conflicts:

```swift
extension CKRecord {
    func systemFieldsData() -> Data {
        let archiver = NSKeyedArchiver(requiringSecureCoding: true)
        encodeSystemFields(with: archiver)
        archiver.finishEncoding()
        return archiver.encodedData
    }
}

// When saving locally
item.cloudKitData = record.systemFieldsData()

// When creating CKRecord for upload
func toCKRecord() -> CKRecord {
    let record: CKRecord
    if let data = cloudKitData {
        let unarchiver = try! NSKeyedUnarchiver(forReadingFrom: data)
        unarchiver.requiresSecureCoding = true
        record = CKRecord(coder: unarchiver)!
    } else {
        record = CKRecord(recordType: "MyType", recordID: myRecordID)
    }
    record["field1"] = value1
    record["field2"] = value2
    return record
}
```

### Multiple CKSyncEngine Instances

- ✅ One engine per database (private + shared = 2 engines OK)
- ❌ Multiple engines for same database (causes conflicts)
- Use zones to separate data within one database

### Force Sync

```swift
// Pull changes from server now
try await engine.fetchChanges()

// Push pending changes now
try await engine.sendChanges()
```

### Don't Call Sync Methods Inside Delegate

Per Apple docs, don't call `fetchChanges()` or `sendChanges()` from within `handleEvent()` — causes infinite loops.

### Batch Sizes

- Uploads: Batched automatically, max ~1MB per batch
- Downloads: Can be much larger (100MB+)
