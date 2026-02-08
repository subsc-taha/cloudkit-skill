# CloudKit Sharing

Complete guide to sharing CloudKit data between iCloud users.

## Table of Contents
1. [Overview](#overview)
2. [Share Types](#share-types)
3. [Creating Shares](#creating-shares)
4. [Sharing UI](#sharing-ui)
5. [Accepting Shares](#accepting-shares)
6. [Managing Participants](#managing-participants)
7. [Shared Database](#shared-database)
8. [Zone Sharing](#zone-sharing)
9. [Permissions](#permissions)
10. [CKSyncEngine with Sharing](#cksyncengine-with-sharing)

## Overview

CloudKit sharing allows users to share records with other iCloud users. Two approaches:

| Approach | Use Case | iOS Version |
|----------|----------|-------------|
| **Record Sharing** | Share individual records | iOS 10+ |
| **Zone Sharing** | Share entire zones | iOS 15+ |

## Share Types

### CKShare

A special record type that defines sharing permissions:

```swift
// CKShare properties
share.publicPermission      // .none, .readOnly, .readWrite
share.currentUserParticipant
share.participants          // All participants
share.owner                 // Share creator
share.recordID              // Share's record ID
share[CKShare.SystemFieldKey.title]         // Display title
share[CKShare.SystemFieldKey.thumbnailImageData]  // Thumbnail
```

### Permission Levels

```swift
public enum CKShare.ParticipantPermission {
    case unknown      // Not determined
    case none         // No access
    case readOnly     // Can view only
    case readWrite    // Can view and edit
}

public enum CKShare.ParticipantAcceptanceStatus {
    case unknown      // Not determined
    case pending      // Invited but not accepted
    case accepted     // Accepted invitation
    case removed      // Was removed
}
```

## Creating Shares

### Basic Share Creation

```swift
// 1. Create the record to share
let note = CKRecord(recordType: "Note", recordID: noteRecordID)
note["title"] = "Shared Document"
note["content"] = "This will be shared"

// 2. Create share for the record
let share = CKShare(rootRecord: note)
share.publicPermission = .none  // Private by default
share[CKShare.SystemFieldKey.title] = "My Shared Note"

// 3. Save both together
let operation = CKModifyRecordsOperation(
    recordsToSave: [note, share],
    recordIDsToDelete: nil
)
operation.modifyRecordsResultBlock = { result in
    switch result {
    case .success:
        print("Share created successfully")
    case .failure(let error):
        print("Failed to create share: \(error)")
    }
}
privateDatabase.add(operation)
```

### Share with Existing Record

```swift
// Fetch existing record
let record = try await database.record(for: recordID)

// Create share
let share = CKShare(rootRecord: record)
share[CKShare.SystemFieldKey.title] = record["title"] as? String ?? "Shared Item"

// Save
try await database.save(share)
```

### Hierarchical Sharing (Parent-Child)

```swift
// Create parent record
let folder = CKRecord(recordType: "Folder")
folder["name"] = "Shared Folder"

// Create share for parent
let share = CKShare(rootRecord: folder)

// Child records automatically included when they reference the parent
let note = CKRecord(recordType: "Note")
note["folder"] = CKRecord.Reference(recordID: folder.recordID, action: .deleteSelf)
note.parent = CKRecord.Reference(recordID: folder.recordID, action: .none)

// Save all
let op = CKModifyRecordsOperation(recordsToSave: [folder, share, note])
database.add(op)
```

## Sharing UI

### UICloudSharingController (UIKit)

```swift
import UIKit
import CloudKit

class ViewController: UIViewController, UICloudSharingControllerDelegate {
    
    func presentShareUI(for record: CKRecord, share: CKShare) {
        let container = CKContainer.default()
        
        let sharingController = UICloudSharingController(share: share, container: container)
        sharingController.delegate = self
        sharingController.availablePermissions = [.allowReadOnly, .allowReadWrite, .allowPrivate]
        
        // For iPad
        sharingController.popoverPresentationController?.sourceView = view
        
        present(sharingController, animated: true)
    }
    
    // MARK: - UICloudSharingControllerDelegate
    
    func cloudSharingController(_ csc: UICloudSharingController, 
                                 failedToSaveShareWithError error: Error) {
        print("Failed to save share: \(error)")
    }
    
    func itemTitle(for csc: UICloudSharingController) -> String? {
        return "My Shared Document"
    }
    
    func itemThumbnailData(for csc: UICloudSharingController) -> Data? {
        return UIImage(named: "thumbnail")?.pngData()
    }
}
```

### Creating Share via UI

```swift
func presentNewShareUI(for record: CKRecord) {
    let container = CKContainer.default()
    
    let sharingController = UICloudSharingController { controller, preparationHandler in
        // Create share when user taps "Share"
        let share = CKShare(rootRecord: record)
        share[CKShare.SystemFieldKey.title] = record["title"] as? String
        
        let operation = CKModifyRecordsOperation(recordsToSave: [record, share])
        operation.modifyRecordsResultBlock = { result in
            switch result {
            case .success:
                preparationHandler(share, container, nil)
            case .failure(let error):
                preparationHandler(nil, nil, error)
            }
        }
        container.privateCloudDatabase.add(operation)
    }
    
    sharingController.delegate = self
    present(sharingController, animated: true)
}
```

### SwiftUI Sharing

```swift
import SwiftUI
import CloudKit

struct ShareButton: View {
    let record: CKRecord
    let share: CKShare
    @State private var showingShareSheet = false
    
    var body: some View {
        Button("Share") {
            showingShareSheet = true
        }
        .sheet(isPresented: $showingShareSheet) {
            CloudSharingView(share: share, container: CKContainer.default())
        }
    }
}

struct CloudSharingView: UIViewControllerRepresentable {
    let share: CKShare
    let container: CKContainer
    
    func makeUIViewController(context: Context) -> UICloudSharingController {
        let controller = UICloudSharingController(share: share, container: container)
        return controller
    }
    
    func updateUIViewController(_ uiViewController: UICloudSharingController, context: Context) {}
}
```

## Accepting Shares

### Handle Share URL

```swift
// In AppDelegate or SceneDelegate
func application(_ application: UIApplication, 
                 userDidAcceptCloudKitShareWith cloudKitShareMetadata: CKShare.Metadata) {
    let container = CKContainer(identifier: cloudKitShareMetadata.containerIdentifier)
    
    let operation = CKAcceptSharesOperation(shareMetadatas: [cloudKitShareMetadata])
    operation.acceptSharesResultBlock = { result in
        switch result {
        case .success:
            print("Share accepted!")
            // Fetch shared records from sharedCloudDatabase
        case .failure(let error):
            print("Failed to accept share: \(error)")
        }
    }
    container.add(operation)
}
```

### SwiftUI Share Handling

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    // Handle share URL
                }
        }
    }
    
    init() {
        // Register for CloudKit shares
        NotificationCenter.default.addObserver(
            forName: NSNotification.Name("CKShareAccepted"),
            object: nil,
            queue: .main
        ) { notification in
            // Handle accepted share
        }
    }
}
```

## Managing Participants

### Add Participant

```swift
// Look up user by email
let lookupInfo = CKUserIdentity.LookupInfo(emailAddress: "friend@example.com")
let operation = CKFetchShareParticipantsOperation(userIdentityLookupInfos: [lookupInfo])

operation.perShareParticipantResultBlock = { lookupInfo, result in
    switch result {
    case .success(let participant):
        participant.permission = .readWrite
        share.addParticipant(participant)
        // Save share to apply changes
    case .failure(let error):
        print("Failed to find user: \(error)")
    }
}
container.add(operation)
```

### Remove Participant

```swift
if let participant = share.participants.first(where: { 
    $0.userIdentity.lookupInfo?.emailAddress == "friend@example.com" 
}) {
    share.removeParticipant(participant)
    try await database.save(share)
}
```

### List Participants

```swift
for participant in share.participants {
    let email = participant.userIdentity.lookupInfo?.emailAddress ?? "Unknown"
    let permission = participant.permission
    let status = participant.acceptanceStatus
    
    print("\(email): \(permission), \(status)")
}
```

## Shared Database

Records shared with you appear in the shared database:

```swift
let sharedDB = container.sharedCloudDatabase

// Fetch all shared record zones
let zones = try await sharedDB.allRecordZones()

for zone in zones {
    print("Shared zone: \(zone.zoneID.zoneName)")
    
    // Query records in shared zone
    let query = CKQuery(recordType: "Note", predicate: NSPredicate(value: true))
    let (results, _) = try await sharedDB.records(matching: query, inZoneWith: zone.zoneID)
    
    for (_, result) in results {
        if case .success(let record) = result {
            print("Shared record: \(record["title"] ?? "")")
        }
    }
}
```

### Fetch Share for Record

```swift
// Get the share that a record belongs to
let operation = CKFetchShareMetadataOperation(shareURLs: [shareURL])
operation.perShareMetadataResultBlock = { url, result in
    switch result {
    case .success(let metadata):
        print("Share owner: \(metadata.ownerIdentity)")
        print("Permission: \(metadata.participantPermission)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
container.add(operation)
```

## Zone Sharing

iOS 15+ allows sharing entire zones:

```swift
// Create zone share
let zoneID = CKRecordZone.ID(zoneName: "SharedZone", ownerName: CKCurrentUserDefaultName)
let share = CKShare(recordZoneID: zoneID)
share[CKShare.SystemFieldKey.title] = "My Shared Zone"

// All records in this zone will be shared
try await database.save(share)
```

### Benefits of Zone Sharing

| Feature | Record Sharing | Zone Sharing |
|---------|----------------|--------------|
| Granularity | Per-record | All records in zone |
| Hierarchy | Manual parent references | Automatic |
| New records | Must explicitly share | Auto-shared |
| Performance | More operations | Fewer operations |

## Permissions

### Public vs Private Sharing

```swift
// Public link (anyone with link can access)
share.publicPermission = .readOnly

// Private (invitation only)
share.publicPermission = .none

// Allow participants to add others
// (No direct API - handled by UICloudSharingController)
```

### Check Current User's Permission

```swift
if let currentParticipant = share.currentUserParticipant {
    switch currentParticipant.permission {
    case .readOnly:
        // Show read-only UI
        editButton.isEnabled = false
    case .readWrite:
        // Allow editing
        editButton.isEnabled = true
    default:
        break
    }
}
```

### Check if User is Owner

```swift
let isOwner = share.currentUserParticipant?.role == .owner

if isOwner {
    // Show share management options
    showManageParticipantsButton()
}
```

## CKSyncEngine with Sharing

CKSyncEngine supports the shared database:

```swift
// Create engine for shared database
let sharedConfig = CKSyncEngine.Configuration(
    database: container.sharedCloudDatabase,
    stateSerialization: loadSharedState(),
    delegate: self
)
let sharedEngine = CKSyncEngine(sharedConfig)

// Handle shared record changes
func handleEvent(_ event: CKSyncEngine.Event, syncEngine: CKSyncEngine) async {
    switch event {
    case .fetchedRecordZoneChanges(let changes):
        for modification in changes.modifications {
            // Check if this is a shared record
            let record = modification.record
            if record.share != nil {
                // This is a shared record
                handleSharedRecord(record)
            }
        }
    default:
        break
    }
}
```

### Separate Engines for Private and Shared

```swift
class SyncManager {
    private var privateEngine: CKSyncEngine!
    private var sharedEngine: CKSyncEngine!
    
    init() {
        // Private database engine
        let privateConfig = CKSyncEngine.Configuration(
            database: container.privateCloudDatabase,
            stateSerialization: loadPrivateState(),
            delegate: self
        )
        privateEngine = CKSyncEngine(privateConfig)
        
        // Shared database engine (records shared with you)
        let sharedConfig = CKSyncEngine.Configuration(
            database: container.sharedCloudDatabase,
            stateSerialization: loadSharedState(),
            delegate: self
        )
        sharedEngine = CKSyncEngine(sharedConfig)
    }
}
```
