---
title: "Core Data基礎"
---

# Core Data基礎

Core DataはApple公式のデータ永続化フレームワークです。本章では、Core Dataの基本的な使い方を解説します。

## Core Data Stackのセットアップ

```swift
class CoreDataManager {
    static let shared = CoreDataManager()

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "AppModel")
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Unable to load persistent stores: \(error)")
            }
        }
        return container
    }()

    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    func save() {
        let context = viewContext
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                print("Error saving context: \(error)")
            }
        }
    }
}
```

## CRUD操作

```swift
// Create
let user = User(context: context)
user.id = UUID()
user.name = "John"
user.email = "john@example.com"
try context.save()

// Read
let fetchRequest = User.fetchRequest()
let users = try context.fetch(fetchRequest)

// Update
user.name = "Jane"
try context.save()

// Delete
context.delete(user)
try context.save()
```

## まとめ

Core Dataを使うことで、大量のデータを効率的に管理できます。次章では、Core Dataの高度な活用方法について解説します。
