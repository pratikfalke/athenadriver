# :package: AthenaSQL  [![codecov](https://codecov.io/gh/henrywu2019/athenasql/branch/uber/graph/badge.svg)](https://codecov.io/gh/henrywu2019/athenasql) [![GoDoc][doc-img]][doc] [![Github release][release-img]][release] [![Build Status][ci-img]][ci] [![Go Report Card][report-card-img]][report-card]


<img align="right" height="64" src="resources/logo.png">

`athenasql` is a fully-featured AWS Athena database driver for Go developed at Uber ATG.
It provides a hassle-free way of querying AWS Athena database with Go standard
library. It not only provides basic features of Athena Go SDK, but 
addresses some SDK's limitation, improves and extends it. It also includes
advanced features like Athena workgroup and tagging creation, driver read-only mode and so on.

## Features

Except the basic features provided by Go `database/sql` like error handling, database pool and reconnection, `athenasql` supports the following features out of box:

- Support multiple AWS authorization methods
- Full support of [Athena Basic Data Types](https://docs.aws.amazon.com/athena/latest/ug/data-types.html)
- Full support of [Athena Advanced Type](https://docs.aws.amazon.com/athena/latest/ug/querying-athena-tables.html) for queries with Geospatial identifiers, ML and UDFs 
- Full support of *ALL* Athena Query Statements, including [DDL](https://docs.aws.amazon.com/athena/latest/ug/language-reference.html), [DML](https://docs.aws.amazon.com/athena/latest/ug/functions-operators-reference-section.html) and [UTILITY](https://github.com/aws/aws-sdk-go/blob/master/service/athena/api.go#L5002)
- Support newly added [`INSERT INTO...VALUES`](https://aws.amazon.com/about-aws/whats-new/2019/09/amazon-athena-adds-support-inserting-data-into-table-results-of-select-query/)
- Athena workgroup and tagging support including remote workgroup creation 
- Go sql's prepared statement support 
- Go sql's `DB.Exec()` and `db.ExecContext()` support
- Query cancelling support 
- Mask columns with specific values 
- Database missing value handling 
- Read-Only mode 

## How to set up/install/test `athenasql`


### Prerequisites - AWS Credentials & S3 Query Result Bucket 

To be able to query AWS Athena, you need to have an AWS account at [Amazon AWS's website](https://aws.amazon.com/). To
 give it a shot, a free
 tier account is enough. You also need to have a pair of AWS `access key ID` and `secret access key`.
You can get it from [AWS Security Credentials section of Identity and Access Management (IAM)](https://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html).
If you don't have one, please create it. The following is a screenshot from my temporary free tier account:

![How to create AWS credentials](resources/aws_keys.png)

In addition to AWS credentials, you also need an s3 bucket to store query result. Just go to 
[AWS S3 web console page](https://s3.console.aws.amazon.com/s3/home) to create one.
In the examples below, the s3 bucket I use is `s3://henrywuqueryresults/`.

In most cases, you need the following 4 prerequisites \:

- S3 Output bucket
- `access key ID`
- `secret access key`
- AWS region

For more details on `athenasql`'s support on AWS credentials & S3 query result bucket, please refer to section
 [Support Multiple AWS Authorization Methods](#support-multiple-aws-authentication-methods).

### Installation

```scala
go get github.com/uber/athenasql
```

### Tests

We provide unit tests and integration tests in the codebase.

#### Unit Test

All the unit tests are self-contained and passed even in no-internet environment. Test coverage is 100%. 

```bash
$ cd $GOPATH/src/github.com/uber/athenasql/go
✔ /opt/share/go/path/src/github.com/uber/athenasql [uber|✚ 1…12] 
21:35 $ go test -coverprofile=coverage.out github.com/uber/athenasql/go  && \
 go tool cover -func=coverage.out |grep -v 100.0%
ok  	github.com/uber/athenasql/go	9.255s	coverage: 100.0% of statements
```


#### Integration Test

All integration tests are under [`examples`](https://github.com/uber/athenasql/tree/master/examples) folder. Please make sure all prerequisites are met so that you can run the code on your own machine.

## How to use `athenasql`

`athenasql` is very easy to use. What you need to do it to import it in your code and then use the standard Go `database/sql` as usual.

```scala
import athenasql "github.com/uber/athenasql/go"
```

The following are coding examples to demonstrate `athenasql`'s features and how you should use `athenasql` in youe Go application.
Please be noted the code is for demonstration purpose only, so please follow your own coding style or best practice if necessary.

### Get Started - A Simple Query

The following is the simplest example for demonstration purpose. The source code is available at [dml_select_simple.go](https://github.com/uber/athenasql/blob/master/examples/query/dml_select_simple.go)\.

```scala
package main

import (
	"database/sql"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// Step 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	// Step 2. Open Connection.
	db, _ := sql.Open(drv.DBDriverName, conf.Stringify())
	// Step 3. Query and print results
	var url string
	_ = db.QueryRow("SELECT url from sampledb.elb_logs limit 1").Scan(&url)
	println(url)
}
```

To make it work for you, please replace `OutputBucket`, `Region`, `AccessID` and
 `SecretAccessKey` with your own values. `sampledb` is provided by Amazon so you don't have to worry about it.

To Build it:
```scala
$ go build examples/query/dml_select_simple.go 
```

Run it and you can see output like:
```scala
$ ./dml_select_simple 
https://www.example.com/articles/553
```

### Support Multiple AWS Authentication Methods

`athenasql` uses access keys(Access Key ID and Secret Access Key) to sign programmatic requests to AWS.
When if the AWS_SDK_LOAD_CONFIG environment variable was set, `athenasql` uses [`Shared Config`](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/#pkg-overview), respects [AWS CLI Configuration and Credential File Settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) and gives it even higher priority over the values set in `athenasql.Config`.

#### Use AWS CLI Config For Authentication

When environment variable `AWS_SDK_LOAD_CONFIG` is set, it will read `aws_access_key_id`(AccessID) and `aws_secret_access_key`(SecretAccessKey)
from `~/.aws/credentials`, `region` from `~/.aws/config`. For details about `~/.aws/credentials` and `~/.aws/config`, please check [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).

But you still need to specify correct `OutputBucket` in `athenasql.Config` because it is not in the AWS client config.

`OutputBucket` is critical in Athena. Even if you have a default value set in Athean web console, you must pass one programmatically or you will get error:
`No output location provided. An output location is required either through the Workgroup result configuration setting or as an API input.`


The sample code below enforces AWS_SDK_LOAD_CONFIG is set, so `athenasql`'s AWS Session will be created from the configuration values from the shared config (~/.aws/config) and shared credentials (~/.aws/credentials) files.
Even if we pass all dummy values as parameters in `NewDefaultConfig()` except `OutputBucket`, they are overridden by
the values in AWS CLI config files, so it doesn't really matter.

```scala
// To use AWS CLI's Config for authentication
func useAWSCLIConfigForAuth() {
	os.Setenv("AWS_SDK_LOAD_CONFIG", "1")
	// 1. Set AWS Credential in Driver Config.
	conf, err := drv.NewDefaultConfig(secret.OutputBucketProd, drv.DummyRegion,
		drv.DummyAccessID, drv.DummySecretAccessKey)
	if err != nil {
		return
	}
	// 2. Open Connection.
	db, _ := sql.Open(drv.DriverName, conf.Stringify())
	// 3. Query and print results
	var i int
	_ = db.QueryRow("SELECT 456").Scan(&i)
	println("with AWS CLI Config:", i)
	os.Unsetenv("AWS_SDK_LOAD_CONFIG")
}
```

If your AWS CLI setting is valid like mine, this function should output:

```scala
with AWS CLI Config: 456
```

#### Use `athenasql` Config For Authentication

When environment variable `AWS_SDK_LOAD_CONFIG` is NOT set, you need to pass valid(NOT dummy) `region`, `accessID`, `secretAccessKey` into `athenasql.NewDefaultConfig()`, in addition to `outputBucket`.

The sample code below ensure `AWS_SDK_LOAD_CONFIG` is not set, then pass four valid parameters into `NewDefaultConfig()`:

```scala
// To use athenasql's Config for authentication
func useAthenaSQLConfigForAuth() {
	os.Unsetenv("AWS_SDK_LOAD_CONFIG")
	// 1. Set AWS Credential in Driver Config.
	conf, err := drv.NewDefaultConfig(secret.OutputBucketDev, secret.Region,
		secret.AccessID, secret.SecretAccessKey)
	if err != nil {
		return
	}
	// 2. Open Connection.
	db, _ := sql.Open(drv.DriverName, conf.Stringify())
	// 3. Query and print results
	var i int
	_ = db.QueryRow("SELECT 123").Scan(&i)
	println("with AthenaSQL Config:", i)
}
```
The sample output:

```scala
with AthenaSQL Config: 123
```

The full code is here at [examples/auth.go](https://github.com/henrywu2019/athenasql/tree/master/examples/auth.go).

### Full Support of All Data Types 

As we said, `athenasql` supports all Athena data types. 
In the following sample code, we use an SQL statement to `SELECT` som simple data of all the advanced types and then print them out.

```scala
package main

import (
	"context"
	"database/sql"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, err := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	if err != nil {
		panic(err)
	}
	// 2. Open Connection.
	dsn := conf.Stringify()
	db, _ := sql.Open(drv.DBDriverName, dsn)
	// 3. Query and print results
	query := "SELECT JSON '\"Hello Athena\"', " +
		"ST_POINT(-74.006801, 40.70522), " +
		"ROW(1, 2.0),  INTERVAL '2' DAY, " +
		"INTERVAL '3' MONTH, " +
		"TIME '01:02:03.456', " +
		"TIME '01:02:03.456 America/Los_Angeles', " +
		"TIMESTAMP '2001-08-22 03:04:05.321 America/Los_Angeles';"
	rows, err := db.Query(query)
	if err != nil {
		panic(err)
	}
	defer rows.Close()
	println(drv.ColsRowsToCSV(rows))
}
```

Sample output:
```bash
"Hello Athena",00 00 00 00 01 01 00 00 00 20 25 76 6d 6f 80 52 c0 18 3e 22 a6 44 5a 44 40,
{field0=1, field1=2.0},2 00:00:00.000,0-3,0000-01-01T01:02:03.456-07:52,
0000-01-01T01:02:03.456-07:52,2001-08-22T03:04:05.321-07:00
```

we can see `athenasql` can handle all these advanced types correctly.


### Query With Workgroup and Tag 

`athenasql` supports workgroup and tagging features of Athena. When you query Athena, you can specify the
 workgroup and tags attached with your query. Resource/cost tagging are based on workgroup. If the workgroup doesn't
 exist , by default it will be created programmatically.
 
If you want to disable programmatically creating workgroup and tags, you need to explicitly call:
```scala
Config.SetWGRemoteCreationAllowed(false)
```
In this case, you need to make sure the workgroup you specifies must exist, or you will get error. An example is like
 below:

```scala
package main

import (
	"database/sql"
	"log"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	wgTags := drv.NewWGTags()
	wgTags.AddTag("Uber User", "henry.wu@uber.com")
	wgTags.AddTag("Uber ID", "123456")
	wgTags.AddTag("Uber Role", "SDE")
	// Specify that workgroup `henry_wu` is used for the following query
	wg := drv.NewWG("henry_wu", nil, wgTags)
	conf.SetWorkGroup(wg)
	// comment out the line below to allow remote workgroup creation and
	// the query will be successful!!!
	conf.SetWGRemoteCreationAllowed(false)
	// 2. Open Connection.
	dsn := conf.Stringify()
	db, _ := sql.Open(drv.DBDriverName, dsn)
	// 3. Query and print results
	rows, err := db.Query("select url from sampledb.elb_logs limit 3")
	if err != nil {
		log.Fatal(err)
		return
	}
	defer rows.Close()

	var url string
	for rows.Next() {
		if err := rows.Scan(&url); err != nil {
			log.Fatal(err)
		}
		println(url)
	}
}
```

But I don't have a workgroup named `henry_wu` in AWS Athena, so I got sample output:
```scala
2020/01/20 15:29:52 Workgroup henry_wu doesn't exist and workgroup remote creation
 is disabled.
```


After commenting out `conf.SetWGRemoteCreationAllowed(false)` at line 27, the output becomes:

```scala
https://www.example.com/articles/553
http://www.example.com/images/501
https://www.example.com/images/183
```

and I can see a new workgroup named `henry_wu` is created in AWS Athena console: [https://us-east-2.console.aws
.amazon.com/athena/workgroups/home](https://us-east-2.console.aws.amazon.com/athena/workgroups/home)

![Athena Workgroup and Tags Automatic Creation](resources/workgroup.png)

###  Prepared Statement Support for Athena DB 

Athena doesn't support prepared statement originally. However, it could be very helpful in some 
scenarios like where part of the query is from user input. `athenasql` supports prepared statements 
to help you to deal with those scenarios. An example is as follows:

```scala
package main

import (
	"database/sql"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	// 2. Open Connection.
	db, _ := sql.Open(drv.DBDriverName, conf.Stringify())
	// 3. Prepared Statement
	statement, err := db.Prepare("CREATE TABLE sampledb.urls AS " +
		"SELECT url FROM sampledb.elb_logs where request_ip=? limit ?")
	if err != nil {
		panic(err)
	}
	// 4. Execute prepared Statement
	if result, e := statement.Exec("244.157.42.179", 2); e == nil {
		if rowsAffected, err := result.RowsAffected(); err == nil {
			println(rowsAffected)
		}
	}
}
```

Sample output:
```scala
2
```

You can also use the `?` syntax with `DB.Query()` or `DB.Exec()` directly.

```scala
	rows, err := db.Query("SELECT request_timestamp,elb_name "+
		"from sampledb.elb_logs where url=? limit 1",
		"https://www.example.com/jobs/878")
	if err != nil {
		return
	}
	println(drv.ColsRowsToCSV(rows))
```

Sample Output:
```bash
request_timestamp,elb_name
2015-01-06T04:03:01.351843Z,elb_demo_006
```


###  `DB.Exec()` and `DB.ExecContext()` 

According to Go source code, `DB.Exec()` and `DB.ExecContext()` execute a query that doesn't return rows, 
such as an `INSERT` or `UPDATE`.
It's true that you can use `DB.Exec()` and `DB.Query()` interchangeably to execute the same SQL statements.
\
However, the two methods are for different use cases and return different types of results. According to Go `database/sql` library, the result 
returned from `DB.Exec()` can tell you how many rows were affected by the query and the last inserted ID for `INSERT INTO` statement, 
which is always *-1* for Athena because auto-increment primary key feature is not supported by Athena.\
In contrast, `DB.Query()` will return a `sql.Rows` object which includes all columns and rows details.

When the only concern is if the execution is successful or not, `DB.Exec()` is 
preferred to `DB.Query()` . The best coding practice is:

```scala
if _, err := DB.Exec(`<SQL_STATEMENT>`); err != nil {
    log_or_panic(err)
}
```

In cases of `INSERT INTO`, `CTAG` and `CVAS`, you may want to know when the execution 
is successful how many rows are affected by your query. Then you can use `result.RowsAffected()` as 
demonstrated in the following example:


```scala
package main

import (
	"context"
	"database/sql"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	var conf *drv.Config
	var err error
	if conf, err = drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey"); err != nil {
		panic(err)
	}
	// 2. Open Connection.
	db, _ := sql.Open(drv.DBDriverName, conf.Stringify())
	// 3. Execute and print results
	if _, err = db.ExecContext(context.Background(),
		"DROP TABLE IF EXISTS sampledb.urls"); err != nil {
		panic(err)
	}

	var result sql.Result
	if result, err = db.Exec("CREATE TABLE sampledb.urls AS "+
		"SELECT url FROM sampledb.elb_logs where request_ip=? limit ?",
		"244.157.42.179", 1); err != nil {
		panic(err)
	}
	println(result.RowsAffected())

	if result, err = db.Exec("INSERT INTO sampledb.urls VALUES (?),(?),(?)",
		"abc", "efg", "xyz"); err != nil {
		panic(err)
	}
	println(result.RowsAffected())
	println(result.LastInsertId()) // not supported by Athena
}
```

Sample output:
```scala
1
3
```

### Mask Columns with Specific Values 

Sometimes, database contains sensitive information and you may need to mask columns with specific values. If you don't
 want to display some columns, you can mask them by calling:

```scala
Config.SetMaskedColumnValue("columnName", "maskValue")
```

For example, if you want to mask all rows of column `password`, you can specify:

```scala
Config.SetMaskedColumnValue("password", "xxx")
```

Then all the passwords will be displayed as `xxx` in the query result set. The following is an example to
 mask column `url` in the result set.


```scala
package main

import (
	"database/sql"
	"log"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	conf.SetMaskedColumnValue("url", "xxx")
	// 2. Open Connection.
	dsn := conf.Stringify()
	db, _ := sql.Open(drv.DBDriverName, dsn)
	// 3. Query and print results
	rows, err := db.Query("select request_timestamp, url from " +
		"sampledb.elb_logs limit 3")
	if err != nil {
		log.Fatal(err)
		return
	}
	defer rows.Close()

	var requestTimestamp string
	var url string
	for rows.Next() {
		if err := rows.Scan(&requestTimestamp, &url); err != nil {
			log.Fatal(err)
		}
		println(requestTimestamp + "," + url)
	}
}
```


Sample Output:
```scala
2015-01-03T12:00:00.516940Z,xxx
2015-01-03T12:00:00.902953Z,xxx
2015-01-03T12:00:01.206255Z,xxx
```


### Query Cancellation 

AWS Athena is priced upon the data size it scanned. To save money, `athenasql` supports query cancellation. In
 the following example, the query is cancelled if it is not complete after 2 seconds.  

```scala
package main

import (
	"context"
	"database/sql"
	"log"
	"time"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")

	// 2. Open Connection.
	dsn := conf.Stringify()
	db, _ := sql.Open(drv.DBDriverName, dsn)
	// 3. Query cancellation after 2 seconds
	ctx, _ := context.WithTimeout(context.Background(), 2*time.Second)
	rows, err := db.QueryContext(ctx, "select count(*) from sampledb.elb_logs")
	if err != nil {
		log.Fatal(err)
		return
	}
	defer rows.Close()

	var requestTimestamp string
	var url string
	for rows.Next() {
		if err := rows.Scan(&requestTimestamp, &url); err != nil {
			log.Fatal(err)
		}
		println(requestTimestamp + "," + url)
	}
}
```


Sample Output:
```scala
2020/01/20 15:28:35 context deadline exceeded
```

### Missing Value Handling 

It is common to have missing values in S3 file, or Athena DB. When this happens, you can specify if you want to use
 `empty string` or `default data` as the missing value, whichever is better to facilitate your data processing. The default data for Athena column type are defined as below:
 
```scala
func (r *Rows) getDefaultValueForColumnType(athenaType string) interface{} {
	switch athenaType {
	case "tinyint", "smallint", "integer", "bigint":
		return 0
	case "boolean":
		return false
	case "float", "double", "real":
		return 0.0
	case "date", "time", "time with time zone", "timestamp",
		"timestamp with time zone":
		return time.Time{}
	default:
		return ""
	}
}
```

By default, we use empty string to replace missing values and empty string is preferred to default data. To use
 `default data`, you have to explicitly call:

```scala
Config.SetMissingAsEmptyString(false)
Config.SetMissingAsDefault(true)
```

But if you are strict with your data integrity and want an error raised when data are missing, you can set both of
 them `false`.


### Read-Only Mode 

When read-only mode is enabled in `athenasql`, it only allows retrieving information from Athena database.
Any writing and modification to the database will raise an error. This is useful in some cases. By default, read-only mode
is disabled. To enable it, you need to explicitly call:

```scala
Config.SetReadOnly(true)
```

The following is one example. It enables read-only mode in line 19, but tries to create a new table with CTAS statement.
It ends up with raising an error.

```scala
package main

import (
	"context"
	"database/sql"
	"log"
	drv "github.com/uber/athenasql/go"
)

func main() {
	// 1. Set AWS Credential in Driver Config.
	conf, _ := drv.NewDefaultConfig("s3://henrywuqueryresults/",
		"us-east-2", "DummyAccessID", "DummySecretAccessKey")
	conf.SetReadOnly(true)

	// 2. Open Connection.
	dsn := conf.Stringify()
	db, _ := sql.Open(drv.DBDriverName, dsn)
	// 3. Create Table with CTAS statement
	rows, err := db.QueryContext(context.Background(), 
	  "CREATE TABLE sampledb.elb_logs_new AS " +
		"SELECT * FROM sampledb.elb_logs limit 10;")
	if err != nil {
		log.Fatal(err)
		return
	}
	defer rows.Close()
}
```

Sample Output:
```bash
2020/01/26 01:10:28 writing to Athena database is disallowed in read-only mode
```
## Limitations of Go/Athena SDK's and `athenasql`'s Solution

### Column number mismatch in `GetQueryResults` of Athena Go SDK

#### `ColumnInfo` has more number of cloumns than `Rows[0].Data`

> ![](resources/pin.png)**Affected Statements: DESCRIBE TABLE/VIEW, SHOW SCHEMA/TABLE/...**

- Sample Query:

```sql
DESC sampledb.elb_logs
```

- Analysis:

![Column number mismatch issue example 1](resources/issue_1.png)

We can see there are 3 columns according to `ColumnInfo` under `ResultSetMetadata`. But in the first row `Rows[0]`, we see there is only 1 field: `"elb_name \tstring    \t    "`. I would imagine there could have been 3 items in the `Data[0]`, but somehow the code author doesn't split it with tab(`\t`), so it ends up with only 1 item. The same issue happens for `SHOW` statement.

For more sample code, please check [util_desc_table.go](https://github.com/uber/athenasql/blob/master/examples/query/util_desc_table.go), [util_desc_view.go](https://github.com/uber/athenasql/blob/master/examples/query/util_desc_view.go), and [util_show.go](https://github.com/uber/athenasql/blob/master/examples/query/util_show.go).

- `awsathendriver`'s Solution:

`athenasql` fixes this issue by splitting `Rows[0].Data[0]` string with tab, and replace the original row with a new row which has the same number of data with columns.

#### `ColumnInfo` has cloumns but `Rows` are empty

> ![](resources/pin.png)**Affected Statements: [`CTAS`](https://docs.aws.amazon.com/athena/latest/ug/ctas.html), CVAS, INSERT INTO**

Sample Query:

```sql
CREATE TABLE sampledb.elb_logs_copy WITH (
    format = 'TEXTFILE',
    external_location = 's3://external-location-henrywu/elb_logs_copy', 
    partitioned_by = ARRAY['ssl_protocol'])
AS SELECT * FROM sampledb.elb_logs
```

Analysis:

![Column number mismatch issue example 2](resources/issue_3.png)

In the above [`CTAS`](https://docs.aws.amazon.com/athena/latest/ug/ctas.html) statement, we see there is one column of type `bigint` named
 `"rows"` in the resultset, but `ResultSet.Rows` is empty. Since there is no
  row, that one column doesn't make sense, or at least is confusing. The same
  issue happens for `INSERT INTO` statement.

- `awsathendriver`'s Solution:

Because this issue happens only in statements [`CTAS`](https://docs.aws.amazon.com/athena/latest/ug/ctas.html), `CVAS`, and `INSERT INTO
`, where `UpdateCount` is always valid and is the only meaningful information
 returned from Athena, `athenasql` sets `UpdateCount` as the value of
  the returned row.

For more sample code, please check [ddl_ctas.go](https://github.com/uber/athenasql/blob/master/examples/query/ddl_ctas.go), [ddl_cvas.go](https://github.com/uber/athenasql/blob/master/examples/query/ddl_cvas.go), and [dml_insert_into.go](https://github.com/uber/athenasql/blob/master/examples/query/dml_insert_into.go).

### Type Loss for map, struct, array etc

One of Athena Go SDK's limitations is the type information could be lost after 
querying. I think there are two reasons for this type infromation loss.

The first reason is Athena SDK doesn't provide the full type information for complex type data.
It assumes the application developers know the data schema and should take the responsibility of data serialization.

To dig into the code, all query results are stored in data structure 
[`ResultSet`](https://docs.aws.amazon.com/athena/latest/APIReference/API_ResultSet.html).
From the UML class graph of `ResultSet` below, we can see the type 
information are stored in `ColumnInfo`'s pointer to string variable `Type`, 
which is only a type name of data type, not containing any type metadata. 
For example, querying a map of `string->boolean` will return the type name `map`, 
but you cannot find the information `string->boolean` in the `ResultSet`. For simple type like `integer`, 
`boolean` or `string`, it is sufficient to serialize them to Go type, but for more complex types like `array`, 
`struct`, `map` or nested types, the type information is lost here.

![UML class graph of `ResultSet`](resources/ResultSet_Uml.png)

The second reason is the difference between Athena data type and Go data type. 
Some Athena builtin data type like `Row`, `DECIMAL(p, s)`, `varbinary`, `interval year to month`
are not supported in Go standard library. Therefore, there is no way to serialize them in driver level.

- `awsathendriver`'s Solution:

For data types: `array`, `map`, `json`, `char`, `varchar`, `varbinary`, `row`, `string`, `binary`, `struct`, `interval year to month`, `interval day to second`, `decimal`, `awsathendriver` returns the string representation of the data. The developers can firstly retrieve the string representation, and then serialize to user defined type on their own.

For time and date types: `date`, `time`, `time with time zone`, `timestamp`, `timestamp with time zone`, `awsathendriver` returns Go's [`time.Time`](https://golang.org/pkg/time/#Time).

Some sample code are available at [dml_select_array.go](https://github.com/uber/athenasql/blob/master/examples/query/dml_select_array.go),
[dml_select_map.go](https://github.com/uber/athenasql/blob/master/examples/query/dml_select_map.go), [dml_select_time.go](https://github.com/uber/athenasql/blob/master/examples/query/dml_select_time.go).


## FAQ

The following is a collection of questions from our software developers and data scientists.

### Does `athenasql` support database reconnection?

Yes. `database/sql` maintains a connection pool internally and handles connection pooling, reconnecting, and retry logic for you.
One pitfall of writing Go sql application is cluttering the code with error-handling and retry.
I tested in my application with `athenasql` by turning off and on Wifi and VPN, it works very well with database reconnection.

### Does `athenasql` support batched query?
  
No. `athenasql` is an implementation of `sql.driver` in Go `database/sql`, where there is no batch query support.
There might be some workaround for some specific case though. For instance, 
if you want to insert many rows, you can use [db.Exec](https://golang.org/pkg/database/sql/#DB.Exec) 
by replacing multiple inserts with one insert and multiple VALUES.
 
### How to use `athenasql` to get total row number of result set?

You have to use `rows.Next()` to iterate all rows and use a counter to get row number. It is because Go `database/sql` was designed in a streaming query way with big data considered. That is why it only supports using `Next()` to iterate. So there is no way for random access of row. In Athena case, we only have random access of all the rows within one result page as the picture shown below:

![Encapsulation of driver.Rows in sql.Rows](resources/sql_Rows.png) 

But due to encapsulation, more sepcifically the `rowsi` is _private_, we
 cannot access it directly like when we using Athena Go SDK. We have to use `Next()` to access it one by one.

### Is there any way to randomly access row with `athenasql`?

No. The reason is the same as answer to the previous question.

### Does `athenasql` support getting the rows affected by my query?
  
To put it simple, YES. But there is some limitation and best practice to follow.
  
The recommended way is to use `DB.Exec()` to get it. Please refer to \ref{db-exec}.

You can get it with `DB.Query()` too. In the returned `ResultSet`, there is
 an `UpdateCount` member variable. If the query is one of [`CTAS`](https://docs.aws.amazon.com/athena/latest/ug/ctas.html), `CVAS` and `INSERT INTO`, `UpdateCount` will contain meaningful value. The result will be of a one row and one column. The column name is `rows`, and the row is an `int`, which is exactly `UpdateCount`. I would suggest to use `QueryRow` or `QueryRowContext` since it is a one-row result. By the way, the document for [`GetQueryResults`](https://docs.aws.amazon.com/athena/latest/APIReference/API_GetQueryResults.html) seems not very accurate.

![UpdateCount for CTAS, VTAS, and INSERT INTO](resources/issue_2.png)

In practice, not only [`CTAS`](https://docs.aws.amazon.com/athena/latest/ug/ctas.html) statement but also `CVAS` and `INSERT INTO` will make a meaningful `UpdateCount`.

## Development Status: Stable

All APIs are finalized, and no breaking changes will be made in the 1.x series of releases.

This library is now at version 1 and follows [SemVer](http://semver.org/) strictly.


## Contributing

We encourage and support an active, healthy community of contributors &mdash;
including you! Details are in the [contribution guide](resources/CONTRIBUTING.md) and
the [code of conduct](resources/CODE_OF_CONDUCT.md). The athenasql maintainers keep an eye on
issues and pull requests, but you can also report any negative conduct to
[**oss-conduct@uber.com**](oss-conduct@uber.com). That email list is a private, safe space; even the athenasql
maintainers don't have access, so don't hesitate to hold us to a high
standard.


### `athenasql` UML Class Diagram

For the contributors, the following is `athenasql` Package's UML Class Diagram which may help you to
 understand the code. You can also check the reference section below for some useful materials. 


![`athenasql` Package's UML Class Diagram](resources/athenasql.png)


## Reference 

- [Amazon Athena User Guide](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
- [Amazon Athena API Reference - Describes the Athena API operations in detail.](https://docs.aws.amazon.com/athena/latest/APIReference/Welcome.html)
- [Amazon Athena Go Doc](https://godoc.org/github.com/aws/aws-sdk-go/service/athena)
- [Data type mappings that the JDBC driver supports between Athena, JDBC, and Java](https://s3.amazonaws.com/athena-downloads/drivers/JDBC/SimbaAthenaJDBC_2.0.5/docs/Simba+Athena+JDBC+Driver+Install+and+Configuration+Guide.pdf#page=37)
- [Service Quotas](https://docs.aws.amazon.com/athena/latest/ug/service-limits.html)
- [Go sql connection pool](http://go-database-sql.org/connection-pool.html)
- [Common Pitfalls When Using database/sql in Go](https://www.vividcortex.com/blog/2015/09/22/common-pitfalls-go/)
- [Implement Sql Database Driver in 100 Lines of Go](https://vyskocil.org/blog/implement-sql-database-driver-in-100-lines-of-go/)


`athenasql` is brought to you by Uber ATG Infrastructure. Copyright (c) 2020 Uber Technologies, Inc.

<img align="right" height="64" src="resources/atg-infra.png">


[doc-img]: http://img.shields.io/badge/GoDoc-Reference-blue.svg
[doc]: https://godoc.org/uber/athenasql

[release-img]: https://img.shields.io/badge/athenasql-v1.0.0-brightgreen
[release]: https://github.com/uber/athenasql/releases

[ci-img]: https://api.travis-ci.com/uber/athenasql.svg?branch=master
[ci]: https://travis-ci.com/uber/athenasql/branches

[report-card-img]: https://goreportcard.com/badge/github.com/henrywu2019/athenasql
[report-card]: https://goreportcard.com/report/github.com/henrywu2019/athenasql

[release-policy]: https://golang.org/doc/devel/release.html#policy