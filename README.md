![OctoSQL](images/octosql.svg)OctoSQL
=======
OctoSQL is a query tool that allows you to join, analyse and transform data from multiple databases and file formats using SQL.

[![CircleCI](https://circleci.com/gh/cube2222/octosql.svg?style=shield)](https://circleci.com/gh/cube2222/octosql)
[![GoDoc](https://godoc.org/github.com/cube2222/octosql?status.svg)](https://godoc.org/github.com/cube2222/octosql)
[![Gitter](https://badges.gitter.im/octosql/general.svg)](https://gitter.im/octosql/general?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

## Table of Contents
- [What is OctoSQL?](#what-is-octosql)
- [Installation](#installation)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
  - [JSON](#json)
  - [CSV](#csv)
  - [Excel](#excel)
  - [Parquet](#parquet)
  - [PostgreSQL](#postgresql)
  - [MySQL](#mysql)
  - [Redis](#redis)
  - [Kafka](#kafka)
- [Documentation](#documentation)
- [Architecture](#architecture)
- [Datasource Pushdown Operations](#datasource-pushdown-operations)
- [Roadmap](#roadmap)

## What is OctoSQL?
OctoSQL is a SQL query engine which allows you to write standard SQL queries on data stored in multiple SQL databases, NoSQL databases and files in various formats trying to push down as much of the work as possible to the source databases, not transferring unnecessary data. 

OctoSQL does that by creating an internal representation of your query and later translating parts of it into the query languages or APIs of the source databases. Whenever a datasource doesn't support a given operation, OctoSQL will execute it in memory, so you don't have to worry about the specifics of the underlying datasources. 

With OctoSQL you don't need O(n) client tools or a large data analysis system deployment. Everything's contained in a single binary.

### Why the name?
OctoSQL stems from Octopus SQL.

Octopus, because octopi have many arms, so they can grasp and manipulate multiple objects, like OctoSQL is able to handle multiple datasources simultaneously.

## Installation
Either download the binary for your operating system (Linux, OS X and Windows are supported) from the [Releases page](https://github.com/cube2222/octosql/releases), or install using the go command line tool:
```bash
GO111MODULE=on go get -u github.com/cube2222/octosql/cmd/octosql
```

## Quickstart
Let's say we have a csv file with cats, and a redis database with people (potential cat owners). Now we want to get a list of cities with the number of distinct cat names in them and the cumulative number of cat lives (as each cat has up to 9 lives left).

First, create a configuration file ([Configuration Syntax](#configuration))
For example:
```yaml
dataSources:
  - name: cats
    type: csv
    config:
      path: "~/Documents/cats.csv"
  - name: people
    type: redis
    config:
      address: "localhost:6379"
      password: ""
      databaseIndex: 0
      databaseKeyName: "id"
```

Then, set the **OCTOSQL_CONFIG** environment variable to point to the configuration file.
```bash
export OCTOSQL_CONFIG=~/octosql.yaml
```
You can also use the --config command line argument.

Finally, query to your hearts desire:
```bash
octosql "SELECT p.city, FIRST(c.name), COUNT(DISTINCT c.name) cats, SUM(c.livesleft) catlives
FROM cats c JOIN people p ON c.ownerid = p.id
GROUP BY p.city
ORDER BY catlives DESC
LIMIT 9"
```
Example output:
```
+---------+--------------+------+----------+
| p.city  | c.name_first | cats | catlives |
+---------+--------------+------+----------+
| Warren  | Zoey         |   68 |      570 |
| Gadsden | Snickers     |   52 |      388 |
| Staples | Harley       |   54 |      383 |
| Buxton  | Lucky        |   45 |      373 |
| Bethany | Princess     |   46 |      366 |
| Noxen   | Sheba        |   49 |      361 |
| Yorklyn | Scooter      |   45 |      359 |
| Tuttle  | Toby         |   57 |      356 |
| Ada     | Jasmine      |   49 |      351 |
+---------+--------------+------+----------+
```
You can choose between table, tabbed, json and csv output formats.

## Configuration
The configuration file has the following form
```yaml
dataSources:
  - name: <table_name_in_octosql>
    type: <datasource_type>
    config:
      <datasource_specific_key>: <datasource_specific_value>
      <datasource_specific_key>: <datasource_specific_value>
      ...
  - name: <table_name_in_octosql>
    type: <datasource_type>
    config:
      <datasource_specific_key>: <datasource_specific_value>
      <datasource_specific_key>: <datasource_specific_value>
      ...
    ...
```
### Supported Datasources
#### JSON
JSON file in one of the following forms:
- one record per line, no commas
- JSON list of records
##### options:
- path - path to file containing the data, **required**
- arrayFormat - if the JSON list of records format should be used, **optional**: defaults to `false`
- batchSize - number of records extracted from json file at the same time, **optional**: defaults to `1000`

---
#### CSV
CSV file separated using commas.\
The file may or may not have column names as it's first row.
##### options:
- path - path to file containing the data, **required**
- headerRow - whether the first row of the CSV file contains column names or not, **optional**: defaults to `true`
- separator - columns separator, **optional**: defaults to `","`
- batchSize - number of records extracted from csv file at the same time, **optional**: defaults to `1000`

---
#### Excel
A single table in an Excel spreadsheet.\
The table may or may not have column names as it's first row.\
The table can be in any sheet, and start at any point, but it cannot
contain spaces between columns nor spaces between rows.
##### options:
- path - path to file, **required**
- headerRow - does the first row contain column names, **optional**: defaults to `true`
- sheet - name of the sheet in which data is stored, **optional**: defaults to `"Sheet1"`
- rootCell - name of cell (i.e "A3", "BA14") which is the leftmost cell of the first, **optional**: defaults to `"A1"`
- timeColumns - a list of columns to parse as datetime values with second precision
row, **optional**: defaults to `[]`
- batchSize - number of records extracted from excel file at the same time, **optional**: defaults to `1000`

___
#### Parquet
A single Parquet file.\
Nested repeated elements are *not supported*.\
Currently *unsupported* logical types:\
&nbsp;&nbsp;&nbsp;&nbsp; \- ENUM \
&nbsp;&nbsp;&nbsp;&nbsp; \- TIME with NANOS precision \
&nbsp;&nbsp;&nbsp;&nbsp; \- TIMESTAMP with NANOS precision (both UTC and non-UTC) \
&nbsp;&nbsp;&nbsp;&nbsp; \- INTERVAL \
&nbsp;&nbsp;&nbsp;&nbsp; \- MAP
##### options
- path - path to file, **required**
- batchSize - number of records extracted from parquet file at the same time, **optional**: defaults to `1000`

---
#### PostgreSQL
Single PostgreSQL database table.
##### options:
- address - address including port number, **optional**: defaults to `localhost:5432`
- user - **required**
- password - **required**
- databaseName - **required**
- tableName - **required**
- batchSize - number of records extracted from PostgreSQL database at the same time, **optional**: defaults to `1000`

---
#### MySQL
Single MySQL database table.
##### options:
- address - address including port number, **optional**: defaults to `localhost:3306`
- user - **required**
- password - **required**
- databaseName - **required**
- tableName - **required**
- batchSize - number of records extracted from MySQL database at the same time, **optional**: defaults to `1000`

---
#### Redis
Redis database with the given index. Currently only hashes are supported.
##### options:
- address - address including port number, **optional**: defaults to `localhost:6379`
- password - **optional**: defaults to `""`
- databaseIndex - index number of Redis database, **optional**: defaults to `0`
- databaseKeyName - column name of Redis key in OctoSQL records, **optional**: defaults to `"key"`
- batchSize - number of records extracted from Redis database at the same time, **optional**: defaults to `1000`

___
#### Kafka
## TODO
##### **optional**
- brokers - list of broker addresses (separately hosts and ports) used to connect to the kafka cluster, **optional**: defaults to `["localhost"], ["9092"]`
- topic - name of topic to read messages from, **required**
- startOffset - offset from which the first batch of messages will be read, **optional**: defaults to `-1`
- batchSize - number of records extracted from Redis database at the same time, **optional**: defaults to `1`
- json - should the messegas be decoded as JSON, **optional**: defaults to `false`

## Documentation
Documentation for the available functions: https://github.com/cube2222/octosql/wiki/Function-Documentation

Documentation for the available aggregates: https://github.com/cube2222/octosql/wiki/Aggregate-Documentation

Documentation for the available table valued functions: https://github.com/cube2222/octosql/wiki/Table-Valued-Functions-Documentation

The SQL dialect documentation: TODO ;) in short though:

Available SQL constructs: Select, Where, Order By, Group By, Offset, Limit, Left Join, Right Join, Inner Join, Distinct, Union, Union All, Subqueries, Operators, Table Valued Functions.

Available SQL types: Int, Float, String, Bool, Time, Duration, Tuple (array), Object (e.g. JSON)

## Architecture
An OctoSQL invocation gets processed in multiple phases.

### SQL AST
First, the SQL query gets parsed into an abstract syntax tree. This phase only rules out syntax errors.

### Logical Plan
The SQL AST gets converted into a logical query plan. This plan is still mostly a syntactic validation. It's the most naive possible translation of the SQL query. However, this plan already has more of a map-filter-reduce form.

If you wanted to add a new query language to OctoSQL, the only problem you'd have to solve is translating it to this logical plan.

### Physical Plan
The logical plan gets converted into a physical plan. This conversion finds any semantic errors in the query. If this phase is reached, then the input is correct and OctoSQL will be able execute it.

This phase already understands the specifics of the underlying datasources. So it's here where the optimizer will iteratively transform the plan, pushing computation nodes down to the datasources, and deduplicating unnecessary parts.

The optimizer uses a pattern matching approach, where it has rules for matching parts of the physical plan tree and how those patterns can be restructured into a more efficient version. The rules are meant to be as simple as possible and make the smallest possible changes. For example, pushing filters under maps, if they don't use any mapped variables. This way, the optimizer just keeps on iterating on the whole tree, until it can't change anything anymore. (each iteration tries to apply each rule in each possible place in the tree) This ensures that the plan reaches a local performance minimum, and the rules should be structured so that this local minimum is equal - or close to - the global minimum. (i.e. one optimization, shouldn't make another - much more useful one - impossible)

Here is an example diagram of an optimized physical plan:
![Physical Plan](images/physical.png)

### Execution Plan
The physical plan gets materialized into an execution plan. This phase has to be able to connect to the actual datasources. It may initialize connections, open files, etc.

### Stream
Starting the execution plan creates a stream, which underneath may hold more streams, or parts of the execution plan to create streams in the future. This stream works in a pull based model.

## Datasource Pushdown Operations
|Datasource	|Equality	|In	| \> < <= >=	|
|---	|---	|---	|---	|
|MySQL	|supported	|supported	|supported	|
|PostgreSQL	|supported	|supported	|supported	|
|Redis	|supported	|supported	|scan	|
|Kafka	|scan	|scan	|scan	|
|Parquet	|scan	|scan	|scan	|
|JSON	|scan	|scan	|scan	|
|CSV	|scan	|scan	|scan	|

Where `scan` means that the whole table needs to be scanned for each access.

## Roadmap
TODO - well, we need to update this big time
- Additional Datasources.
- SQL Constructs:
  - JSON Query
  - HAVING, ALL, ANY
- Parallel expression evaluation.
- Streams support (Kafka, Redis)
- Push down functions, aggregates to databases that support them.
- An in-memory index to save values of subqueries and save on rescanning tables which don't support a given operation, so as not to recalculate them each time.
- MapReduce style distributed execution mode.
- Runtime statistics
- Server mode
- Querying a json or csv table from standard input.
- Integration test suite
- Tuple splitter, returning the row for each tuple element, with the given element instead of the tuple.
- Describe-like functionality as in the diagram above.
