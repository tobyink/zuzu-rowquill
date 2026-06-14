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
		id int,
		company varchar(20),
		name varchar(200),
		PRIMARY KEY (id, company)
	);
""").execute();

schema.add_table( "department", function ( tab ) {
	tab.add_column( "id", "int", primary: true );
	tab.add_column( "company", "varchar", primary: true );
	tab.add_column( "name", "varchar", required: true );
	tab.has_many(
		accessor: "employees",
		table:    "employee",
		join:     { id: "dept", company: "company" },
		where:    { is_deleted: false },
	);
} );

schema.get_dbh().prepare("""
	CREATE TABLE employee (
		id int PRIMARY KEY,
		fullname varchar(200),
		dept int,
		company varchar(20),
		is_deleted bool
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
	tab.add_column( "company", "varchar" );
	tab.add_column( "is_deleted", "bool" );
	tab.has_one(
		accessor: "department",
		table:    "department",
		join:     { dept: "id", company: "company" },
	);
	tab.add_helper( "is_accountant", function ( employee ) {
		return employee.department().name() eq "Accounts";
	} );
} );

let accounts := schema.table("department").create(
	id: 3,
	company: "ACME",
	name: "Accounts",
);
accounts.insert();

let bob := schema.table("employee").create(
	id: 42,
	name: "Bob",
	dept: 3,
	company: "ACME",
	is_deleted: false,
);
bob.insert();

say( bob.department().name() );      // Accounts
say( accounts.employees()[0].name() ); // Bob
say( bob.is_accountant() );          // true
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
- `has_one` and `has_many` relationships with multi-column joins and
  optional search-style `where` filters.
- Custom row helper methods and static table-class methods.

## Relationships

Relationships are declared on the table builder:

```zzs
tab.has_one(
	accessor: "department",
	table:    "department",
	join:     { dept: "id", company: "company" },
);

tab.has_many(
	accessor: "employees",
	table:    "employee",
	join:     { id: "dept", company: "company" },
	where:    { is_deleted: false },
);
```

The `join` mapping keys are columns on the current row's table. Mapping
values are columns on the related table. Multiple mappings are combined
with AND.

The optional `where` condition is also applied to the related table and
uses the same condition style as `search`, including `AND`, `OR`, `NOT`,
and operator arrays.

## Helper Methods

`add_helper` adds a method to generated row objects. The callback
receives the row object as its first argument, followed by any
positional and named arguments passed to the generated method.

```zzs
tab.add_helper( "is_accountant", function ( employee ) {
	return employee.department().name() eq "Accounts";
} );

bob.is_accountant();  // true
```

`add_static` adds a static method to the generated table class. The
callback receives the table class as its first argument.

```zzs
tab.add_static( "in_department", function ( Employee, String department ) {
	return Employee.search(
		dept: schema.table("department").search( name: department )[0].id(),
	);
} );

schema.table("employee").in_department("Accounts");
```

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
