# CloudKit Skill

A comprehensive CloudKit skill for AI agents — covering the complete Apple CloudKit framework documentation.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![iOS 15+](https://img.shields.io/badge/iOS-15%2B-green)
![Swift 5.9+](https://img.shields.io/badge/Swift-5.9%2B-orange)

## Overview

This skill provides exhaustive documentation for Apple's CloudKit framework, enabling AI agents to assist with:

- **iCloud data persistence** with automatic cross-device sync
- **CKSyncEngine** implementation (iOS 17+)
- **Real-time sync** via subscriptions and push notifications
- **Data sharing** between iCloud users
- **Error handling** and conflict resolution
- **Privacy compliance** (encryption, GDPR)

## 📁 Files

| File | Size | Description |
|------|------|-------------|
| `SKILL.md` | 12KB | Quick reference and setup guide |
| `references/api-reference.md` | 24KB | Complete API listing — all classes, methods, properties |
| `references/cksyncengine.md` | 12KB | Full CKSyncEngine implementation guide |
| `references/operations.md` | 15KB | All CKOperation classes |
| `references/subscriptions-notifications.md` | 15KB | Push notifications and real-time sync |
| `references/sharing.md` | 13KB | CKShare, participants, zone sharing |
| `references/user-discovery.md` | 8KB | CKUserIdentity and user lookup |
| `references/privacy-security.md` | 10KB | Encryption, access controls, GDPR |
| `references/error-handling.md` | 9KB | All CKError codes and retry strategies |
| `references/schema-design.md` | 9KB | Versioning, migrations, best practices |
| `references/crud-operations.md` | 9KB | Create, Read, Update, Delete operations |
| `references/troubleshooting.md` | 10KB | Common issues and debugging |

**Total: 12 files, ~147KB**

## 🎯 Coverage

Verified against [Apple's official CloudKit documentation](https://developer.apple.com/documentation/cloudkit):

### Core Classes
- `CKContainer`, `CKDatabase`, `CKOperationGroup`
- `CKRecord`, `CKRecord.ID`, `CKRecord.Reference`
- `CKRecordZone`, `CKRecordZone.ID`
- `CKAsset`

### Sync & Operations
- `CKSyncEngine` (iOS 17+) — complete implementation
- All `CKOperation` subclasses (17+)
- `CKQuery`, `CKQueryOperation`
- Change tokens and incremental sync

### Subscriptions & Notifications
- `CKRecordZoneSubscription`
- `CKQuerySubscription`
- `CKDatabaseSubscription`
- `CKNotification` (all types)

### Sharing
- `CKShare`, `CKShare.Participant`, `CKShare.Metadata`
- `UICloudSharingController`
- Zone sharing (iOS 15+)

### User Discovery
- `CKUserIdentity`
- `CKUserIdentity.LookupInfo`
- Discovery operations

### Errors
- All 35+ `CKError` codes
- Retry strategies
- Conflict resolution

## 🚀 Quick Start

```swift
import CloudKit

// Initialize
let container = CKContainer.default()
let privateDB = container.privateCloudDatabase

// Save a record
let record = CKRecord(recordType: "Note")
record["title"] = "My Note"
record["content"] = "Hello CloudKit"
let saved = try await privateDB.save(record)

// Query records
let predicate = NSPredicate(format: "title BEGINSWITH %@", "My")
let query = CKQuery(recordType: "Note", predicate: predicate)
let (results, _) = try await privateDB.records(matching: query)
```

## 📱 CKSyncEngine (iOS 17+)

The recommended approach for sync:

```swift
class SyncManager: CKSyncEngineDelegate {
    private var engine: CKSyncEngine!
    
    init() {
        let config = CKSyncEngine.Configuration(
            database: CKContainer.default().privateCloudDatabase,
            stateSerialization: loadCachedState(),
            delegate: self
        )
        engine = CKSyncEngine(config)
    }
    
    func handleEvent(_ event: CKSyncEngine.Event, syncEngine: CKSyncEngine) async {
        switch event {
        case .stateUpdate(let update):
            saveCachedState(update.stateSerialization)
        case .fetchedRecordZoneChanges(let changes):
            for modification in changes.modifications {
                saveLocally(modification.record)
            }
        default: break
        }
    }
}
```

## 🔧 Installation

### For AI Agents (OpenClaw/Clawd)

Add to your skills directory:
```bash
git clone https://github.com/subsc-taha/cloudkit-skill.git skills/cloudkit
```

### For Reference

Browse the documentation directly on GitHub or clone for local access.

## 📚 Resources

- [Apple CloudKit Documentation](https://developer.apple.com/documentation/cloudkit)
- [Apple Sample: CKSyncEngine](https://github.com/apple/sample-cloudkit-sync-engine)
- [Apple Sample: Zone Sharing](https://github.com/apple/sample-cloudkit-zonesharing)
- [CloudKit Dashboard](https://icloud.developer.apple.com)

## ⚠️ iOS 26+ Note

iOS 26 introduced a thread-safety bug (FB18043319). Initialize CloudKit on main thread:

```swift
import Network

@MainActor
func preloadCloudKit() {
    if #available(iOS 26.0, *) {
        _ = nw_tls_create_options()
    }
}
```

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

Built for [OpenClaw](https://github.com/openclaw/openclaw) AI agents.
