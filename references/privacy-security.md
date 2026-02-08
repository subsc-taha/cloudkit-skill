# CloudKit Privacy & Security

Data encryption, access controls, and privacy compliance.

## Table of Contents
1. [Encryption](#encryption)
2. [Access Controls](#access-controls)
3. [Web Auth Tokens](#web-auth-tokens)
4. [Data Deletion](#data-deletion)
5. [Container Identification](#container-identification)
6. [Privacy Best Practices](#privacy-best-practices)
7. [GDPR & Compliance](#gdpr--compliance)

## Encryption

### Encrypted Fields

Store sensitive data with end-to-end encryption:

```swift
let record = CKRecord(recordType: "Note")

// Standard field (Apple can access)
record["title"] = "Meeting Notes"

// Encrypted field (end-to-end encrypted)
record.encryptedValues["content"] = "Confidential meeting details..."
record.encryptedValues["attachmentData"] = sensitiveData
```

### Encryption Properties

| Property | Standard Fields | Encrypted Fields |
|----------|----------------|------------------|
| Visibility | Apple can access | Only user can decrypt |
| Queryable | Yes | No |
| Indexable | Yes | No |
| Dashboard | Visible | Hidden |
| Key management | Apple | User's iCloud Keychain |

### What to Encrypt

```swift
// ENCRYPT - sensitive user data
record.encryptedValues["healthData"] = ...
record.encryptedValues["financialInfo"] = ...
record.encryptedValues["personalNotes"] = ...
record.encryptedValues["passwords"] = ...
record.encryptedValues["location"] = ...

// DON'T ENCRYPT - needs querying/indexing
record["category"] = "personal"     // For filtering
record["createdAt"] = Date()        // For sorting
record["searchKeywords"] = [...]    // For search
```

### Encrypted Data Reset

Users can reset their encrypted data during account recovery:

```swift
// Handle in CKSyncEngine
case .fetchedDatabaseChanges(let changes):
    for deletion in changes.deletions {
        if deletion.reason == .encryptedDataReset {
            // User had to reset encryption during account recovery
            // Re-upload all data to minimize loss
            clearLocalSyncState()
            reuploadAllRecords()
        }
    }

// Or check error key
if let userInfo = error.userInfo,
   userInfo[CKErrorUserDidResetEncryptedDataKey] as? Bool == true {
    // Handle encrypted data reset
}
```

## Access Controls

### Database Access Levels

| Database | Who Can Access | Use For |
|----------|---------------|---------|
| Private | Record owner only | Personal data |
| Public | All users | Shared content, leaderboards |
| Shared | Owner + invited participants | Collaboration |

### Record-Level Permissions

For shared records, control access per-participant:

```swift
let share = CKShare(rootRecord: record)

// Public permission (anyone with link)
share.publicPermission = .none       // Invitation only
share.publicPermission = .readOnly   // Anyone can view
share.publicPermission = .readWrite  // Anyone can edit

// Per-participant permission
for participant in share.participants {
    switch participant.permission {
    case .readOnly:
        // Can view, not edit
    case .readWrite:
        // Full access
    case .none, .unknown:
        // No access
    @unknown default:
        break
    }
}
```

### Changing Participant Access

```swift
// Change permission
if let participant = share.participants.first(where: { 
    $0.userIdentity.lookupInfo?.emailAddress == "user@example.com" 
}) {
    participant.permission = .readOnly  // Downgrade to read-only
    try await database.save(share)
}

// Remove participant
share.removeParticipant(participant)
try await database.save(share)
```

## Web Auth Tokens

For web-based CloudKit access:

```swift
let operation = CKFetchWebAuthTokenOperation(apiToken: "your-api-token")

operation.fetchWebAuthTokenResultBlock = { result in
    switch result {
    case .success(let webToken):
        // Use token for CloudKit JS or REST API
        print("Web token: \(webToken)")
    case .failure(let error):
        print("Failed to fetch token: \(error)")
    }
}

CKContainer.default().add(operation)
```

### CloudKit JS Integration

```javascript
// Use token in web app
CloudKit.configure({
    containers: [{
        containerIdentifier: 'iCloud.com.yourcompany.app',
        apiToken: 'your-api-token',
        webToken: webTokenFromSwift  // From CKFetchWebAuthTokenOperation
    }]
});
```

## Data Deletion

### Responding to Delete Requests

Users can request data deletion via Apple. Handle `CKAccountChanged` notification:

```swift
NotificationCenter.default.addObserver(
    forName: .CKAccountChanged,
    object: nil,
    queue: .main
) { _ in
    Task {
        await handleAccountChange()
    }
}

func handleAccountChange() async {
    let status = try? await CKContainer.default().accountStatus()
    
    if status == .noAccount {
        // User signed out or deleted account
        // Delete local cached data
        deleteAllLocalData()
    }
}
```

### Programmatic Deletion

```swift
// Delete all user data
func deleteAllUserData() async throws {
    let container = CKContainer.default()
    let privateDB = container.privateCloudDatabase
    
    // Delete all zones (deletes all records)
    let zones = try await privateDB.allRecordZones()
    for zone in zones where zone.zoneID.zoneName != CKRecordZone.default().zoneID.zoneName {
        try await privateDB.deleteRecordZone(withID: zone.zoneID)
    }
    
    // Delete subscriptions
    let subscriptions = try await privateDB.allSubscriptions()
    for subscription in subscriptions {
        try await privateDB.deleteSubscription(withID: subscription.subscriptionID)
    }
}
```

### Export User Data

Provide data export for privacy compliance:

```swift
func exportUserData() async throws -> Data {
    let container = CKContainer.default()
    let privateDB = container.privateCloudDatabase
    
    var exportData: [[String: Any]] = []
    
    // Fetch all zones
    let zones = try await privateDB.allRecordZones()
    
    for zone in zones {
        // Query all records in zone
        let query = CKQuery(recordType: "YourRecordType", predicate: NSPredicate(value: true))
        let (results, _) = try await privateDB.records(matching: query, inZoneWith: zone.zoneID)
        
        for (_, result) in results {
            if case .success(let record) = result {
                var recordData: [String: Any] = [:]
                for key in record.allKeys() {
                    recordData[key] = record[key]
                }
                exportData.append(recordData)
            }
        }
    }
    
    return try JSONSerialization.data(withJSONObject: exportData, options: .prettyPrinted)
}
```

## Container Identification

### Get Container Info

```swift
let container = CKContainer.default()

// Container identifier
let containerID = container.containerIdentifier
print("Container: \(containerID ?? "default")")

// Check capabilities
container.accountStatus { status, error in
    // ...
}
```

### Multiple Containers

```swift
// Access specific container
let customContainer = CKContainer(identifier: "iCloud.com.company.app2")

// Each container has separate databases
let privateDB1 = CKContainer.default().privateCloudDatabase
let privateDB2 = customContainer.privateCloudDatabase
```

### Container in Entitlements

```xml
<!-- In .entitlements file -->
<key>com.apple.developer.icloud-container-identifiers</key>
<array>
    <string>iCloud.com.yourcompany.yourapp</string>
</array>
<key>com.apple.developer.icloud-services</key>
<array>
    <string>CloudKit</string>
</array>
```

## Privacy Best Practices

### 1. Minimize Data Collection

```swift
// Only store what you need
record["email"] = email  // Only if needed for functionality
// Don't store: IP addresses, device IDs, etc.
```

### 2. Use Encrypted Fields

```swift
// Always encrypt PII
record.encryptedValues["fullName"] = user.fullName
record.encryptedValues["dateOfBirth"] = user.dob
record.encryptedValues["address"] = user.address
```

### 3. Provide Transparency

```swift
// Show what data you store
func showDataStoredAlert() {
    let alert = UIAlertController(
        title: "Data Stored in iCloud",
        message: "We store your notes, preferences, and sync data in your private iCloud container. Only you can access this data.",
        preferredStyle: .alert
    )
    // ...
}
```

### 4. Support Data Portability

```swift
// Offer export in standard formats
func exportToJSON() async throws -> URL { ... }
func exportToCSV() async throws -> URL { ... }
```

### 5. Handle Account Changes

```swift
// Always respond to sign-out
func handleSignOut() {
    // Clear local cache
    clearLocalData()
    
    // Stop sync
    syncEngine.cancel()
    
    // Clear UI
    resetToSignedOutState()
}
```

## GDPR & Compliance

### Right to Access

```swift
// Implement data export
func handleDataAccessRequest() async throws {
    let data = try await exportUserData()
    // Provide to user
}
```

### Right to Erasure

```swift
// Implement complete deletion
func handleErasureRequest() async throws {
    try await deleteAllUserData()
    clearLocalData()
}
```

### Right to Portability

```swift
// Standard format export
func handlePortabilityRequest() async throws -> Data {
    return try await exportUserData()  // JSON format
}
```

### Data Retention

```swift
// Don't keep data longer than needed
func cleanupOldData() async throws {
    let thirtyDaysAgo = Calendar.current.date(byAdding: .day, value: -30, to: Date())!
    
    let predicate = NSPredicate(format: "deletedAt < %@", thirtyDaysAgo as NSDate)
    let query = CKQuery(recordType: "DeletedItem", predicate: predicate)
    
    let (results, _) = try await database.records(matching: query)
    let idsToDelete = results.map { $0.key }
    
    let operation = CKModifyRecordsOperation(recordsToSave: nil, recordIDsToDelete: idsToDelete)
    database.add(operation)
}
```

### Privacy Policy Requirements

Document in your privacy policy:
- What data is stored in CloudKit
- How data is encrypted
- How users can access/delete their data
- Data retention periods
- Third-party access (Apple infrastructure)
