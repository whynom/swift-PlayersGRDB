# Remaking the GRDB demo app

The GRDB Players demo app is fantastic for learning the basics of GRDB.  Going through it helped me a lot but for me to really understand the demo app (not all of GRDB) I felt like rebuilding it step by step would be very helpful.  

This readme will also attempt to explain to some degree each step, especially ones that confused me.

The best place to start is always the beginning.

---

## Part 1.  The basics

### Things that will be accomplished

1. Make a database 
2. Make a `Player`
3. Save that `Player` to the database
4. Read information about the `Player` from our created database.

### Add the package dependency
The dependency can be added to the project in XCode by going to

File -> Add package dependency

Enter the github url for the package in the search bar and install
https://github.com/groue/GRDB.swift


### Make a test we want to pass.
We add these tests so we can get all the basics going.  Once we can build and pass all these tests we'll be able to

Remember to add GRDB to your app's testing target as well or you'll get an error on the import GRDB line.

``` swift
import Testing
import GRDB
@testable import PlayersGRDB

struct PlayersGRDBTests {

    @Test func insert() throws {
        // Given an empty database
        let appDatabase = try makeEmptyTestDatabase()

        // When we insert a player
        var insertedPlayer = Player(name: "Arthur", score: 1000)
        try appDatabase.savePlayer(&insertedPlayer)

        // Then the inserted player has an id
        #expect(insertedPlayer.id != nil)

        // Then the inserted player exists in the database
        let fetchedPlayer = try appDatabase.reader.read(Player.fetchOne)

        #expect(fetchedPlayer == insertedPlayer)

    }

    /// Return an empty, in-memory, `AppDatabase`.
    private func makeEmptyTestDatabase() throws -> AppDatabase {
        let dbQueue = try DatabaseQueue(configuration: AppDatabase.makeConfiguration())
        return try AppDatabase(dbQueue)
    }
}
```

This just tests that we can make an empty database and insert a record into it.

Of course just adding that to our tests files, pretty much every one of those lines will fail, so let's attempt to start getting rid of those errors.


### The beginnings of the `AppDatabase` class.
The basics of a database are the `Writer` and a `Migrator` which depends upon that `Writer`.  The `Migrator` should make it so that whenever we make changes to our database, we can migrate from the old version to the new one without any problems.

Make sure GRDB is also added to the app target in the general tab of the app under Frameworks, Libraries, and Embedded Content.

Let's make a Database folder in our main app folder and add a file named AppDatabase.  Add the following code to that file.

``` swift
import Foundation
import GRDB

final class AppDatabase: Sendable {

    private let dbWriter: any DatabaseWriter

    init(_ dbWriter: any GRDB.DatabaseWriter) throws {
        self.dbWriter = dbWriter
        try migrator.migrate(dbWriter)
    }

    private var migrator: DatabaseMigrator {
        var migrator = DatabaseMigrator()

#if DEBUG
        migrator.eraseDatabaseOnSchemaChange = true
#endif

        migrator.registerMigration("v1") { db in
            try db.create(table: "player") { t in
                t.autoIncrementedPrimaryKey("id")
                t.column("name", .text).notNull()
                t.column("score", .integer).notNull()
            }
        }

        return migrator
    }
}
```

### AppDatabase `makeConfiguration` function
This extension is added to our `AppDatabase` file to fullfill our need for a configuration

``` swift
extension AppDatabase {
    static func makeConfiguration(_ config: Configuration = Configuration()) -> Configuration {
        return config
    }
}
```

###  `Player` model
We also need a very basic `Player` model so we have something to test.

Make a folder called `Models` in our `Database` folder and add a new file called `Player`.  Add the following code to that file.

``` swift
import GRDB

struct Player: Equatable {
    var id: Int64?
    var name: String
    var score: Int
}
```

### Necessary player conformance

In our first test, we wish to be able to save a player to our database, so we need the type to conform to `Codable` `FetchableRecord` and `MutablePersistableRecord`.  This also defines our table columns and gives us the `didInsert` function.

Within the `Player` file add the following.

``` swift
extension Player: Codable, FetchableRecord, MutablePersistableRecord {
    enum Columns {
        static let name = Column(CodingKeys.name)
        static let score = Column(CodingKeys.score)
    }

    mutating func didInsert(_ inserted: InsertionSuccess) {
        id = inserted.rowID
    }
}
```

### We need a reader
Our test fails here because we have no reader.
``` swift
let fetchedPlayer = try appDatabase.reader.read(Player.fetchOne) // Value of type 'AppDatabase' has no member 'reader'
```

We add this to our AppDatabase file.
``` swift
extension AppDatabase {
    /// Provides a read-only access to the database.
    var reader: any GRDB.DatabaseReader {
        dbWriter
    }
}

```

###  We need to save a player
Add this extension in our Player file so we can save a `Player`.

```swift
extension AppDatabase {
    func savePlayer(_ player: inout Player) throws {
        try dbWriter.write { db in
            try player.save(db)
        }
    }
}
```
### Add a very basic reader
Add this to our AppDatabase file
``` swift
extension AppDatabase {
    /// Provides a read-only access to the database.
    var reader: any GRDB.DatabaseReader {
        dbWriter
    }
}
```

### What we've done
That was a lot of code to slop on all in one sitting, but it's the _bare_ minimum needed to get things up and running.  None of it was terribly difficult but as we add to it, we'll see where the parts of the code we've omitted will fail us, and add those pieces of code then so we can see precisely what everything does. 
