# rowquill

`db/rowquill` is a small row-object ORM for ZuzuScript. It maps
database tables to generated row classes and supports the core
operations expected from simple active-record style objects.

```zzs
from db/rowquill import Schema;
from std/db import DB;

let schema := new Schema( dbh: DB.temp() );

schema.get_dbh().prepare("""
	CREATE TABLE department (
		id int PRIMARY KEY,
		name varchar(200)
	);
""").execute();

schema.add_table( "department", function ( tab ) {
	tab.add_column( "id", "int", primary: true );
	tab.add_column( "name", "varchar", required: true );
	tab.has_many(
		accessor: "employees",
		table:    "employee",
		join:     "my.id = their.dept",
	);
} );

schema.get_dbh().prepare("""
	CREATE TABLE employee (
		id int PRIMARY KEY,
		fullname varchar(200),
		dept int REFERENCES department(id)
	);
""").execute();

schema.add_table( "employee", function ( tab ) {
	tab.add_column( "id", "int", primary: true );
	tab.add_column(
		"fullname",
		"varchar",
		required: true,
		accessor: "name",
	);
	tab.add_column( "dept", "int" );
	tab.has_one(
		accessor: "department",
		table:    "department",
		join:     "my.dept = their.id",
	);
} );

let accounts := schema.table("department").create(
	id: 3,
	name: "Accounts",
);
accounts.insert();

let bob := schema.table("employee").create(
	id: 42,
	name: "Bob",
	dept: 3,
);
bob.insert();

say( bob.department().name() );      // Accounts
say( accounts.employees()[0].name() ); // Bob
```

## Features

- `create`, `find`, `search`, `insert`, `update`, `insert_or_update`,
  and `delete`.
- Generated column accessors with optional alternate names.
- Primary key lookup by scalar, array, or dictionary.
- Dirty tracking so `update` only writes changed non-primary-key fields.
- Raw column storage in `column_data`.
- Inflate and deflate callbacks for structured values such as JSON.
- `length`, `pattern`, `validate`, and `required` column checks.
- `search` supports equality, common comparison operators, `LIKE`,
  `NOT LIKE`, `AND`, `OR`, and `NOT`.
- Simple `has_one` and `has_many` relationships.

## Relationships

Relationships are declared on the table builder:

```zzs
tab.has_one(
	accessor: "department",
	table:    "department",
	join:     "my.dept = their.id",
);

tab.has_many(
	accessor: "employees",
	table:    "employee",
	join:     "my.id = their.dept",
);
```

The current join parser supports equality joins written as
`my.column = their.column`.

## Column Options

`add_column` accepts the SQL column name, a type label, and options.

```zzs
tab.add_column(
	"fullname",
	"varchar",
	"length": 200,
	required: true,
	accessor: "name",
);
```

Supported options include:

- `primary`: marks a primary key column.
- `accessor`: provides an alternate method name.
- `inflate`: converts raw database values to application values.
- `deflate`: converts application values to raw database values.
- `length`: limits raw string length.
- `pattern`: requires raw values to match a regex.
- `validate`: validates application values with a callback.
- `required`: forbids null values in setters and writes.

Multiple `length`, `pattern`, and `validate` options are honoured.
