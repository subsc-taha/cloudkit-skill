# CloudKit API Reference

Complete API listing for all CloudKit classes, methods, and properties.

## Table of Contents
1. [Core Objects](#core-objects)
2. [Record Classes](#record-classes)
3. [Zone Classes](#zone-classes)
4. [Query Classes](#query-classes)
5. [Subscription Classes](#subscription-classes)
6. [Notification Classes](#notification-classes)
7. [Sharing Classes](#sharing-classes)
8. [Operation Classes](#operation-classes)
9. [Identity Classes](#identity-classes)
10. [Utility Classes](#utility-classes)

---

## Core Objects

### CKContainer

Entry point to CloudKit.

```swift
// Creation
CKContainer.default()
CKContainer(identifier: "iCloud.com.company.app")

// Properties
container.containerIdentifier        // String?
container.privateCloudDatabase       // CKDatabase
container.publicCloudDatabase        // CKDatabase  
container.sharedCloudDatabase        // CKDatabase

// Account Status
try await container.accountStatus()  // CKAccountStatus

// User Record
try await container.userRecordID()   // CKRecord.ID

// Permissions
try await container.requestApplicationPermission(.userDiscoverability)
try await container.status(forApplicationPermission: .userDiscoverability)

// Operations
container.add(operation)
container.cancelAllOperations()

// Long-lived Operations
container.fetchAllLongLivedOperationIDs { ids, error in }
container.fetchLongLivedOperation(withID: id) { op, error in }
```

### CKDatabase

Storage layer.

```swift
// Properties
database.databaseScope  // .private, .public, .shared

// Records
try await database.save(record)
try await database.record(for: recordID)
try await database.records(for: recordIDs)
try await database.deleteRecord(withID: recordID)

// Queries
try await database.records(matching: query)
try await database.records(matching: query, inZoneWith: zoneID)

// Zones
try await database.save(zone)
try await database.allRecordZones()
try await database.recordZone(for: zoneID)
try await database.deleteRecordZone(withID: zoneID)

// Subscriptions
try await database.save(subscription)
try await database.allSubscriptions()
try await database.subscription(for: subscriptionID)
try await database.deleteSubscription(withID: subscriptionID)

// Operations
database.add(operation)
```

### CKOperationGroup

Group related operations.

```swift
let group = CKOperationGroup()
group.name = "Initial Sync"
group.expectedSendSize = .megabytes
group.expectedReceiveSize = .gigabytes
group.operationGroupID  // UUID

operation.group = group
```

---

## Record Classes

### CKRecord

Single data item.

```swift
// Creation
CKRecord(recordType: "Note")
CKRecord(recordType: "Note", recordID: recordID)
CKRecord(recordType: "Note", zoneID: zoneID)

// Fields (subscript)
record["title"] = "My Title"
record["count"] = 42
record["date"] = Date()
record["data"] = Data()
record["location"] = CLLocation(latitude: 37.7749, longitude: -122.4194)
record["reference"] = CKRecord.Reference(recordID: parentID, action: .none)
record["asset"] = CKAsset(fileURL: url)
record["list"] = ["a", "b", "c"]

let title = record["title"] as? String

// Encrypted Fields
record.encryptedValues["secret"] = "confidential"
let secret = record.encryptedValues["secret"] as? String

// Metadata (read-only)
record.recordID           // CKRecord.ID
record.recordType         // String
record.creationDate       // Date?
record.modificationDate   // Date?
record.creatorUserRecordID      // CKRecord.ID?
record.lastModifiedUserRecordID // CKRecord.ID?
record.recordChangeTag    // String?

// Keys
record.allKeys()          // [String]
record.changedKeys()      // [String]

// Sharing
record.parent             // CKRecord.Reference?
record.share              // CKRecord.Reference?
record.setParent(parentRecord)
record.setParent(parentReference)

// System Fields (for persistence)
record.encodeSystemFields(with: archiver)

// Full-text search
record.allTokens()        // [String]
```

### CKRecord.ID

Unique identifier.

```swift
// Creation
CKRecord.ID(recordName: "unique-id")
CKRecord.ID(recordName: "unique-id", zoneID: zoneID)

// Properties
recordID.recordName       // String
recordID.zoneID           // CKRecordZone.ID
```

### CKRecord.Reference

Relationship between records.

```swift
// Creation
CKRecord.Reference(recordID: parentID, action: .none)
CKRecord.Reference(recordID: parentID, action: .deleteSelf)
CKRecord.Reference(record: parentRecord, action: .none)

// Properties
reference.recordID        // CKRecord.ID
reference.action          // .none or .deleteSelf
```

### CKAsset

Binary data.

```swift
// Creation
CKAsset(fileURL: localFileURL)

// Properties
asset.fileURL             // URL? (temporary, read after fetch)
```

---

## Zone Classes

### CKRecordZone

Logical grouping of records.

```swift
// Creation
CKRecordZone(zoneName: "MyZone")
CKRecordZone(zoneID: zoneID)

// Default Zone
CKRecordZone.default()

// Properties
zone.zoneID               // CKRecordZone.ID
zone.capabilities         // CKRecordZone.Capabilities
zone.share                // CKRecord.Reference? (for zone sharing)

// Capabilities
.fetchChanges             // Supports change tracking
.atomic                   // Supports atomic operations
.sharing                  // Supports sharing
.zoneWideSharing          // Supports zone-level sharing
```

### CKRecordZone.ID

Zone identifier.

```swift
// Creation
CKRecordZone.ID(zoneName: "MyZone", ownerName: CKCurrentUserDefaultName)

// Properties
zoneID.zoneName           // String
zoneID.ownerName          // String

// Constants
CKCurrentUserDefaultName  // Current user
CKRecordZone.ID.defaultZoneName
```

---

## Query Classes

### CKQuery

Query definition.

```swift
// Creation
CKQuery(recordType: "Note", predicate: predicate)

// Properties
query.recordType          // String
query.predicate           // NSPredicate
query.sortDescriptors     // [NSSortDescriptor]?
```

### CKQueryOperation

Execute query.

```swift
// Creation
CKQueryOperation(query: query)
CKQueryOperation(cursor: cursor)

// Configuration
operation.zoneID = zoneID
operation.desiredKeys = ["title", "content"]
operation.resultsLimit = 100

// Callbacks
operation.recordMatchedBlock = { recordID, result in }
operation.queryResultBlock = { result in }  // includes cursor
```

### CKLocationSortDescriptor

Sort by distance.

```swift
let descriptor = CKLocationSortDescriptor(
    key: "location",
    relativeLocation: userLocation
)
query.sortDescriptors = [descriptor]
```

---

## Subscription Classes

### CKSubscription (Base)

```swift
// Properties
subscription.subscriptionID      // String
subscription.subscriptionType    // .query, .recordZone, .database
subscription.notificationInfo    // CKSubscription.NotificationInfo?
```

### CKSubscription.NotificationInfo

```swift
let info = CKSubscription.NotificationInfo()

// Silent push
info.shouldSendContentAvailable = true

// Visible notification
info.alertBody = "New data available"
info.alertLocalizationKey = "ALERT_KEY"
info.alertLocalizationArgs = ["arg1"]
info.title = "Title"
info.titleLocalizationKey = "TITLE_KEY"
info.soundName = "default"
info.shouldBadge = true

// Payload
info.desiredKeys = ["title", "preview"]
info.category = "MESSAGE_CATEGORY"
info.collapseIDKey = "recordID"
```

### CKRecordZoneSubscription

```swift
let sub = CKRecordZoneSubscription(zoneID: zoneID)
let sub = CKRecordZoneSubscription(zoneID: zoneID, subscriptionID: "my-sub")
```

### CKQuerySubscription

```swift
let sub = CKQuerySubscription(
    recordType: "Note",
    predicate: predicate,
    subscriptionID: "notes-sub",
    options: [.firesOnRecordCreation, .firesOnRecordUpdate, .firesOnRecordDeletion]
)
sub.zoneID = zoneID  // Optional: limit to zone
```

### CKDatabaseSubscription

```swift
let sub = CKDatabaseSubscription(subscriptionID: "db-sub")
```

---

## Notification Classes

### CKNotification (Base)

```swift
// Create from push payload
let notification = CKNotification(fromRemoteNotificationDictionary: userInfo)

// Properties
notification.notificationType    // .query, .recordZone, .database, .readNotification
notification.notificationID      // CKNotification.ID?
notification.containerIdentifier // String?
notification.subscriptionID      // String?
notification.isPruned            // Bool
```

### CKQueryNotification

```swift
let queryNotif = notification as! CKQueryNotification

queryNotif.queryNotificationReason  // .recordCreated, .recordUpdated, .recordDeleted
queryNotif.recordID                 // CKRecord.ID?
queryNotif.recordFields             // [String: CKRecordValue]? (from desiredKeys)
queryNotif.databaseScope            // CKDatabase.Scope
```

### CKRecordZoneNotification

```swift
let zoneNotif = notification as! CKRecordZoneNotification

zoneNotif.recordZoneID              // CKRecordZone.ID?
zoneNotif.databaseScope             // CKDatabase.Scope
```

### CKDatabaseNotification

```swift
let dbNotif = notification as! CKDatabaseNotification

dbNotif.databaseScope               // CKDatabase.Scope
```

---

## Sharing Classes

### CKShare

```swift
// Creation (record sharing)
CKShare(rootRecord: record)
CKShare(rootRecord: record, shareID: shareID)

// Creation (zone sharing, iOS 15+)
CKShare(recordZoneID: zoneID)

// Properties
share.recordID              // CKRecord.ID
share.publicPermission      // .none, .readOnly, .readWrite
share.url                   // URL?
share.participants          // [CKShare.Participant]
share.currentUserParticipant // CKShare.Participant?
share.owner                 // CKShare.Participant?

// System Fields
share[CKShare.SystemFieldKey.title] = "My Share"
share[CKShare.SystemFieldKey.thumbnailImageData] = imageData

// Manage Participants
share.addParticipant(participant)
share.removeParticipant(participant)
```

### CKShare.Participant

```swift
participant.userIdentity       // CKUserIdentity
participant.role               // .owner, .privateUser, .publicUser
participant.permission         // .unknown, .none, .readOnly, .readWrite
participant.acceptanceStatus   // .unknown, .pending, .accepted, .removed
```

### CKShare.Metadata

```swift
metadata.containerIdentifier   // String
metadata.share                 // CKShare
metadata.rootRecord            // CKRecord?
metadata.rootRecordID          // CKRecord.ID
metadata.hierarchicalRootRecordID // CKRecord.ID?
metadata.participantPermission // CKShare.ParticipantPermission
metadata.participantRole       // CKShare.ParticipantRole
metadata.participantStatus     // CKShare.ParticipantAcceptanceStatus
metadata.ownerIdentity         // CKUserIdentity
```

### CKAllowedSharingOptions

```swift
let options = CKAllowedSharingOptions(
    allowedParticipantPermissionOptions: [.readOnly, .readWrite],
    allowedParticipantAccessOptions: [.anyoneWithLink, .specifiedRecipientsOnly]
)
```

---

## Operation Classes

### CKOperation (Base)

```swift
operation.configuration.timeoutIntervalForRequest = 30
operation.configuration.timeoutIntervalForResource = 300
operation.configuration.isLongLived = true
operation.qualityOfService = .userInitiated
operation.group = operationGroup

operation.cancel()
operation.isCancelled
```

### Record Operations

```swift
// Modify
CKModifyRecordsOperation(recordsToSave: [...], recordIDsToDelete: [...])
operation.savePolicy = .changedKeys  // .ifServerRecordUnchanged, .allKeys
operation.isAtomic = true
operation.perRecordSaveBlock = { recordID, result in }
operation.perRecordDeleteBlock = { recordID, result in }
operation.modifyRecordsResultBlock = { result in }

// Fetch
CKFetchRecordsOperation(recordIDs: [...])
operation.desiredKeys = ["title"]
operation.perRecordResultBlock = { recordID, result in }
operation.fetchRecordsResultBlock = { result in }
```

### Zone Operations

```swift
// Modify
CKModifyRecordZonesOperation(recordZonesToSave: [...], recordZoneIDsToDelete: [...])
operation.perRecordZoneSaveBlock = { zoneID, result in }
operation.perRecordZoneDeleteBlock = { zoneID, result in }
operation.modifyRecordZonesResultBlock = { result in }

// Fetch
CKFetchRecordZonesOperation(recordZoneIDs: [...])
CKFetchRecordZonesOperation.fetchAllRecordZonesOperation()
operation.perRecordZoneResultBlock = { zoneID, result in }
operation.fetchRecordZonesResultBlock = { result in }

// Fetch Changes
CKFetchRecordZoneChangesOperation(recordZoneIDs: [...], configurationsByRecordZoneID: [...])
operation.recordWasChangedBlock = { recordID, result in }
operation.recordWithIDWasDeletedBlock = { recordID, recordType in }
operation.recordZoneFetchResultBlock = { zoneID, result in }
operation.fetchRecordZoneChangesResultBlock = { result in }

// Database Changes
CKFetchDatabaseChangesOperation(previousServerChangeToken: token)
operation.recordZoneWithIDChangedBlock = { zoneID in }
operation.recordZoneWithIDWasDeletedBlock = { zoneID in }
operation.recordZoneWithIDWasPurgedBlock = { zoneID in }
operation.recordZoneWithIDWasDeletedDueToUserEncryptedDataResetBlock = { zoneID in }
operation.fetchDatabaseChangesResultBlock = { result in }
```

### Subscription Operations

```swift
// Modify
CKModifySubscriptionsOperation(subscriptionsToSave: [...], subscriptionIDsToDelete: [...])
operation.perSubscriptionSaveBlock = { subID, result in }
operation.perSubscriptionDeleteBlock = { subID, result in }
operation.modifySubscriptionsResultBlock = { result in }

// Fetch
CKFetchSubscriptionsOperation(subscriptionIDs: [...])
CKFetchSubscriptionsOperation.fetchAllSubscriptionsOperation()
operation.perSubscriptionResultBlock = { subID, result in }
operation.fetchSubscriptionsResultBlock = { result in }
```

### Share Operations

```swift
// Fetch Metadata
CKFetchShareMetadataOperation(shareURLs: [...])
operation.perShareMetadataResultBlock = { url, result in }
operation.fetchShareMetadataResultBlock = { result in }

// Accept Shares
CKAcceptSharesOperation(shareMetadatas: [...])
operation.perShareResultBlock = { metadata, result in }
operation.acceptSharesResultBlock = { result in }

// Fetch Participants
CKFetchShareParticipantsOperation(userIdentityLookupInfos: [...])
operation.perShareParticipantResultBlock = { lookupInfo, result in }
operation.fetchShareParticipantsResultBlock = { result in }
```

---

## Identity Classes

### CKUserIdentity

```swift
identity.userRecordID          // CKRecord.ID?
identity.lookupInfo            // CKUserIdentity.LookupInfo?
identity.nameComponents        // PersonNameComponents?
identity.hasiCloudAccount      // Bool
identity.contactIdentifiers    // [String]
```

### CKUserIdentity.LookupInfo

```swift
CKUserIdentity.LookupInfo(emailAddress: "user@example.com")
CKUserIdentity.LookupInfo(phoneNumber: "+1234567890")
CKUserIdentity.LookupInfo(userRecordID: recordID)

lookupInfo.emailAddress        // String?
lookupInfo.phoneNumber         // String?
lookupInfo.userRecordID        // CKRecord.ID?
```

### Discovery Operations

```swift
// Discover All
CKDiscoverAllUserIdentitiesOperation()
operation.userIdentityDiscoveredBlock = { identity in }
operation.discoverAllUserIdentitiesResultBlock = { result in }

// Discover Specific
CKDiscoverUserIdentitiesOperation(userIdentityLookupInfos: [...])
operation.userIdentityDiscoveredBlock = { identity, lookupInfo in }
operation.discoverUserIdentitiesResultBlock = { result in }
```

---

## Utility Classes

### CKServerChangeToken

Opaque token for incremental sync.

```swift
// Store/retrieve
let data = try NSKeyedArchiver.archivedData(withRootObject: token, requiringSecureCoding: true)
let token = try NSKeyedUnarchiver.unarchivedObject(ofClass: CKServerChangeToken.self, from: data)
```

### CKSyncEngine (iOS 17+)

See [cksyncengine.md](cksyncengine.md) for complete guide.

```swift
// Creation
CKSyncEngine(configuration: config)

// Properties
engine.database               // CKDatabase
engine.state                  // CKSyncEngine.State

// State Management
engine.state.add(pendingRecordZoneChanges: [...])
engine.state.add(pendingDatabaseChanges: [...])
engine.state.remove(pendingRecordZoneChanges: [...])
engine.state.hasPendingUploads

// Manual Sync
try await engine.fetchChanges()
try await engine.sendChanges()

// Cancel
engine.cancelOperations()
```

### CKError

See [error-handling.md](error-handling.md) for complete guide.

```swift
error.code                    // CKError.Code
error.userInfo                // [String: Any]
error.retryAfterSeconds       // Double?
error.serverRecord            // CKRecord? (for conflicts)
error.partialErrorsByItemID   // [AnyHashable: Error]?
```
