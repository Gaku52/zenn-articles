---
title: "Core Data応用"
---

# Core Data応用

本章では、Core Dataのより高度な機能と最適化手法を解説します。

## バックグラウンドコンテキスト

```swift
func performBackgroundTask<T>(_ block: @escaping (NSManagedObjectContext) throws -> T) async throws -> T {
    try await withCheckedThrowingContinuation { continuation in
        CoreDataManager.shared.persistentContainer.performBackgroundTask { context in
            do {
                let result = try block(context)
                continuation.resume(returning: result)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

## Fetch Request の最適化

```swift
let fetchRequest = User.fetchRequest()
fetchRequest.predicate = NSPredicate(format: "age >= %d", 18)
fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
fetchRequest.fetchLimit = 20
fetchRequest.fetchBatchSize = 20
```

## NSFetchedResultsController

```swift
class UserListViewModel: NSObject, ObservableObject {
    private let fetchedResultsController: NSFetchedResultsController<User>

    init() {
        let fetchRequest = User.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]

        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: fetchRequest,
            managedObjectContext: CoreDataManager.shared.viewContext,
            sectionNameKeyPath: nil,
            cacheName: nil
        )

        super.init()
        fetchedResultsController.delegate = self

        try? fetchedResultsController.performFetch()
    }
}
```

## まとめ

Core Dataの高度な機能を活用することで、パフォーマンスとユーザー体験が向上します。次章では、Realmデータベースについて解説します。
