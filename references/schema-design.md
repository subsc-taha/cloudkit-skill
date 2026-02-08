# CloudKit Schema Design

Best practices for designing CloudKit schemas that are maintainable, performant, and future-proof.

## Table of Contents
1. [Record Types](#record-types)
2. [Field Design](#field-design)
3. [Record IDs](#record-ids)
4. [Versioning](#versioning)
5. [References vs Foreign Keys](#references-vs-foreign-keys)
6. [Indexes and Queries](#indexes-and-queries)
7. [Encryption](#encryption)
8. [Migration Strategies](#migration-strategies)

## Record Types

### Naming Conventions

```swift
// Good - PascalCase, descriptive
"Note", "UserProfile", "SharedDocument", "TaskItem"

// Avoid - unclear or too generic
"Data", "Item", "Object", "Record"
```

### One Type per Concept

```swift
// Good - separate types
"Note", "Folder", "Tag"

// Avoid - generic container
"Item" with "itemType" field
```

## Field Design

### Use Optional Fields

CloudKit schemas can only add fields, not remove them. Design defensively:

```swift
// In your Swift model
struct Note {
    let id: String
    let title: String
    let content: String?      // Optional from day 1
    let priority: Int?        // May not exist on old records
    let tags: [String]?       // Added in v1.1
    let location: CLLocation? // Added in v1.2
}

// When reading from CKRecord
let priority = record["priority"] as? Int ?? 0  // Default fallback
```

### Avoid Enums - Use Strings

Enums break when new values are added:

```swift
// ❌ Bad - enum breaks on new values
enum Priority: Int {
    case low = 0
    case medium = 1
    case high = 2
    // v1.1 adds: case critical = 3
    // Old clients crash on unknown value
}

// ✅ Good - string is forward-compatible
record["priority"] = "high"

// Client can handle unknown values
let priority = record["priority"] as? String ?? "medium"
let knownPriorities = ["low", "medium", "high", "critical"]
if !knownPriorities.contains(priority) {
    // Treat as default or show as custom
}
```

### Field Types Best Practices

| Use | For |
|-----|-----|
| `String` | Text, identifiers, enum-like values |
| `Int` | Counts, versions, sequential IDs |
| `Double` | Prices, measurements, calculations |
| `Bool` (as Int) | Flags, toggles |
| `Date` | Timestamps (creation, modification, due dates) |
| `Data` | Small binary (<1MB), serialized objects |
| `CKAsset` | Large files (images, documents) |
| `[String]` | Tags, categories, simple lists |

### What NOT to Store

- Authentication tokens
- Passwords or secrets
- Data that changes every second (use local-only)
- Large data (>1MB) in regular fields (use CKAsset)

## Record IDs

### Use UUIDs

```swift
// Good - globally unique, no collisions
let recordID = CKRecord.ID(recordName: UUID().uuidString)

// Also good - prefixed for readability
let recordID = CKRecord.ID(recordName: "note_\(UUID().uuidString)")
```

### Embed Type in ID for Debugging

```swift
// Helps when debugging across zones
let recordID = CKRecord.ID(recordName: "Note:\(uuid)")

// Or use structured IDs
let recordID = CKRecord.ID(recordName: "v1:Note:\(uuid)")
```

### Match Local IDs

```swift
// Your local model ID should match record name
struct Note: Identifiable {
    let id: String  // Same as CKRecord.ID.recordName
    
    var recordID: CKRecord.ID {
        CKRecord.ID(recordName: id, zoneID: zoneID)
    }
}
```

## Versioning

### Schema Version Field

```swift
record["_schemaVersion"] = 2

// When reading
let version = record["_schemaVersion"] as? Int ?? 1

switch version {
case 1:
    // Old format - migrate
    migrateFromV1(record)
case 2:
    // Current format
    break
case 3...:
    // Future version - app needs update
    showUpdatePrompt()
default:
    break
}
```

### App Version Field

```swift
record["_appVersion"] = "1.2.0"

// Detect records from newer app versions
if let appVersion = record["_appVersion"] as? String,
   appVersion.compare(currentVersion, options: .numeric) == .orderedDescending {
    // Record from future version - handle gracefully or prompt update
}
```

### Handle Unknown Fields

```swift
func hasUnknownFields(_ record: CKRecord) -> Bool {
    let knownKeys = Set(["title", "content", "createdAt", "_schemaVersion"])
    let recordKeys = Set(record.allKeys())
    let unknownKeys = recordKeys.subtracting(knownKeys)
    return !unknownKeys.isEmpty
}
```

## References vs Foreign Keys

### CKRecord.Reference (Apple's Way)

```swift
// Child references parent
let note = CKRecord(recordType: "Note")
note["folder"] = CKRecord.Reference(recordID: folderID, action: .deleteSelf)

// Pros:
// - Automatic cascade delete with .deleteSelf
// - CloudKit validates existence
// - Built-in query support

// Cons:
// - More complex to manage
// - Limited flexibility
```

### String Foreign Keys (Simple Approach)

```swift
// Store parent ID as string
note["folderID"] = folder.recordID.recordName

// Pros:
// - Simpler to manage
// - Works with any ID format
// - Easy to query and join locally

// Cons:
// - No cascade delete
// - No server-side validation
// - Manual joins required
```

### Recommendation

For sync-focused apps (mirroring all data locally), **string foreign keys** are simpler. Use references only when you need cascade delete.

## Indexes and Queries

### Automatic Indexes

CloudKit auto-indexes:
- Record ID (always queryable)
- Reference fields

### Queryable Fields

Enable in CloudKit Dashboard for fields you need to filter/sort:

```
Dashboard → Schema → Record Types → [Your Type] → [Field] → [Queryable]
```

### Encrypted Fields Can't Be Indexed

```swift
// This field CANNOT be used in queries
record.encryptedValues["secretNote"] = "Can't search this"

// This field CAN be queried
record["category"] = "work"
```

### Query Optimization

```swift
// Good - uses indexed field
let predicate = NSPredicate(format: "category == %@", "work")

// Expensive - full table scan if not indexed
let predicate = NSPredicate(format: "content CONTAINS %@", "meeting")
```

## Encryption

### When to Use Encrypted Fields

```swift
// Use encryptedValues for:
record.encryptedValues["content"] = sensitiveText    // Personal notes
record.encryptedValues["apiKey"] = userToken          // Credentials
record.encryptedValues["healthData"] = jsonData       // Medical info

// Use regular fields for:
record["title"] = "Meeting Notes"     // Needs to be searchable
record["category"] = "work"           // Needs indexing
record["createdAt"] = Date()          // Needs sorting
```

### Encryption Trade-offs

| Encrypted | Regular |
|-----------|---------|
| ✅ End-to-end encrypted | ❌ Apple can read |
| ✅ Private to user | ❌ Visible in Dashboard |
| ❌ Not queryable | ✅ Can filter/sort |
| ❌ Not indexable | ✅ Fast lookups |

### Hybrid Approach

```swift
// Index metadata, encrypt content
record["title"] = title                           // Searchable
record["category"] = category                     // Filterable  
record["createdAt"] = Date()                      // Sortable
record.encryptedValues["content"] = sensitiveBody // Private
record.encryptedValues["attachments"] = jsonData  // Private
```

## Migration Strategies

### Add New Fields (Safe)

```swift
// Version 1
record["title"] = title
record["content"] = content

// Version 2 - just add new field
record["tags"] = tags ?? []  // Default for old records
record["priority"] = priority ?? "medium"
```

### Change Field Type (Careful)

```swift
// Version 1: priority as Int
record["priority"] = 1

// Version 2: priority as String - use new field name
record["priorityLevel"] = "high"

// Migration: read both, write new
let priority: String
if let newValue = record["priorityLevel"] as? String {
    priority = newValue
} else if let oldValue = record["priority"] as? Int {
    priority = ["low", "medium", "high"][oldValue]
} else {
    priority = "medium"
}
```

### Deprecate Fields

You can't delete fields from CloudKit schema, but you can stop using them:

```swift
// Old code
record["category"] = category

// New code - just don't write to it anymore
// Keep reading for backward compatibility with old records
let category = record["categoryV2"] as? String 
    ?? record["category"] as? String  // Fallback to old
    ?? "uncategorized"
```

### Full Data Migration

For major changes, migrate records:

```swift
func migrateAllRecords() async throws {
    let query = CKQuery(recordType: "NoteV1", predicate: NSPredicate(value: true))
    let (results, _) = try await database.records(matching: query)
    
    var newRecords: [CKRecord] = []
    var oldIDs: [CKRecord.ID] = []
    
    for (id, result) in results {
        guard case .success(let oldRecord) = result else { continue }
        
        // Create new version
        let newRecord = CKRecord(recordType: "NoteV2")
        newRecord["title"] = oldRecord["title"]
        newRecord["content"] = oldRecord["body"]  // Field renamed
        newRecord["priority"] = priorityString(from: oldRecord["priority"])
        newRecord["_migratedFrom"] = id.recordName
        
        newRecords.append(newRecord)
        oldIDs.append(id)
    }
    
    // Save new, delete old
    let operation = CKModifyRecordsOperation(
        recordsToSave: newRecords,
        recordIDsToDelete: oldIDs
    )
    try await database.add(operation)
}
```
