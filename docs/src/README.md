# idris2-sqlite3 Idris2 Bindings to the sqlite3 C-API

First and foremost, this library provides
low-level bindings to the SQLite C-API, that can be used by any
Idris projects planning to interact with SQLite databases. In addition,
a domain-specific language (DSL) for writing correctly typed SQL queries and
statements is provided. Finally, derivable marshallers for storing Idris
values in and retrieving them from SQLite databases are provided.

A lot of this is still work in progress, but several things turned out
quite nice already (in my opinion). Below is a short example application
with some comments. First, some imports:

```idris
module README

import Data.WithID
import Derive.Sqlite3
import Control.RIO.Sqlite3

%default total
%language ElabReflection
```

We start by defining two SQLite tables: One for employees in
a company, and a second one for organisational units. Employees
are linked to the unit they work in by means of a foreign key,
while each unit has a supervisor, who are themselves employees:

```idris
public export
Units : Table
Units =
  table "units"
  [ C "unit_id" INTEGER
  , C "name"    TEXT
  , C "head"    INTEGER
  ]

public export
Employees : Table
Employees =
  table "employees"
    [ C "employee_id" INTEGER
    , C "name"        TEXT
    , C "salary"      REAL
    , C "unit_id"     INTEGER
    ]
```

As you can see, tables are defined by giving them a name together with
a list of columns. Every column comes with its name and the SQLite type
of the values stored in that column.

We can now define the commands for creating the two tables:

```idris
export
createUnits : Cmd TCreate
createUnits =
  createTable Units
    [ PrimaryKey ["unit_id"]
    , AutoIncrement "unit_id"
    , ForeignKey Employees ["head"] ["employee_id"]
    , NotNull "name"
    , Unique ["name"]
    ]

export
createEmployees : Cmd TCreate
createEmployees =
  createTable Employees
    [ PrimaryKey ["employee_id"]
    , AutoIncrement "employee_id"
    , ForeignKey Units ["unit_id"] ["unit_id"]
    , NotNull "name"
    , NotNull "salary"
    , NotNull "unit_id"
    , Check ("salary" > 0)
    ]
```

As you can see, column and table constraints are listed in the command
used to create tables. All string literals in the code above are being
checked to actually be proper column names of the table we define.

In addition, we might want to define a bunch of Idris types for
querying the database as well as for inserting data. We could just
use heterogeneous lists for that (from `Data.List.Quantifiers`, and
this library would be perfectly fine with that, but for results
we use a lot in our code, proper record types might be more
convenient).

```idris
public export
record OrgUnit (h : Type) where
  constructor U
  name : String
  head : h

%runElab derive "OrgUnit" [Show, Eq, ToRow, FromRow]

public export
record Employee (u : Type) where
  constructor E
  name   : String
  salary : Double
  unit   : u

%runElab derive "Employee" [Show, Eq, ToRow, FromRow]
```

As you can see, we leave one value in each record type abstract. This
will depend on what kind of information we provide or collect
for the given field. Note also, that we derived interfaces of type
`FromRow` and `ToRow, which allow us to convert values of these types from and
to rows in a database table. Let's write the code for inserting
values next:

```idris
export
insertUnit : OrgUnit Bits32 -> Cmd TInsert
insertUnit = insert Units ["name", "head"]

export
insertEmployee : Employee Bits32 -> Cmd TInsert
insertEmployee = insert Employees ["name", "salary", "unit_id"]
```

When inserting stuff, we use `Bits32` numbers for the identifiers
of foreign keys in the tables. Note, that the record types do not
include a field for the primary keys in the tables. We want these
to be auto-generated by SQLite itself.

I'm now going to show how to query the database. If we want to fetch
full rows from a table, it is convenient to wrap our record values
in a type called `WithID` if we want to include the identifier
in the list of results.

```idris
rawEmployee : Query (WithID $ Employee Bits32)
rawEmployee =
  SELECT
    ["employee_id", "name", "salary", "unit_id"]
    [< FROM Employees]
  `ORDER_BY` [ASC "salary", ASC "name"]
```

The query DSL is designed in such a way that it resembles proper
SQL, while being correctly typed (and checked) by Idris.

A more useful thing to do would probably be to list the name of each
employee's unit instead of the unit's ID. This requires an `INNER JOIN`,
which is just as easy to set up:

```idris
employee : Query (WithID $ Employee String)
employee =
  SELECT
    ["e.employee_id", "e.name", "e.salary", "u.name"]
    [< FROM (Employees `AS` "e")
    ,  JOIN (Units `AS` "u") `USING` ["unit_id"]
    ]
  `WHERE`    ("e.salary" > 3000.0)
  `ORDER_BY` [ASC "e.salary", ASC "e.name"]
```

This example also shows that we can rename the tables and
columns in a query and that we can reference them by these
new names. Here's what happens in case of a typo:

```idris
failing "Can't find an implementation for IsJust"
  employeeFail : Query (WithID $ Employee String)
  employeeFail =
    SELECT
      ["e.employee_id", "e.name", "e.salary", "u.name"]
      [< FROM (Employees `AS` "e")
      ,  JOIN (Units `AS` "u") `USING` ["unit_id"]
      ]
    `ORDER_BY` [ASC "e.sulary", ASC "e.name"]
```

The error message might not be very helpful, but at least Idris checks
that we didn't make any mistakes.

As a final example, let's compute some basic statistics about
the organizational units. Because I'm too lazy to come up with
another record type, we just return a heterogeneous list in
this case, which can be simplified by using the `LQuery` type
alias:

```idris
unitStats : LQuery [String,Bits32,Double,Double,Double]
unitStats =
  SELECT
    [ "u.name"
    ,  Count "e.name" `AS` "num_employees"
    ,  Avg "e.salary" `AS` "average_salary"
    ,  Min "e.salary" `AS` "min_salary"
    ,  Max "e.salary" `AS` "max_salary"
    ]
    [< FROM (Employees `AS` "e")
    ,  JOIN (Units `AS` "u") `USING` ["unit_id"]
    ]
    `GROUP_BY` ["e.unit_id"]
    `ORDER_BY` [ASC "average_salary"]
```

Finally, let's run some example code. We are going to use the
`Control.RIO.App` effect type to get proper error handling.
We can wrap and run a list of commands in a single transaction
by using the `cmds` utility. For querying the database, we
can use `query`, which takes a `Query t` argument plus a
natural number corresponding to the maximal number of rows
we want to collect and accumulate in a list.

```idris
0 Errs : List Type
Errs = [SqlError]

handlers : All (Handler ()) Errs
handlers = [ printLn ]

app : App Errs ()
app = withDB ":memory:" $ do
  cmds $
    [ createUnits
    , createEmployees
    , insertUnit $ U "Sales" 1
    , insertUnit $ U "R&D" 2
    , insertUnit $ U "HR" 3
    , insertEmployee $ E "Sarah" 8300.0 1
    , insertEmployee $ E "Ben" 8000.0 2
    , insertEmployee $ E "Gil" 7750.0 3
    , insertEmployee $ E "Cathy" 3000.0 1
    , insertEmployee $ E "John" 3100.0 1
    , insertEmployee $ E "Abby" 3000.0 2
    , insertEmployee $ E "May" 3100.0 2
    , insertEmployee $ E "Brian" 3000.0 2
    , insertEmployee $ E "Benny" 3100.0 2
    , insertEmployee $ E "Rob" 3100.0 3
    , insertEmployee $ E "Zelda" 3100.0 3
    , insertEmployee $ E "Gundi" 2050.0 1
    , insertEmployee $ E "Valeri" 5010.0 1
    , insertEmployee $ E "Ronja" 4010.0 1
    ]
  es <- query employee 1000
  traverse_ printLn es
  ss <- query unitStats 1000
  traverse_ printLn ss

main : IO ()
main = runApp handlers app
```

If you are using [pack](https://github.com/stefan-hoeck/idris2-pack) for
managing and installing Idris dependencies, you can compile and run
this example program simply by typing `pack exec docs/src/README.md`
from this project's root directory.

## Next Steps

I'm planning to write a proper tutorial for this library explaining
in detail how to use it and also discussing some of the design
decisions that led to the current implementation. This is still
work in progress, so stay tuned.

## Credits

This library is based on previous work by
@MarcelineVQ: [idris-sqlite3](https://github.com/MarcelineVQ/idris-sqlite3).

<!-- vi: filetype=idris2:syntax=markdown
-->