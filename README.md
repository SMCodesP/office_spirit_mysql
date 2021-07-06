MySQL and MariaDB driver for Deno. Tested on: MySQL 5.6, 8.0, MariaDB 5.5, 10.0, 10.2, 10.5.

Features:
- Prepared statements.
- Binary protocol. Query parameters are sent separately from text query.
- Sane connections pooling. Connections are reset after usage (locks are freed).
- Pool for connections to multiple servers.
- Streaming BLOBs.
- Custom handler for LOCAL INFILE.
- Made with CPU and RAM efficiency in mind.

Basic example:

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.query("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text)");
		await conn.query("INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3')");

		for await (let row of await conn.query("SELECT * FROM t_log"))
		{	console.log(row);
		}
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Connections

Connections to database servers are managed by `MyPool` object.

```ts
new MyPool(options?: MyPoolOptions|Dsn|string)
```

Options are:

```ts
interface MyPoolOptions
{	dsn?: Dsn|string;
	maxConns?: number;
	onLoadFile?: (filename: string) => Promise<(Deno.Reader & Deno.Closer) | undefined>;
	sqlPolicy?: SqlPolicy;
}
```
- `dsn` - Default data source name of this pool.
- `maxConns` - Limit to number of simultaneous connections in this pool. When reached `pool.haveSlots()` returns false, and new connection request will wait.
- `onLoadFile` - Handler for `LOAD DATA LOCAL INFILE` query.
- `sqlPolicy` - whitelisted or blacklisted SQL identifiers and functions for SQL fragments, when using `sql` template literal to generate an SQL string (see below).

Options can be given just as DSN string, or a `Dsn` object, that contains parsed DSN string.

Data source name is specified in URL format, with "mysql://" protocol.

Format: `mysql://user:password@host:port/schema?param1=value1&param2=value2#INITIAL_SQL`
Or: `mysql://user:password@localhost/path/to/named.pipe/schema`

Example: `mysql://root@localhost/`
Or: `mysql://root:hello@[::1]/?keepAliveTimeout=10000&foundRows`

Possible parameters:
- `keepAliveTimeout` (number) milliseconds - each connection will persist for this period of time, before termination, so it can be reused when someone else asks for the same connection
- `keepAliveMax` (number) - how many times at most to recycle each connection
- `maxColumnLen` (number) bytes - if a column was longer, it's value is skipped, and it will be returned as NULL (this doesn't apply to `conn.makeLastColumnReader()` - see below)
- `foundRows` (boolean) - if present, will use "found rows" instead of "affected rows" in resultsets (see [here](https://dev.mysql.com/doc/refman/8.0/en/information-functions.html#function_row-count) how CLIENT_FOUND_ROWS flag affects result of `Row_count()` function)
- `ignoreSpace` (boolean) - if present, parser on server side can ignore spaces before '(' in built-in function names (see description [here](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_ignore_space))
- `multiStatements` (boolean) - if present, SQL can contain multiple statements separated with ';', so you can upload dumps, but SQL injection attacks become more risky

Connection from the pool can be asked with `pool.forConn()` function:

```ts
MyPool.forConn<T>(callback: (conn: MyConn) => Promise<T>, dsn?: Dsn|string): Promise<T>
```
If `dsn` is not provided, the default DSN of the pool will be used. You can ask connections to different servers.

The requested connection will be available in the provided `callback`, and when it completes, this connection will return back to pool.

Connection state is reset before returning to the pool. This means that incomplete transaction will be rolled back, and all kind of locks will be cleared.
Then this connection can be idle in the pool for at most `keepAliveTimeout` milliseconds, and if nobody was interested in it during this period, it will be terminated.
If somebody killed a connection while it was idle in the pool, and you asked to use this connection again, the first query on this connection can fail.
If this happens, another connection will be tried, and your query will be retried. This process is transparent to you.

In the beginning of `callback`, `conn` may be not connected to the server. It will connect on first requested query.

If you want to deal with multiple simultaneous connections, you can call `pool.session()` to start a cross-server session.

```ts
MyPool.session<T>(callback: (session: MySession) => Promise<T>): Promise<T>
```
During this session you can call `session.conn()` to get a connection. At the end of callback all the connections will return to the pool, if they didn't before.

```ts
MySession.conn(dsn?: Dsn|string, fresh=false): MyConn
```
`MySession.conn()` returns the connection object (`MyConn`) immediately, but actual connection will be established on first SQL query.

With `true` second argument, always new connection is returned. Otherwise, if there's already a connection to the same DSN in this session, it will be picked up.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root@localhost');

pool.session
(	async (session) =>
	{	let conn1 = session.conn(); // default DSN
		let conn2 = session.conn(); // the same object
		let conn3 = session.conn(undefined, true); // another connection to default DSN
		let conn4 = session.conn('mysql://tests@localhost'); // connection to different DSN

		console.log(conn1 == conn2); // prints true

		let connId2 = conn2.queryCol("SELECT Connection_id()").first();
		let connId3 = conn3.queryCol("SELECT Connection_id()").first();
		let connId4 = conn4.queryCol("SELECT Connection_id()").first();

		console.log(await Promise.all([connId2, connId3, connId4])); // prints 3 different connection ids
	}
);

await pool.onEnd();
pool.closeIdle();
```
At the end of callback all active connections will be returned to the pool. However you can call `conn.end()` to free a connection earlier.

## Making queries

`MyConn` object has the following methods for making simple queries:

```ts
MyConn.execute(sql: SqlSource, params?: object|null): Promise<Resultsets>
MyConn.query(sql: SqlSource, params?: object|null): ResultsetsPromise
MyConn.queryMap(sql: SqlSource, params?: object|null): ResultsetsPromise
MyConn.queryArr(sql: SqlSource, params?: object|null): ResultsetsPromise
MyConn.queryCol(sql: SqlSource, params?: object|null): ResultsetsPromise
```
`execute` method executes it's query and discards returned rows.
Returned `Resultsets` object contains `lastInsertId`, `affectedRows`, and more such information about the query.
If there were multiple resultsets, it will contain only information about the last one.

`query*` methods return `ResultsetsPromise` which is subclass of `Promise<Resultsets>`.
Awaiting it gives you `Resultsets` object.
Iterating over `Resultsets` yields rows.

If the query you executed didn't return rows (query like `INSERT`), then zero rows will be yielded, and `resultsets.columns` will be empty array.
`resultsets.lastInsertId` and `resultsets.affectedRows` will show relevant information.

If there're rows, you need to iterate them to the end, before you can execute another query.
You can read all the rows with `Resultsets.all()` or `ResultsetsPromise.all()`.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text)");
		await conn.execute("INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3')");

		console.log(await conn.query("SELECT * FROM t_log").all()); // ResultsetsPromise.all()

		let res = await conn.query("SELECT * FROM t_log");
		console.log(res.columns);
		console.log(await res.all()); // Resultsets.all()
	}
);

await pool.onEnd();
pool.closeIdle();
```
If your query returns single row, you can read it with `Resultsets.first()` or `ResultsetsPromise.first()`.
It returns the first row itself, not an array of rows.
And it skips all further rows, if they exist.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text)");
		await conn.execute("INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3')");

		console.log(await conn.query("SELECT Count(*) FROM t_log").first()); // ResultsetsPromise.all()

		let res = await conn.query("SELECT Count(*) FROM t_log");
		console.log(res.columns);
		console.log(await res.first()); // Resultsets.all()
	}
);

await pool.onEnd();
pool.closeIdle();
```
You can iterate the resultset with `for await` loop, or you can call `ResultsetsPromise.forEach()` or `Resultsets.forEach()` method.

```ts
ResultsetsPromise.forEach<T>(callback: (row: any) => T|Promise<T>): Promise<T|undefined>
```

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text)");
		await conn.execute("INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3')");

		// for await loop
		for await (let row of await conn.query("SELECT * FROM t_log"))
		{	console.log(row);
		}

		// ResultsetsPromise.forEach()
		await conn.query("SELECT * FROM t_log").forEach
		(	row =>
			{	console.log(row);
			}
		);
	}
);

await pool.onEnd();
pool.closeIdle();
```

- `MyConn.query()` method iterates over rows as Javascript default objects with fields.
- `MyConn.queryMap()` method iterates over rows as `Map` objects.
- `MyConn.queryArr()` method iterates over rows as `Array`s with column values without column names.
- `MyConn.queryCol()` method iterates over first column values of each row.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text)");
		await conn.execute("INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3')");

		console.log(await conn.queryCol("SELECT Count(*) FROM t_log").first()); // prints 3
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Query parameters

### Positional parameters

You can use `?` placeholders in SQL query strings, and supply array of parameter values to be substituted in place of them.
This library doesn't parse the provided SQL string, but uses MySQL built-in functionality, so the parameters are substituted on MySQL side.
Placeholders can appear only in places where expressions are allowed.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, `time` timestamp, message text)");
		await conn.execute("INSERT INTO t_log SET `time`=Now(), message='Message 1'");

		let row = await conn.query("SELECT `time` + INTERVAL ? DAY AS 'time', message FROM t_log WHERE id=?", [3, 1]).first();
		console.log(row);
	}
);

await pool.onEnd();
pool.closeIdle();
```

### Named parameters

For named parameters you can use `@name` placeholders, and this library uses MySQL session variables to send parameters data.
To execute such query, another pre-query is sent to the server, like `SET @days=?, @id=?`.
Parameter names will override session variables with the same names.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, `time` timestamp, message text)");
		await conn.execute("INSERT INTO t_log SET `time`=Now(), message='Message 1'");

		let row = await conn.query("SELECT `time` + INTERVAL @days DAY AS 'time', message FROM t_log WHERE id=@`id`", {days: 3, id: 1}).first();
		console.log(row);
	}
);

await pool.onEnd();
pool.closeIdle();
```

### Generate SQL string with quoted parameters

This library also provides string-template function that escapes SQL values in strings and quoted literals.

```ts
import {sql} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let message = `It's the message`;
let number = 0.1;
let column = 'The number';
console.log('' + sql`SELECT '${message}', '${number}' AS "${column}"`); // prints: SELECT 'It''s the message', 0.1 AS `The number`
```

The `sql` template function provides the following functionality:

1. To quote values, surround the parameter with apostrophes, like `'${param}'`. If the parameter is a string, characters inside it will be properly escaped (according to noBackslashEscapes argument of `toString()`). If the value is a number, quotes around it will be removed. If it's a `null`, or an `undefined`, a Javascript function or a Symbol, it will be substituted with `NULL` literal. If it's boolean `true` or `false`, it will be substituted with `TRUE` and `FALSE` respectively. `Date` objects will be printed as MySQL dates. Typed arrays will be printed like `x'0102...'`.
2. To quote identifiers (column, table, routine names, etc.) surround the parameter with backticks or double quotes, like `"${param}"`. Double quotes will be replaced with backticks.
3. To insert a safe SQL expression, surround the parameter with parentheses. The inserted SQL fragment will be validated not to contain the following characters (unless quoted): `@ [ ] { } ;`, commas except in parentheses, comments, unterminated literals, unbalanced parentheses. Identifiers in this SQL fragment will be backtick-quoted according to chosen policy. Strings in the SQL fragment are always treated as `noBackslashEscapes` (backslash is regular character), so to represent a string with a new line, you need `const expr = "Char_length('Line\n')"`, not `const expr = "Char_length('Line\\n')"`.

The `sql` template function returns `Sql` object that can be stringified, or converted to bytes.

```ts
Sql.toString(noBackslashEscapes=false): string

Sql.encode(noBackslashEscapes=false, useBuffer?: Uint8Array): Uint8Array
```

Also the `Sql` object has public property called `sqlPolicy`, that allows to whitelist identifiers in SQL fragments.

```ts
Sql.sqlPolicy: SqlPolicy | undefined
```

```ts
import {sql, SqlPolicy} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

const value1 = "The string is: 'name'. The backslash is: \\";
const value2 = 123.4;
const value3 = null;
const expr1 = "id=10 AND value IS NOT NULL";

let select = sql`SELECT '${value1}', '${value2}', '${value3}' FROM t WHERE (${expr1})`;

console.log(select+'');             // SELECT 'The string is: ''name''. The backslash is: \\', 123.4, NULL FROM t WHERE (`id`=10 AND `value` IS NOT NULL)
console.log(select.toString(true)); // SELECT 'The string is: ''name''. The backslash is: \', 123.4, NULL FROM t WHERE (`id`=10 AND `value` IS NOT NULL)

select.sqlPolicy = new SqlPolicy('id not');
console.log(select+'');             // SELECT 'The string is: ''name''. The backslash is: \\', 123.4, NULL FROM t WHERE (id=10 `AND` `value` `IS` NOT `NULL`)
```

If you pass the `Sql` object (the result of calling `sql` template function) to functions like `conn.execute()` or `conn.query()`, the object will be converted to bytes using the correct value for `noBackslashEscapes`, that is found on `conn.noBackslashEscapes`. The `sqlPolicy` can be provided to the `Pool` object after it's creation, and before starting to create connections.

```ts
import {MyPool, sql, SqlPolicy} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.options({sqlPolicy: new SqlPolicy('AND OR XOR NOT')});

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, `time` timestamp, message text)");
		await conn.execute("INSERT INTO t_log SET `time`=Now(), message='Message 1'");

		const timeColumnName = 'time';
		const days = 3;
		const id = 1;
		let row = await conn.query(sql`SELECT "${timeColumnName}" + INTERVAL '${days}' DAY AS 'time', message FROM t_log WHERE id='${id}'`).first();
		console.log(row);
	}
);

await pool.onEnd();
pool.closeIdle();
```

The `SqlPolicy` object allows to specify how to quote raw identifiers and functions in SQL fragments.

```ts
SqlPolicy.constructor(idents?: string, functions?: string)
```

If `idents` and/or `functions` argument is omitted or `undefined`, the default value is used.

For `idents` the default value is: `NOT AND OR XOR BETWEEN SEPARATOR IS NULL DISTINCT LIKE CHAR MATCH AGAINST INTERVAL YEAR MONTH WEEK DAY HOUR MINUTE SECOND MICROSECOND CASE WHEN THEN ELSE END`.

For `functions` is: `! SELECT VALUES`.

The policy is specified by whitespace-separated list of identifiers. If the first character is `!`, so it's a blacklist policy. Otherwise it's whitelist.

To print the default policy, you can do:

```ts
import {SqlPolicy} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let policy = new SqlPolicy;

console.log('Identifiers policy: ', policy.idents);
console.log('Functions policy: ', policy.functions);
```

## Reading long BLOBs

This library tries to have everything needed in real life usage. It's possible to read long data without storing it in memory.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.query("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, `time` timestamp, message text)");
		await conn.query("INSERT INTO t_log SET `time`=Now(), message='long long message'");

		let row = await conn.makeLastColumnReader("SELECT `time`, message FROM t_log WHERE id=1");
		await Deno.copy(row.message, Deno.stdout);
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Writing long BLOBS

Query parameter values can be of various types, including `Deno.Reader`. If some parameter is `Deno.Reader`, the parameter value will be read from this reader (without storing the whole BLOB in memory).

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	await conn.execute("CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, `time` timestamp, message text)");

		let file = await Deno.open('/etc/passwd', {read: true});
		try
		{	// Write the file to db
			await conn.execute("INSERT INTO t_log SET `time`=Now(), message=?", [file]);
		}
		finally
		{	file.close();
		}

		// Read the contents back from db
		let row = await conn.makeLastColumnReader("SELECT `time`, message FROM t_log WHERE id=1");
		await Deno.copy(row.message, Deno.stdout);
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Importing big dumps

Functions like `MyConn.execute()`, `MyConn.query()`, etc. allow to provide SQL query in several forms.

```ts
MyConn.query(sql: SqlSource, params?: object|null): ResultsetsPromise;

type SqlSource = string | Uint8Array | Sql | Deno.Reader&Deno.Seeker | Deno.Reader&{readonly size: number};
```
This allows to read SQL from files.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests?multiStatements');

pool.forConn
(	async (conn) =>
	{	let filename = await Deno.makeTempFile();
		try
		{	await Deno.writeTextFile
			(	filename,
				`	CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, c_time timestamp, message text);

					INSERT INTO t_log SET c_time=Now(), message='long long message';
				`
			);

			let file = await Deno.open(filename, {read: true});
			try
			{	await conn.execute(file);
			}
			finally
			{	file.close();
			}

			console.log(await conn.query("SELECT c_time, message FROM t_log").all());
		}
		finally
		{	await Deno.remove(filename);
		}
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Prepared statements

Function `conn.forQuery()` prepares an SQL statement, that you can execute multiple times, each time with different parameters.

```ts
forQuery<T>(sql: SqlSource, callback: (prepared: Resultsets) => Promise<T>): Promise<T>
```

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests');

pool.forConn
(	async (conn) =>
	{	// CREATE TABLE
		await conn.query("CREATE TEMPORARY TABLE t_messages (id integer PRIMARY KEY AUTO_INCREMENT, message text)");

		// INSERT
		await conn.forQuery
		(	"INSERT INTO t_messages SET message=?",
			async (prepared) =>
			{	for (let i=1; i<=3; i++)
				{	await prepared.exec(['Message '+i]);
				}
			}
		);

		// SELECT
		console.log(await conn.query("SELECT * FROM t_messages").all());
	}
);

await pool.onEnd();
pool.closeIdle();
```

## LOAD DATA LOCAL INFILE

If this feature is enabled on your server, you can register a custom handler that will take `LOAD DATA LOCAL INFILE` requests.

```ts
import {MyPool, sql} from 'https://deno.land/x/office_spirit_mysql/mod.ts';
import {dirname} from "https://deno.land/std@0.97.0/path/mod.ts";

let pool = new MyPool('mysql://root:hello@localhost/tests');

// Set handler for LOAD DATA LOCAL INFILE queries
const ALLOWED_DIRS = ['/tmp'];
pool.options
(	{	async onLoadFile(filename: string)
		{	if (ALLOWED_DIRS.includes(dirname(filename)))
			{	return Deno.open(filename, {read: true});
			}
		}
	}
);

// Download some public example CSV file from github
let data = await fetch('https://raw.githubusercontent.com/lukes/ISO-3166-Countries-with-Regional-Codes/master/all/all.csv');
let filename = await Deno.makeTempFile();
await Deno.writeTextFile(filename, await data.text());

// Create temporary table, load the data to it, and then select it back
pool.forConn
(	async (conn) =>
	{	// CREATE TABLE
		await conn.execute
		(	`	CREATE TEMPORARY TABLE t_countries
				(	country_code char(2) CHARACTER SET latin1 NOT NULL PRIMARY KEY,
					country_name varchar(128) NOT NULL
				)
			`
		);

		// LOAD DATA
		let res = await conn.execute
		(	sql
			`	LOAD DATA LOCAL INFILE '${filename}'
				INTO TABLE t_countries
				FIELDS TERMINATED BY ','
				ENCLOSED BY '"'
				IGNORE 1 LINES
				(@name, @code)
				SET
					country_code = @code,
					country_name = @name
			`
		);
		console.log(res.statusInfo);

		// SELECT
		console.log(await conn.query("SELECT * FROM t_countries LIMIT 3").all());
	}
);

await pool.onEnd();
pool.closeIdle();
```

## Connection status

`MyConn` object has several status variables:

- `conn.serverVersion: string` - remote server version, as it reports (for example my server reports "8.0.25-0ubuntu0.21.04.1").
- `conn.connectionId: number` - thread ID of the connection, that `SHOW PROCESSLIST` shows.
- `conn.autocommit: boolean` - true if the connection is currently in autocommit mode. Queries like `SET autocommit=0` will affect this flag.
- `conn.inTrx: boolean` - true if a transaction was started. Queries like `START TRANSACTION` and `ROLLBACK` will affect this flag.
- `conn.inTrxReadonly: boolean` - true if a readonly transaction was started. Queries like `START TRANSACTION READ ONLY` and `ROLLBACK` will affect this flag.
- `conn.noBackslashEscapes: boolean` - true, if the server is configured not to use backslash escapes in string literals. Queries like `SET sql_mode='NO_BACKSLASH_ESCAPES'` will affect this flag.
- `conn.schema: string` - if your server supports change schema notifications, this will be current default schema (database) name. Queries like `USE new_schema` will affect this value.

Initially these variables can be empty. They are set after actual connection to the server, that happens after issuing the first query. Or you can call `await conn.connect()`.

## Resultsets

`conn.execute()`, and `conn.query*()` methods all return `Resultsets` object, that contains information about your query result.
Also this object allows to iterate over rows that the query returned.

If your query returned multiple resultsets, `conn.execute()` skips them, and returns only the status of the last one.

`conn.query*()` functions don't skip resultsets, and `await resultsets.nextResultset()` will advance to the next result, and return true.
If there are no more resultsets, `await resultsets.nextResultset()` returns false.
And you must read or discard all the resultsets before being able to issue next queries.

```ts
import {MyPool} from 'https://deno.land/x/office_spirit_mysql/mod.ts';

let pool = new MyPool('mysql://root:hello@localhost/tests?multiStatements');

pool.forConn
(	async (conn) =>
	{	let resultsets = await conn.query
		(	`	CREATE TEMPORARY TABLE t_log (id integer PRIMARY KEY AUTO_INCREMENT, message text);

				INSERT INTO t_log (message) VALUES ('Message 1'), ('Message 2'), ('Message 3');

				SELECT * FROM t_log;
			`
		);

		console.log(resultsets.affectedRows); // prints 0

		await resultsets.nextResultset();
		console.log(resultsets.affectedRows); // prints 3

		await resultsets.nextResultset();
		console.log(resultsets.columns.length); // prints 2

		for await (let row of resultsets)
		{	console.log(row);
		}
	}
);

await pool.onEnd();
pool.closeIdle();
```

`Resultsets` object has the following properties and methods:

- `Resultsets.lastInsertId: number|bigint` - In INSERT queries this is last generated AUTO_INCREMENT ID
- `Resultsets.affectedRows: number|bigint` - In modifying queries, like INSERT, UPDATE and DELETE this shows how many rows were affected by the query
- `Resultsets.foundRows: number|bigint` - If "foundRows" connection attribute is set, will ask the server to report about "found rows" (matched by the WHERE clause), instead of affected, and "affectedRows" will not be used. See [this page](https://dev.mysql.com/doc/c-api/5.7/en/mysql-affected-rows.html) for more information.
- `Resultsets.warnings: number` - Number of warnings produced by the last query. To see the warning messages you can use `SHOW WARNINGS` query.
- `Resultsets.statusInfo: string` - Human-readable information about last query result, if sent by server.
- `Resultsets.noGoodIndexUsed: boolean` - Server can report about nonoptimal queries.
- `Resultsets.noIndexUsed: boolean`
- `Resultsets.isSlowQuery: boolean`
- `Resultsets.columns: Column[]` - Information about columns in resultset.
- `Resultsets.placeholders: Column[]` - Information about `?` placeholders in the SQL query.
- `Resultsets.hasMore: boolean` - True if there are more rows or resultsets to read.
- `Resultsets.exec(params: any[]): Promise<void>` - If this is a prepared query, this function executes it again.
- `Resultsets.all(): Promise<any[]>` - Reads all rows in current resultset to an array.
- `Resultsets.first(): Promise<any>` - Reads all rows in current resultset, and returns the first row.
- `Resultsets.forEach<T>(callback: (row: any) => T | Promise<T>): Promise<T | undefined>` - Reads all rows, and calls the provided callback for each of them.
- `Resultsets.nextResultset(): Promise<boolean>` - Advances to the next resultset of this query, if there is one. Returns true if moved to the next resultset.
- `Resultsets.discard(): Promise<void>` - Reads and discards all the rows in all the resultsets of this query.
