# CloudKit CRUD Operations

Direct CloudKit API operations for when you're not using CKSyncEngine.

## Table of Contents
1. [Database Access](#database-access)
2. [Create Records](#create-records)
3. [Read Records](#read-records)
4. [Update Records](#update-records)
5. [Delete Records](#delete-records)
6. [Batch Operations](#batch-operations)
7. [Queries](#queries)
8. [Assets](#assets)
9. [References](#references)

## Database Access

```swift
import CloudKit

let container = CKContainer.default()
// Or custom container:
// let container = CKContainer(identifier: "iCloud.com.yourcompany.appname")

// Three databases available:
let privateDB = container.privateCloudDatabase   // User's private data
let publicDB = container.publicCloudDatabase     // Shared with all users
let sharedDB = container.sharedCloudDatabase     // Shared records from others
```

## Create Records

### Basic Record

```swift
// Create a new record
let record = CKRecord(recordType: "Note")
record["title"] = "My First Note"
record["content"] = "Hello, CloudKit!"
record["createdAt"] = Date()
record["priority"] = 1
record["isComplete"] = false

// Save to database
let savedRecord = try await privateDB.save(record)
print("Saved with ID: \(savedRecord.recordID)")
```

### Record with Custom ID

```swift
// Create record ID with custom name
let recordID = CKRecord.ID(recordName: UUID().uuidString)
let record = CKRecord(recordType: "Note", recordID: recordID)
```

### Record in Custom Zone

```swift
// Create zone first (private database only)
let zoneID = CKRecordZone.ID(zoneName: "NotesZone", ownerName: CKCurrentUserDefaultName)
let zone = CKRecordZone(zoneID: zoneID)
try await privateDB.save(zone)

// Create record in zone
let recordID = CKRecord.ID(recordName: UUID().uuidString, zoneID: zoneID)
let record = CKRecord(recordType: "Note", recordID: recordID)
```

## Read Records

### Fetch by ID

```swift
let recordID = CKRecord.ID(recordName: "my-record-id")
let record = try await privateDB.record(for: recordID)
let title = record["title"] as? String
```

### Fetch Multiple by IDs

```swift
let recordIDs = [
    CKRecord.ID(recordName: "id1"),
    CKRecord.ID(recordName: "id2"),
    CKRecord.ID(recordName: "id3")
]

let results = try await privateDB.records(for: recordIDs)
for (recordID, result) in results {
    switch result {
    case .success(let record):
        print("Got: \(record["title"] ?? "")")
    case .failure(let error):
        print("Failed to fetch \(recordID): \(error)")
    }
}
```

## Update Records

```swift
// Fetch existing record
let recordID = CKRecord.ID(recordName: "my-record-id")
let record = try await privateDB.record(for: recordID)

// Modify fields
record["title"] = "Updated Title"
record["modifiedAt"] = Date()

// Save changes
let updatedRecord = try await privateDB.save(record)
```

### Encrypted Fields

```swift
// Use encryptedValues for sensitive data
record.encryptedValues["secretNote"] = "Confidential information"
record.encryptedValues["apiKey"] = "sk_live_xxx"

// Reading encrypted values
let secret = record.encryptedValues["secretNote"] as? String
```

## Delete Records

### Delete Single Record

```swift
let recordID = CKRecord.ID(recordName: "my-record-id")
try await privateDB.deleteRecord(withID: recordID)
```

### Delete Zone (Deletes All Records in Zone)

```swift
let zoneID = CKRecordZone.ID(zoneName: "NotesZone", ownerName: CKCurrentUserDefaultName)
try await privateDB.deleteRecordZone(withID: zoneID)
```

## Batch Operations

For multiple records, use `CKModifyRecordsOperation`:

```swift
// Create records
let record1 = CKRecord(recordType: "Note")
record1["title"] = "Note 1"

let record2 = CKRecord(recordType: "Note")
record2["title"] = "Note 2"

// Delete some records
let deleteID = CKRecord.ID(recordName: "old-record")

// Batch operation
let operation = CKModifyRecordsOperation(
    recordsToSave: [record1, record2],
    recordIDsToDelete: [deleteID]
)

// Configure behavior
operation.savePolicy = .changedKeys  // Only send modified fields
operation.isAtomic = true  // All or nothing

// Modern async API
let (savedRecords, deletedIDs) = try await operation.modifyRecords(in: privateDB)
```

### Save Policies

```swift
operation.savePolicy = .ifServerRecordUnchanged  // Default - conflict if changed
operation.savePolicy = .changedKeys              // Only modified keys, safer merging
operation.savePolicy = .allKeys                  // Overwrite everything
```

## Queries

### Basic Query

```swift
let predicate = NSPredicate(format: "title == %@", "My Note")
let query = CKQuery(recordType: "Note", predicate: predicate)
query.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

let (results, cursor) = try await privateDB.records(matching: query)
for (_, result) in results {
    if case .success(let record) = result {
        print(record["title"] ?? "")
    }
}
```

### Query Predicates

```swift
// Equality
NSPredicate(format: "priority == %d", 1)

// Comparison
NSPredicate(format: "priority > %d", 3)

// String matching
NSPredicate(format: "title BEGINSWITH %@", "My")
NSPredicate(format: "title CONTAINS %@", "important")

// Date range
NSPredicate(format: "createdAt > %@", oneWeekAgo)
NSPredicate(format: "createdAt BETWEEN %@", [startDate, endDate])

// Boolean
NSPredicate(format: "isComplete == %@", NSNumber(value: true))

// Compound
NSPredicate(format: "priority > %d AND isComplete == %@", 3, NSNumber(value: false))

// IN list
NSPredicate(format: "category IN %@", ["work", "personal"])

// Location (for CLLocation fields)
NSPredicate(format: "distanceToLocation:fromLocation:(location, %@) < %f", userLocation, 1000)

// All records (no filter)
NSPredicate(value: true)
```

### Pagination with Cursor

```swift
func fetchAllNotes() async throws -> [CKRecord] {
    var allRecords: [CKRecord] = []
    var cursor: CKQueryOperation.Cursor? = nil
    
    repeat {
        let query = CKQuery(recordType: "Note", predicate: NSPredicate(value: true))
        let (results, nextCursor) = try await privateDB.records(matching: query, desiredKeys: nil, resultsLimit: 100, cursor: cursor)
        
        for (_, result) in results {
            if case .success(let record) = result {
                allRecords.append(record)
            }
        }
        
        cursor = nextCursor
    } while cursor != nil
    
    return allRecords
}
```

### Fetch Changes (Zone-based Sync)

```swift
// Only works with custom zones in private database
var changeToken: CKServerChangeToken? = loadCachedToken()

let zoneID = CKRecordZone.ID(zoneName: "NotesZone", ownerName: CKCurrentUserDefaultName)
let config = CKFetchRecordZoneChangesOperation.ZoneConfiguration()
config.previousServerChangeToken = changeToken

let operation = CKFetchRecordZoneChangesOperation(recordZoneIDs: [zoneID], configurationsByRecordZoneID: [zoneID: config])

operation.recordZoneFetchResultBlock = { zoneID, result in
    switch result {
    case .success((let newToken, _, _)):
        self.saveToken(newToken)
    case .failure(let error):
        print("Error: \(error)")
    }
}

privateDB.add(operation)
```

## Assets

### Upload File

```swift
// Create temporary file
let fileURL = FileManager.default.temporaryDirectory.appendingPathComponent("image.jpg")
try imageData.write(to: fileURL)

// Create asset
let asset = CKAsset(fileURL: fileURL)
record["photo"] = asset

try await privateDB.save(record)

// Clean up temp file
try FileManager.default.removeItem(at: fileURL)
```

### Download Asset

```swift
let record = try await privateDB.record(for: recordID)
if let asset = record["photo"] as? CKAsset,
   let fileURL = asset.fileURL {
    let data = try Data(contentsOf: fileURL)
    let image = UIImage(data: data)
}
```

### Multiple Assets

```swift
record["photos"] = [asset1, asset2, asset3] as CKRecordValue
```

## References

### Create Reference

```swift
// Create parent record
let folder = CKRecord(recordType: "Folder")
folder["name"] = "Work"
let savedFolder = try await privateDB.save(folder)

// Create child with reference
let note = CKRecord(recordType: "Note")
note["title"] = "Meeting Notes"
note["folder"] = CKRecord.Reference(recordID: savedFolder.recordID, action: .deleteSelf)
```

### Reference Actions

```swift
// .none - No action on parent deletion
CKRecord.Reference(recordID: parentID, action: .none)

// .deleteSelf - Delete this record when parent is deleted
CKRecord.Reference(recordID: parentID, action: .deleteSelf)
```

### Query by Reference

```swift
let folderRef = CKRecord.Reference(recordID: folderID, action: .none)
let predicate = NSPredicate(format: "folder == %@", folderRef)
let query = CKQuery(recordType: "Note", predicate: predicate)
```

## Supported Field Types

| Swift Type | CKRecord Type |
|------------|---------------|
| `String` | String |
| `Int`, `Double` | Number |
| `Bool` | Number (0/1) |
| `Date` | Date/Time |
| `Data` | Bytes (max 1MB) |
| `CLLocation` | Location |
| `CKAsset` | Asset (files) |
| `CKRecord.Reference` | Reference |
| `[Any]` | List (of above types) |

## Anti-Patterns

### ❌ Using Default Zone for Production

```swift
// BAD: Default zone lacks atomic operations and sharing
let record = CKRecord(recordType: "Note")
privateDB.save(record) { _, _ in }

// GOOD: Use custom zone
let zoneID = CKRecordZone.ID(zoneName: "NotesZone", ownerName: CKCurrentUserDefaultName)
let recordID = CKRecord.ID(recordName: UUID().uuidString, zoneID: zoneID)
let record = CKRecord(recordType: "Note", recordID: recordID)
```

### ❌ Missing Account Status Check

```swift
// BAD: Assumes iCloud is available
func saveUserData() {
    container.privateCloudDatabase.save(record) { _, _ in }
}

// GOOD: Check account status first
container.accountStatus { status, error in
    guard status == .available else {
        // Handle: .noAccount, .restricted, .couldNotDetermine, .temporarilyUnavailable
        return
    }
    self.container.privateCloudDatabase.save(record) { _, _ in }
}
```

### ❌ Not Observing Account Changes

```swift
// BAD: Assumes account persists
class DataManager {
    let container = CKContainer.default()
}

// GOOD: Observe account changes
NotificationCenter.default.addObserver(
    forName: .CKAccountChanged,
    object: nil,
    queue: .main
) { _ in
    // Re-check account status, clear private data cache if user changed
}
```

### ❌ Individual Saves Instead of Batch

```swift
// BAD: Separate network call for each record
for record in records {
    database.save(record) { _, _ in }
}

// GOOD: Single batch operation
let operation = CKModifyRecordsOperation(recordsToSave: records, recordIDsToDelete: nil)
operation.modifyRecordsResultBlock = { result in }
database.add(operation)
```

### ❌ Storing Child Arrays in Parent

```swift
// BAD: Causes conflict resolution nightmares
let parentRecord = CKRecord(recordType: "Album")
parentRecord["photoIDs"] = photoIDs as CKRecordValue

// GOOD: Child references parent
let photoRecord = CKRecord(recordType: "Photo")
let albumRef = CKRecord.Reference(recordID: albumRecord.recordID, action: .deleteSelf)
photoRecord["album"] = albumRef
```

### ❌ String Literals for Keys

```swift
// BAD: Typos won't be caught
record["titel"] = title

// GOOD: Type-safe keys
enum RecordKeys: String {
    case title, createdAt, category
}
record[RecordKeys.title.rawValue] = title
```

### ❌ Exceeding Record Size

```swift
// BAD: May exceed 1MB limit
record["imageData"] = largeImageData as CKRecordValue

// GOOD: Use CKAsset for binary data
let tempURL = FileManager.default.temporaryDirectory.appendingPathComponent("temp.jpg")
try imageData.write(to: tempURL)
record["image"] = CKAsset(fileURL: tempURL)
```

### ❌ UI Updates on Background Thread

```swift
// BAD: CloudKit callbacks are on background thread
database.fetch(withRecordID: recordID) { record, error in
    self.titleLabel.text = record?["title"] as? String  // Crash!
}

// GOOD: Dispatch to main thread
database.fetch(withRecordID: recordID) { record, error in
    DispatchQueue.main.async {
        self.titleLabel.text = record?["title"] as? String
    }
}
```

### ❌ Downloading All Fields

```swift
// BAD: Downloads everything including large assets
let query = CKQuery(recordType: "Photo", predicate: predicate)
database.perform(query, inZoneWith: nil) { records, error in }

// GOOD: Only fetch needed fields
let operation = CKQueryOperation(query: query)
operation.desiredKeys = ["title", "timestamp"]
database.add(operation)
```
