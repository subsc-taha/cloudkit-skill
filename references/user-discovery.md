# CloudKit User Discovery

Find and identify iCloud users for sharing and collaboration.

## Table of Contents
1. [Overview](#overview)
2. [CKUserIdentity](#ckuseridentity)
3. [Discovering Users](#discovering-users)
4. [Current User](#current-user)
5. [Lookup Methods](#lookup-methods)
6. [Privacy Considerations](#privacy-considerations)

## Overview

CloudKit provides user discovery to:
- Find other iCloud users to share with
- Get the current user's identity
- Look up users by email or phone

**Important**: Users must explicitly grant permission for discovery.

## CKUserIdentity

Represents an iCloud user:

```swift
let identity: CKUserIdentity

// User record ID (for CloudKit operations)
let userRecordID = identity.userRecordID

// Contact info (if discoverable)
let lookupInfo = identity.lookupInfo
let email = lookupInfo?.emailAddress
let phone = lookupInfo?.phoneNumber

// Display name components
let nameComponents = identity.nameComponents
let givenName = nameComponents?.givenName   // "John"
let familyName = nameComponents?.familyName  // "Doe"

// Check if contacts match
let hasContacts = identity.hasiCloudAccount
```

### CKUserIdentity.LookupInfo

Used to find users:

```swift
// By email
let lookupInfo = CKUserIdentity.LookupInfo(emailAddress: "user@example.com")

// By phone number
let lookupInfo = CKUserIdentity.LookupInfo(phoneNumber: "+1234567890")

// By user record ID
let lookupInfo = CKUserIdentity.LookupInfo(userRecordID: recordID)
```

## Discovering Users

### Discover Users in Contacts

Find iCloud users from the device's contacts:

```swift
let operation = CKDiscoverAllUserIdentitiesOperation()

var discoveredUsers: [CKUserIdentity] = []

operation.userIdentityDiscoveredBlock = { identity in
    discoveredUsers.append(identity)
    print("Found: \(identity.nameComponents?.givenName ?? "Unknown")")
}

operation.discoverAllUserIdentitiesResultBlock = { result in
    switch result {
    case .success:
        print("Found \(discoveredUsers.count) users")
    case .failure(let error):
        print("Discovery failed: \(error)")
    }
}

CKContainer.default().add(operation)
```

### Async/Await Version

```swift
func discoverAllUsers() async throws -> [CKUserIdentity] {
    var users: [CKUserIdentity] = []
    
    let operation = CKDiscoverAllUserIdentitiesOperation()
    operation.userIdentityDiscoveredBlock = { identity in
        users.append(identity)
    }
    
    return try await withCheckedThrowingContinuation { continuation in
        operation.discoverAllUserIdentitiesResultBlock = { result in
            switch result {
            case .success:
                continuation.resume(returning: users)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
        CKContainer.default().add(operation)
    }
}
```

## Current User

### Get Current User Identity

```swift
let container = CKContainer.default()

// Fetch current user's identity
container.fetchUserRecordID { recordID, error in
    guard let recordID = recordID else {
        print("Error: \(error?.localizedDescription ?? "Unknown")")
        return
    }
    
    // Now fetch full identity
    container.discoverUserIdentity(withUserRecordID: recordID) { identity, error in
        guard let identity = identity else { return }
        
        print("Current user: \(identity.nameComponents?.givenName ?? "Unknown")")
    }
}

// Async version
let recordID = try await container.userRecordID()
let identity = try await container.userIdentity(forUserRecordID: recordID)
```

### Check Account Status

```swift
let status = try await CKContainer.default().accountStatus()

switch status {
case .available:
    print("iCloud available")
case .noAccount:
    print("No iCloud account")
case .restricted:
    print("iCloud restricted")
case .couldNotDetermine:
    print("Could not determine status")
case .temporarilyUnavailable:
    print("Temporarily unavailable")
@unknown default:
    break
}
```

## Lookup Methods

### Look Up by Email

```swift
let lookupInfos = [
    CKUserIdentity.LookupInfo(emailAddress: "friend1@example.com"),
    CKUserIdentity.LookupInfo(emailAddress: "friend2@example.com")
]

let operation = CKDiscoverUserIdentitiesOperation(userIdentityLookupInfos: lookupInfos)

operation.userIdentityDiscoveredBlock = { identity, lookupInfo in
    print("Found \(lookupInfo.emailAddress ?? "unknown"): \(identity.nameComponents?.givenName ?? "")")
}

operation.discoverUserIdentitiesResultBlock = { result in
    switch result {
    case .success:
        print("Lookup complete")
    case .failure(let error):
        print("Lookup failed: \(error)")
    }
}

CKContainer.default().add(operation)
```

### Look Up for Sharing

```swift
// Find user to add as share participant
func findUserForSharing(email: String) async throws -> CKShare.Participant? {
    let lookupInfo = CKUserIdentity.LookupInfo(emailAddress: email)
    
    let operation = CKFetchShareParticipantsOperation(userIdentityLookupInfos: [lookupInfo])
    
    return try await withCheckedThrowingContinuation { continuation in
        var foundParticipant: CKShare.Participant?
        
        operation.perShareParticipantResultBlock = { lookupInfo, result in
            if case .success(let participant) = result {
                foundParticipant = participant
            }
        }
        
        operation.fetchShareParticipantsResultBlock = { result in
            switch result {
            case .success:
                continuation.resume(returning: foundParticipant)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
        
        CKContainer.default().add(operation)
    }
}
```

## Privacy Considerations

### Request Permission

```swift
let container = CKContainer.default()

container.requestApplicationPermission(.userDiscoverability) { status, error in
    switch status {
    case .granted:
        print("User granted discoverability")
    case .denied:
        print("User denied discoverability")
    case .couldNotComplete:
        print("Could not complete: \(error?.localizedDescription ?? "")")
    case .initialState:
        print("Not yet requested")
    @unknown default:
        break
    }
}

// Async version
let status = try await container.requestApplicationPermission(.userDiscoverability)
```

### Check Current Permission

```swift
let container = CKContainer.default()

container.status(forApplicationPermission: .userDiscoverability) { status, error in
    switch status {
    case .granted:
        // Can discover users
        discoverUsers()
    case .denied:
        // Show explanation UI
        showDiscoverabilityExplanation()
    default:
        // Request permission
        requestDiscoverabilityPermission()
    }
}
```

### What Users Control

Users can control discoverability in **Settings → iCloud → Apps Using iCloud → Look Me Up By Email**

| Setting | Effect |
|---------|--------|
| On | Other apps can find user by email/phone |
| Off | User won't appear in discovery results |

### Best Practices

1. **Explain why** before requesting permission
2. **Handle denial gracefully** — offer manual invite
3. **Don't cache identities** too long — they can change
4. **Respect privacy** — only discover when needed
5. **Provide alternatives** — share links work without discovery

```swift
func inviteUser(email: String, share: CKShare) async {
    // Try discovery first
    if let participant = try? await findUserForSharing(email: email) {
        share.addParticipant(participant)
    } else {
        // Fall back to email invite
        sendEmailInvite(to: email, shareURL: share.url)
    }
}
```
