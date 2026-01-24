---
title: "Realmデータベース"
---

# Realmデータベース

Realmは、モバイル向けに最適化された高速なデータベースです。本章では、Realmの基本的な使い方を解説します。

## Realmの導入

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/realm/realm-swift.git", from: "10.40.0")
]
```

## モデル定義

```swift
import RealmSwift

class User: Object {
    @Persisted(primaryKey: true) var id: UUID
    @Persisted var name: String
    @Persisted var email: String
    @Persisted var createdAt: Date
}
```

## CRUD操作

```swift
// Create
let realm = try! Realm()
let user = User()
user.id = UUID()
user.name = "John"
user.email = "john@example.com"
user.createdAt = Date()

try! realm.write {
    realm.add(user)
}

// Read
let users = realm.objects(User.self)
let filteredUsers = users.filter("age >= 18").sorted(byKeyPath: "createdAt", ascending: false)

// Update
try! realm.write {
    user.name = "Jane"
}

// Delete
try! realm.write {
    realm.delete(user)
}
```

## まとめ

Realmは高速でシンプルなAPIを提供し、Core Dataの代替として活用できます。次章では、UserDefaultsのベストプラクティスについて解説します。
