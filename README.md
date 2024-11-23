# Remaking the GRDB demo app

The GRDB Players demo app is fantastic for learning the basics of GRDB.  Going through it helped me a lot but for me to really understand the demo app (not all of GRDB) I felt like rebuilding it step by step would be very helpful.  I'm attempting to rebuild it by commits so that each commit adds another step so one can go through by commit and understand each piece.

This readme will also attempt to explain to some degree each step, especially ones that confused me.

The best place to start is always the beginning.

---

## The basics
### The first thing we need is a test we want to pass.

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
