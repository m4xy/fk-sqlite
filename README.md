# fk-sqlite

[SQLite](https://sqlite.org) for [fckt](https://github.com/m4xy/fckt): SQL run through an effect, so
a function's type says whether it touches a database, and rows read into typed values a column at a
time, with every problem in a row found at once.

```kotlin
import uk.m4xy.sql.*
import uk.m4xy.sql.Columns.*
import uk.m4xy.sql.Databases.*

type User(id: Int, name: String, email: Option<String>)

users: Column<User> = Columns {
  id by int("id")
  name by text("name")
  email by nullable(text("email"))
  User(id, name, email)
}

@Database @Fail named(name: String): List<User> = queryAs(users, "SELECT * FROM users WHERE name = ?", [Text(name)])

main() = Databases.opening("shop.db") {
  script("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT)")
  transaction {
    insert("INSERT INTO users (name, email) VALUES (?, ?)", [Text("Ada"), Text("ada@example.com")])
  }
  named("Ada").forEach { println(it.name) }
}
```

A [kiln](https://github.com/m4xy/kiln) project uses it as `library("uk.m4xy:fk-sqlite:0.1.0")`, with
`ocamlPackages = ["sqlite3"]` in its `FcktLayout`, as this project's `build.fk` has. It needs the
OCaml package: `opam install sqlite3`.

## Running SQL

`Databases.opening(path) { ... }` opens the database in the file, made if it isn't there, or
`":memory:"` for one in memory, and closes it after, even if what's in it fails. Inside, through
`@Database`:

| | |
|---|---|
| `query(sql, values)` | the rows the query gives, each a `Row` of columns and `Value`s |
| `queryAs(columns, sql, values)` | each row read by the columns |
| `queryOne(columns, sql, values)` | the first row, read, if there is one |
| `execute(sql, values)` | runs it, and gives how many rows it changed |
| `insert(sql, values)` | runs it, and gives the row ID it inserted |
| `script(sql)` | runs each statement in it, with no values: a schema, say |
| `transaction { ... }` | keeps what it does if it finishes, and undoes it if it fails |

A `?` in the SQL is given the next of the values, a `Value`: `Null`, `Integer`, `Real`, `Text` or
`Blob`. Values are never written into the SQL, so what they hold can't change what it does. What
SQLite can't do fails, with SQLite's reason and the SQL.

## Reading rows

`Columns` is an applicative, as fk-cli's `Arguments` is: `int`, `real`, `text`, `blob`, `boolean`
and `value` read a column by name, and `nullable(column)` gives `None` for null. A block of them reads
a row into anything, and a row that can't be read fails with every problem in it:

```
column name is the text 'Ada', not an integer; column admin is the integer 7, not 0 or 1; there's
no column missing, reading a row of: SELECT * FROM users
```

## Testing what uses a database

A function that runs SQL is `@Database`, and that's all it knows of the database, so a test can run
it on `":memory:"`, or handle `Database` itself, as `test/database.fk` does to see the statements a
function runs without SQLite at all. Under `Database` is `SqlEngine`, which `Sqlite` handles: the one
place SQLite itself is called.
