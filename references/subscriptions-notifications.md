# CloudKit Subscriptions & Notifications

Complete guide to push notifications and real-time sync triggers.

## Table of Contents
1. [Overview](#overview)
2. [Subscription Types](#subscription-types)
3. [Creating Subscriptions](#creating-subscriptions)
4. [Notification Info](#notification-info)
5. [Receiving Notifications](#receiving-notifications)
6. [Handling Notifications](#handling-notifications)
7. [Subscription Management](#subscription-management)
8. [Best Practices](#best-practices)

## Overview

Subscriptions let CloudKit notify your app when data changes:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Device A  │────▶│   CloudKit  │────▶│   Device B  │
│  (changes)  │     │   Server    │     │   (push)    │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Device C  │
                    │   (push)    │
                    └─────────────┘
```

## Subscription Types

| Type | Triggers On | Database | Use Case |
|------|-------------|----------|----------|
| `CKRecordZoneSubscription` | Any change in zone | Private/Shared | Sync apps |
| `CKQuerySubscription` | Records matching query | All | Alerts, filters |
| `CKDatabaseSubscription` | Zone create/delete | Private/Shared | Database sync |

## Creating Subscriptions

### Zone Subscription (Most Common for Sync)

```swift
let zoneID = CKRecordZone.ID(zoneName: "MyZone", ownerName: CKCurrentUserDefaultName)

let subscription = CKRecordZoneSubscription(
    zoneID: zoneID,
    subscriptionID: "zone-changes-\(zoneID.zoneName)"
)

// Configure notification
let notificationInfo = CKSubscription.NotificationInfo()
notificationInfo.shouldSendContentAvailable = true  // Silent push for background fetch

subscription.notificationInfo = notificationInfo

// Save subscription
try await privateDatabase.save(subscription)
```

### Query Subscription

```swift
// Trigger on specific record conditions
let predicate = NSPredicate(format: "priority == %@ AND isComplete == %@", "high", NSNumber(value: false))

let subscription = CKQuerySubscription(
    recordType: "Task",
    predicate: predicate,
    subscriptionID: "high-priority-incomplete",
    options: [.firesOnRecordCreation, .firesOnRecordUpdate, .firesOnRecordDeletion]
)

// Visible notification
let notificationInfo = CKSubscription.NotificationInfo()
notificationInfo.alertBody = "New high-priority task!"
notificationInfo.soundName = "default"
notificationInfo.shouldBadge = true

subscription.notificationInfo = notificationInfo
try await publicDatabase.save(subscription)
```

### Query Subscription Options

```swift
// When to fire
let options: CKQuerySubscription.Options = [
    .firesOnRecordCreation,    // New record matches
    .firesOnRecordUpdate,      // Updated record matches
    .firesOnRecordDeletion,    // Deleted record matched
    .firesOnce                 // Fire only once, then delete subscription
]

let subscription = CKQuerySubscription(
    recordType: "Message",
    predicate: predicate,
    subscriptionID: "new-messages",
    options: options
)
```

### Database Subscription

```swift
// Notify when zones are created/deleted
let subscription = CKDatabaseSubscription(subscriptionID: "database-changes")

let notificationInfo = CKSubscription.NotificationInfo()
notificationInfo.shouldSendContentAvailable = true
subscription.notificationInfo = notificationInfo

try await privateDatabase.save(subscription)
```

## Notification Info

Configure how notifications appear:

```swift
let notificationInfo = CKSubscription.NotificationInfo()

// Silent push (background sync)
notificationInfo.shouldSendContentAvailable = true

// Visible alert
notificationInfo.alertBody = "You have new data!"
notificationInfo.alertLocalizationKey = "NEW_DATA_ALERT"  // Localized
notificationInfo.alertLocalizationArgs = ["arg1", "arg2"]

// Title (iOS 10+)
notificationInfo.title = "CloudKit Update"
notificationInfo.titleLocalizationKey = "ALERT_TITLE"

// Sound
notificationInfo.soundName = "default"  // System sound
notificationInfo.soundName = "custom.caf"  // Custom sound

// Badge
notificationInfo.shouldBadge = true

// Category (for actionable notifications)
notificationInfo.category = "MESSAGE_CATEGORY"

// Include record fields in payload
notificationInfo.desiredKeys = ["title", "preview"]

// Collapse similar notifications
notificationInfo.collapseIDKey = "recordID"
```

## Receiving Notifications

### App Delegate Setup

```swift
import UIKit
import CloudKit
import UserNotifications

@main
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    
    func application(_ application: UIApplication, 
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        // Request notification permission
        UNUserNotificationCenter.current().delegate = self
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
            if granted {
                DispatchQueue.main.async {
                    application.registerForRemoteNotifications()
                }
            }
        }
        
        return true
    }
    
    // Remote notification registration
    func application(_ application: UIApplication, 
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        // CloudKit handles this automatically - no action needed
    }
    
    func application(_ application: UIApplication, 
                     didFailToRegisterForRemoteNotificationsWithError error: Error) {
        print("Failed to register for remote notifications: \(error)")
    }
    
    // Receive notification while app is running
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
        
        // Check if it's a CloudKit notification
        if let notification = CKNotification(fromRemoteNotificationDictionary: userInfo) {
            handleCloudKitNotification(notification) {
                completionHandler(.newData)
            }
        } else {
            completionHandler(.noData)
        }
    }
}
```

### SwiftUI App Setup

```swift
import SwiftUI
import CloudKit

@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

class AppDelegate: NSObject, UIApplicationDelegate {
    func application(_ application: UIApplication, 
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        
        application.registerForRemoteNotifications()
        return true
    }
    
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
        
        if let notification = CKNotification(fromRemoteNotificationDictionary: userInfo) {
            Task {
                await SyncManager.shared.handleNotification(notification)
                completionHandler(.newData)
            }
        } else {
            completionHandler(.noData)
        }
    }
}
```

## Handling Notifications

### Process CKNotification

```swift
func handleCloudKitNotification(_ notification: CKNotification, completion: @escaping () -> Void) {
    switch notification.notificationType {
    case .query:
        handleQueryNotification(notification as! CKQueryNotification, completion: completion)
        
    case .recordZone:
        handleRecordZoneNotification(notification as! CKRecordZoneNotification, completion: completion)
        
    case .database:
        handleDatabaseNotification(notification as! CKDatabaseNotification, completion: completion)
        
    case .readNotification:
        // User read a shared record
        completion()
        
    @unknown default:
        completion()
    }
}
```

### Query Notification

```swift
func handleQueryNotification(_ notification: CKQueryNotification, completion: @escaping () -> Void) {
    guard let recordID = notification.recordID else {
        completion()
        return
    }
    
    switch notification.queryNotificationReason {
    case .recordCreated:
        // Fetch new record
        fetchRecord(recordID, completion: completion)
        
    case .recordUpdated:
        // Fetch updated record
        fetchRecord(recordID, completion: completion)
        
    case .recordDeleted:
        // Remove from local store
        deleteLocalRecord(recordID)
        completion()
        
    @unknown default:
        completion()
    }
}

func fetchRecord(_ recordID: CKRecord.ID, completion: @escaping () -> Void) {
    Task {
        do {
            let record = try await database.record(for: recordID)
            saveLocally(record)
        } catch {
            print("Failed to fetch record: \(error)")
        }
        completion()
    }
}
```

### Record Zone Notification

```swift
func handleRecordZoneNotification(_ notification: CKRecordZoneNotification, 
                                   completion: @escaping () -> Void) {
    
    guard let zoneID = notification.recordZoneID else {
        completion()
        return
    }
    
    // Fetch all changes in the zone
    fetchZoneChanges(zoneID: zoneID, completion: completion)
}

func fetchZoneChanges(zoneID: CKRecordZone.ID, completion: @escaping () -> Void) {
    let token = loadChangeToken(for: zoneID)
    
    var config = CKFetchRecordZoneChangesOperation.ZoneConfiguration()
    config.previousServerChangeToken = token
    
    let operation = CKFetchRecordZoneChangesOperation(
        recordZoneIDs: [zoneID],
        configurationsByRecordZoneID: [zoneID: config]
    )
    
    operation.recordWasChangedBlock = { recordID, result in
        if case .success(let record) = result {
            saveLocally(record)
        }
    }
    
    operation.recordWithIDWasDeletedBlock = { recordID, _ in
        deleteLocally(recordID)
    }
    
    operation.recordZoneFetchResultBlock = { zoneID, result in
        if case .success((let newToken, _, _)) = result {
            saveChangeToken(newToken, for: zoneID)
        }
    }
    
    operation.fetchRecordZoneChangesResultBlock = { _ in
        completion()
    }
    
    database.add(operation)
}
```

### Database Notification

```swift
func handleDatabaseNotification(_ notification: CKDatabaseNotification,
                                 completion: @escaping () -> Void) {
    
    // Fetch database-level changes (zone additions/deletions)
    let operation = CKFetchDatabaseChangesOperation(
        previousServerChangeToken: loadDatabaseChangeToken()
    )
    
    operation.recordZoneWithIDChangedBlock = { zoneID in
        // Zone has changes - queue for fetching
        self.zonesToFetch.append(zoneID)
    }
    
    operation.recordZoneWithIDWasDeletedBlock = { zoneID in
        // Zone was deleted
        self.deleteLocalZone(zoneID)
    }
    
    operation.fetchDatabaseChangesResultBlock = { result in
        if case .success((let newToken, _)) = result {
            self.saveDatabaseChangeToken(newToken)
            
            // Now fetch changes from modified zones
            self.fetchChangesForZones(self.zonesToFetch) {
                completion()
            }
        } else {
            completion()
        }
    }
    
    database.add(operation)
}
```

## Subscription Management

### List All Subscriptions

```swift
let subscriptions = try await database.allSubscriptions()

for subscription in subscriptions {
    print("ID: \(subscription.subscriptionID)")
    print("Type: \(type(of: subscription))")
    
    if let zoneSubscription = subscription as? CKRecordZoneSubscription {
        print("Zone: \(zoneSubscription.zoneID)")
    }
}
```

### Delete Subscription

```swift
// Delete by ID
try await database.deleteSubscription(withID: "my-subscription-id")

// Delete multiple
let operation = CKModifySubscriptionsOperation(
    subscriptionsToSave: nil,
    subscriptionIDsToDelete: ["sub1", "sub2", "sub3"]
)
database.add(operation)
```

### Update Subscription

```swift
// Fetch existing
let subscription = try await database.subscription(for: "my-subscription-id")

// Modify and re-save
if var querySubscription = subscription as? CKQuerySubscription {
    querySubscription.notificationInfo?.alertBody = "Updated alert text"
    try await database.save(querySubscription)
}
```

### Check Subscription Exists

```swift
func ensureSubscriptionExists(id: String, create: () -> CKSubscription) async throws {
    do {
        _ = try await database.subscription(for: id)
        print("Subscription already exists")
    } catch let error as CKError where error.code == .unknownItem {
        // Doesn't exist - create it
        let subscription = create()
        try await database.save(subscription)
        print("Subscription created")
    }
}
```

## Best Practices

### 1. Use Silent Pushes for Sync

```swift
// For background sync, always use silent pushes
notificationInfo.shouldSendContentAvailable = true

// Don't set alertBody for silent pushes
// notificationInfo.alertBody = nil  // Default
```

### 2. Deduplicate Subscription Creation

```swift
// Create unique, deterministic subscription IDs
let subscriptionID = "zone-\(zoneID.zoneName)-\(zoneID.ownerName)"

// Check before creating
if !existingSubscriptionIDs.contains(subscriptionID) {
    try await createSubscription(id: subscriptionID)
}
```

### 3. Handle Missing Notifications

Notifications can be dropped. Always reconcile on app launch:

```swift
func applicationDidBecomeActive() {
    Task {
        // Always fetch changes on activation
        await syncManager.fetchAllChanges()
    }
}
```

### 4. Mark Notifications as Read

```swift
// Mark notification as read to prevent re-delivery
let operation = CKMarkNotificationsReadOperation(notificationIDsToMarkRead: [notification.notificationID!])
container.add(operation)
```

### 5. Limit Notification Payload Size

```swift
// Only include essential keys
notificationInfo.desiredKeys = ["title"]  // Not all fields

// Large payloads may be truncated
```

### 6. Test with Real Devices

- Push notifications don't work in Simulator
- Use at least two physical devices
- Test with app in background/terminated states

### 7. Handle Subscription Limits

```swift
// CloudKit limits subscriptions per database
// If you hit limits, consolidate subscriptions

// Instead of per-record-type subscriptions:
// let taskSub = CKQuerySubscription(recordType: "Task", ...)
// let noteSub = CKQuerySubscription(recordType: "Note", ...)

// Use one zone subscription:
let zoneSub = CKRecordZoneSubscription(zoneID: zoneID)
```

## Anti-Patterns

### ❌ Creating Duplicate Subscriptions

```swift
// BAD: Creates duplicate on every app launch
func application(_ application: UIApplication, didFinishLaunchingWithOptions...) {
    let subscription = CKQuerySubscription(recordType: "Item", predicate: predicate, options: .firesOnRecordCreation)
    database.save(subscription) { _, _ in }
}

// GOOD: Check before creating, use consistent ID
let subscriptionID = "item-creation-subscription"

database.fetch(withSubscriptionID: subscriptionID) { subscription, error in
    if subscription == nil {
        let newSubscription = CKQuerySubscription(
            recordType: "Item",
            predicate: predicate,
            subscriptionID: subscriptionID,
            options: .firesOnRecordCreation
        )
        let info = CKSubscription.NotificationInfo()
        info.shouldSendContentAvailable = true
        newSubscription.notificationInfo = info
        database.save(newSubscription) { _, _ in }
    }
}
```

### ❌ Missing NotificationInfo

```swift
// BAD: Subscription will fail to save
let subscription = CKQuerySubscription(recordType: "Item", predicate: predicate, options: .firesOnRecordCreation)
database.save(subscription) { _, error in }  // Error!

// GOOD: Always configure notificationInfo
let info = CKSubscription.NotificationInfo()
info.shouldSendContentAvailable = true
subscription.notificationInfo = info
```

### ❌ Wrong Subscription Type for Shared Database

```swift
// BAD: CKQuerySubscription doesn't work with shared database
let sharedDB = CKContainer.default().sharedCloudDatabase
let subscription = CKQuerySubscription(recordType: "SharedItem", predicate: predicate, options: .firesOnRecordCreation)
sharedDB.save(subscription) { _, error in }  // Error!

// GOOD: Use CKDatabaseSubscription for shared database
let subscription = CKDatabaseSubscription(subscriptionID: "shared-db-subscription")
let info = CKSubscription.NotificationInfo()
info.shouldSendContentAvailable = true
subscription.notificationInfo = info
sharedDB.save(subscription) { _, _ in }
```

### ❌ Relying Solely on Push for Sync

```swift
// BAD: Only syncing when push arrives
func application(_ application: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable: Any]) {
    syncData()  // Only sync trigger
}

// GOOD: Multiple sync triggers
func applicationDidBecomeActive(_ application: UIApplication) {
    syncData()  // On app launch/foreground
}

func application(_ application: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable: Any]) {
    syncData()  // On notification
}
// Also implement background fetch
```

### ❌ Not Indexing Predicate Fields

```swift
// BAD: Field not indexed in CloudKit Dashboard
let predicate = NSPredicate(format: "category == %@", "news")
// Error: CKError.invalidArguments when saving subscription

// FIX: Enable "Query" indexing for field in CloudKit Dashboard
```

### Subscription Manager Pattern

```swift
class SubscriptionManager {
    private let subscriptionKey = "cloudkit.subscription.created"

    func ensureSubscriptionExists() {
        guard !UserDefaults.standard.bool(forKey: subscriptionKey) else { return }

        let subscription = CKDatabaseSubscription(subscriptionID: "all-changes")
        let info = CKSubscription.NotificationInfo()
        info.shouldSendContentAvailable = true
        subscription.notificationInfo = info

        let operation = CKModifySubscriptionsOperation(
            subscriptionsToSave: [subscription],
            subscriptionIDsToDelete: nil
        )
        operation.modifySubscriptionsResultBlock = { result in
            if case .success = result {
                UserDefaults.standard.set(true, forKey: self.subscriptionKey)
            }
        }
        database.add(operation)
    }
}
```
