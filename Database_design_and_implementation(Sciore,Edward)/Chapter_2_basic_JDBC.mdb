
# Chapter 2 - Basic JDbC 

A db application interacts with a db engine through methods in its API.

Java applications use the JDBC API.

The JDBC library consists of five Java packages, most of which implement advanced features with little general application. 

The core JDBC functionality is found in the package java.sql.
- basic JDBC - classes and methods required for rudimentary usage
- advanced JDBC - optional features that provide added convenience and flexibility

## 2.1 Basic JDBC

Consists of a few interfaces

Driver
- Connection connect(String url)

Connection
- Statement createStatement() 
- void close()

Statement
- ResultSet executeQuery(String qry)
- int executeUpdate(String cmd)
- void close()

ResultSet
- boolean next()
- int getInt()
- String getString()
- void close()
- ResultSetMetaData getMetaData()

ResultSetMetaData
- int getColumnCount()
- String getColumnName(int column) 
- int getColumnType(int column) 
- int getColumnDisplaySize(int column) 

### 2.1.1 Connecting to a Database Engine

A database engine will have its own proprietary mechanism for making connections with clients. 

Clients need to be as server independent as possible.

Doesn't care about *how* to connect to a server - wants the engine to provide a class it can call  (a *driver*).

Driver classes implement the Driver interface. 

Some databases have multiple Driver implementations 
- one for server-based connections
- one for embedded connections 

A client connects to a database engine by calling a Driver object’s *connect* method.

Arguments
- Connection string: a URL that identifies the driver, the server (for server-based connections), and the database
- Properties

Properties can either be provided through a Property object (works like a hashmap) or as part of the connection string. 

Example connection string: 
"jdbc:derby://localhost/testdb;create=true";

“jdbc:derby:” describes the protocol used by the client. I.e this client is a Derby client that speaks JDBC.

“//localhost” describes the machine where the server is located; can be any domain name or IP address.

“/testdb” describes the path to the database on the server. The end of the path (here, “testdb”) is the directory where all data files for this database will be stored.

Remainder is property values Here, the substring is “create=true”, which tells the engine to create a new database. 

*Each database engine has its own connection string syntax.* 
E.g. for a database where the database is specified when the server is started, it doesnt make sense to provide one in the connection string.
Properties make no sense for a database that doesn't support properties.

The driver class and connection string syntax are vendor-specific. The rest of a JDBC program is completely vendor-neutral. 

 Apart from the name of the driver class and its connection string, a JDBC program only knows about and cares about the vendor-neutral JDBC interfaces. 

Consequently, a basic JDBC client will import 
-  The built-in package java.sql, to obtain the vendor-neutral JDBC interface definitions
- The vendor-supplied package that contains the driver class

### 2.1.2 Disconnecting from a Database Engine

While a client is connected to a database engine, the engine may allocate resources for it to use. 

E.g. a client may request locks that keep other clients from accessing portions of the database. 

Even the ability to connect to an engine can be a resource. 

The license for a commercial database system may restrict the number of simultaneous connections.  Holding a connection could prevent another client from connecting. 

To release valuable resources, clients are expected to disconnect from the engine as soon as the database is no longer needed.
Connection.close()

## 2.1.3 SQL Exceptions

Exceptions can be triggered during client/database interactions in many ways
- The client asks the engine to execute a badly formed SQL statement or an SQL query that accesses a nonexistent table/compares two incompatible values
- The engine aborts the client because of a deadlock between it and a concurrent client.
- Due to a bug in the engine code
- The client cannot access the engine (for a server-based connection)
-- wrong host name
-- host has become unreachable.

Different DB engines have their own internal way of dealing with these exceptions.
To make exception handling vendor independent, JDBC provides its own exception class, SQLException.

When a DB engine encounters an internal exception, it wraps it in an SQL exception and sends it to the client program.
The message string associated with an SQL exception identifies the internal exception that caused it. 

Each database engine is free to provide its own messages.

Most JDBC methods throw an SQL exception.

SQL exceptions are checked, which means that clients must explicitly deal with them.

A connection must be  closed when an exception is thrown or there's a resource leak—the engine cannot easily reclaim the connection’s resources after the client dies. 

## 2.1.4 Executing SQL Statements

A connection can be thought of as a “session” with the database engine, during which the engine executes the clients' SQL statements. 

JDBC supports this idea
- A Connection object has the method createStatement, returning a Statement object 
- A Statement object has two ways to execute SQL statements: 
-- executeQuery 
-- executeUpdate 
--- returns number of affected records 
-- close-method for deallocating resources held by the object

The Statement object, like the Connection object, needs to be closed. 
The easiest solution is to autoclose both objects in the try block.

## 2.1.5 Result Sets

A Statement’s executeQuery method executes an SQL query. 
- called with a string denoting an SQL query
- returns an object of type ResultSet. 

A ResultSet object represents the query’s output records. 
The client can search through the result set to examine these records.

Once a client obtains a result set, it iterates through the output records by calling *next*. 
This method moves to the next record, returning true if the move is successful and false if there are no more records.

A new ResultSet object is always positioned before the first record, and so *you need to call next before you can look at the first record.*

Resultsets tie up valuable resources on the engine. 
*close* release these resources and makes them available for other clients.

## 2.1.6 Using Query Metadata

The schema of a result set is defined as the name, type, and display size of each field.

This information is available through the *ResultSetMetaData* interface.

When a client executes a query, it usually knows the schema of the output table.

However, suppose that a client program allows users to submit queries as input.
The program can call the getMetaData on the query’s result set, which returns an object of type ResultSetMetaData. 
It can then call the methods of this object to determine the output table’s schema.

Typical use of ResultSetMetaData:

First getColumnCount is called to get the number of fields in the result set.

Then getColumnName, getColumnType, and getColumnDisplaySize to determine the name, type, and size of the field at each column. 
Note that column numbers start at 1, not 0.

getColumnType returns an integer that encodes the field type.

These codes are defined as constants in the JDBC class Types - 30 different ones. 
The actual values for these types are not important, because a JDBC program should always refer to the codes by name, not value.
A good example of a client that requires metadata knowledge is a command interpreter, e.g. SimpleIJ. 

The main method begins by reading a connection string from the user and using it to determine the proper driver to use. 

If the connection string contains “//”, then the string must be specifying a
*server-based* connection. Otherwise an embedded connection.

The method then establishes the connection by passing the connection string into the appropriate driver’s connect method.

The main method processes one line of text during each iteration of its while loop. 

If the text is an SQL statement, the method doQuery or doUpdate is called,
as appropriate.

# 2.2 Advanced JDBC

## 2.2.1 Hiding the Driver

In basic JDBC, a client connects to a database engine by calling connect on a Driver object.

This places vendor-specific code into the client program. 
JDBC provides two vendor-neutral classes for keeping driver information out of client programs:
- DriverManager
- DataSource 

DriverManager holds a collection of drivers.

Has static methods to 
- add a driver to the collection
- search the collection for a driver that can  handle a given connection string.

A client repeatedly calls registerDriver to register the driver for each database that it might use. 

When the client wants to connect to a database, it only needs to call the getConnection method and provide it with a connection string.

 The driver manager tries the connection string on each driver in its collection until one of them returns a non-null connection.
 
JDBC allows drivers to be specified in the Java system-properties file, to keep vendor specifics out of the code.

E.g.
jdbc.drivers=org.apache.derby.jdbc.ClientDriver:simpledb.remote.NetworkDriver

This file can be used to revise the driver information used by all JDBC clients without having to recompile any code.

Using DataSource

While the driver manager can hide the drivers from the JDBC clients, it cannot hide the connection string. 

The connection string in the above example contains “jdbc:derby,” so it is evident which driver is intended. 

Only later has DataSource been added to java. 
Currently the preferred strategy for managing drivers.

A DataSource object encapsulates both the driver and the connection string, thereby enabling a client to connect to an engine without knowing any connection details.

To create data sources in Derby, you need the Derby-supplied classes
- ClientDataSource (for server-based connections)
- EmbeddedDataSource (for embedded connections)

 both of which implement DataSource. 
 
 The client code might look like:
 ```
ClientDataSource ds = new ClientDataSource();
ds.setServerName("localhost");
ds.setDatabaseName("studentdb");
Connection conn = ds.getConnection();
```
Each database vendor supplies its own classes that implement DataSource.
Since these are vendor-specific, they can encapsulate the details of its driver, such as the driver name and the syntax of the connection string.
 A program that uses them only needs to specify the requisite values.

The nice thing about using a data source is that the client no longer needs to know the name of the driver or the syntax of the connection string. 
Nevertheless, the class is still vendor-specific, and so client code is thus not completely vendor independent.

This problem can be addressed in various ways.

One solution is for the database administrator to save the DataSource object in a file. 
The DBA can create the object and use Java serialization to write it to the file.

A client can then obtain the data source by reading the file and deserializing it back to a DataSource object. 

This solution is similar to using a properties file. 
Once the DataSource object is saved in the file, it can be used by any JDBC client. 
And the DBA can make changes to the data source by simply replacing the contents of that file.

A second solution is to use a name server (such as a JNDI server) instead of a file.
The DBA places the DataSource object on the name server, and clients then request the data source from the server. 
Given that name servers are a common part of many computing environments, this solution is often easy to implement.

### 2.2.2 Explicit Transaction Handling
Each JDBC client runs as a series of transactions. 

A transaction is an atomic “unit of work,”. If one update in a transaction fails, the engine ensures that all updates made by that transaction will fail.

A transaction commits when its current unit of work has completed successfully.

The engine implements a commit by making all modifications permanent and releasing any resources (e.g., locks) that were assigned to that transaction. 

Once the commit is complete, the engine starts a new transaction.

A transaction rolls back when it cannot commit. 

The database engine implements a rollback by undoing all changes made by that transaction, releasing locks, and starting a new transaction. 

A transaction that has committed or rolled back is said to have completed.

Transactions are implicit in basic JDBC. 

The database engine chooses the boundaries of each transaction, deciding when a transaction should be committed and whether it should be rolled back (autocommit)

During autocommit, the engine executes each statement in its own transaction. 

The engine commits the transaction if the statement successfully completes or
rolls back the transaction. 

- An update command completes as soon as the executeUpdate method has finished
- a query completes when the query’s result set is closed.

A transaction accrues locks, which are not released until the transaction has committed or rolled back. 

Because these locks can cause other transactions to wait, *shorter transactions enable more concurrency.* 

This implies that clients running in autocommit mode should close their result sets as soon as possible.

Autocommit is a reasonable default mode for JDBC clients. 
Having one transaction per SQL statement leads to short transactions and often is the right thing to do. 

However, there are circumstances when a transaction ought to consist of several SQL statements.

Autocommit is undesirable is when a client needs to have two statements active at the same time. 

E.g. one that first retrieves a result set and then one by one checks each against some criteria and executes a second statement to delete those that match. 
The problem is that the deletion statement will be executed while the record set is still open. 

A connection supports only one transaction at a  time, so it must preemptively commit the query’s transaction before it can create a new transaction to execute the deletion. 

Since the query’s transaction has committed, it doesn’t make sense to access the remainder of the record set. 
The code will either throw an exception or have unpredictable behavior.

Autocommit is also undesirable when multiple modifications to the database need to happen together, like teachers switching courses where teacher A teaches course 1 and teacher B teaches course 2 and A must be made responsible for 2, B for 1. 

If the engine crashes after the first call to executeUpdate but before the second one the database will be corrupted.
The statements need to happen in the same transaction. 

Autocommit mode can be inconvenient. 

Suppose that your program is performing multiple insertions, say by loading data from a text file. 

If the engine crashes while the program is running, then some of the records will be inserted and some will not.
Hard to determine where the program failed and to rewrite it to insert only the missing records. 

A better alternative is to place all the insertion commands in the same transaction. 
Then all of them would get rolled back after a system crash, and it would be possible to simply rerun the client.

The Connection interface has multiple methods that lets the client handle its transactions explicitly. 
- setAutoCommit(false) - turns off autocommit  
- commit/rollback() - ends current transaction, starts new one. 

When a client turns off autocommit, it becomes responsible for rolling back failed SQL statements.
 
In particular, if an exception is thrown during a transaction, the client exception-handling code must perform a rollback.

Code without automcommit calls setAutoCommit immediately after the connection is created; calls commit immediately after the statements have completed. 

The catch block contains the call to rollback. Needs to be placed inside its own try block, in case it throws an exception.

An exeception during rollback is no cause for concern; the DB itself will appropriately handle the situation and complete the rollback. 

## 2.2.3 Transaction Isolation Levels
A DB server typically has several active clients at the same time, each running their own transaction. 
By executing these transactions concurrently, the server can improve their throughput and response time. 
However, problems arise when several transactions interact with the same data (and at least one of them writes). 

Example 1: Reading Uncommitted Data

One transaction reads the total amount on all accounts and we have two Accounts A: 100, B: 0 and another transaction that moves the 100 from A to B, i.e. decrements A by 100, increments B by 100.

The first transaction may first read A and see 100. Before it reads B, the first transaction succeds and has now incremented B. It then sees that B has 100. 

The final total is incorrectly 200, not 100. 

Example 2: Unexpected Changes to an Existing Record

Assume that two people wants to transfer money to Account 0 with a balance of 0.

Transaction T1 wants to increment the account by 100.
Transaction T2 by 200. 

Both transactions read the current value of 0, respectively adds their amount to it and write it back to the record.

Depending on which writes last the end result is either a balance of 100 or 200. Not 300 as it should have been. 

Example 3: Unexpected Changes to the Number of Records

You have $1000 that you want to divide across all accounts of which there are 100, so you transfer $10 to each.

However, after the result set from which the count was read is closed, a new account is opened (*phantom records*)

When running the update statement for all accounts, we now select 101 to add $10 dollars to and thus we end up spending $1010, not the original 1000.
 

The only way to guarantee that an arbitrary transaction will not have problems is to execute it in complete isolation from the other transactions (*serializability*).

Serializable transactions are slow, because they require the database engine to significantly reduce the amount of concurrency it allows.

JDBC allows clients to configure one of the following isolation levels, trading speed for potential problems:

- Read-Uncommitted 
-- no isolation at all 
-- could suffer any of the problems from the above examples
- Read-Committed isolation 
-- prevents a transaction from accessing uncommitted values
-- nonrepeatable reads and phantoms are still possible
- Repeatable-Read isolation
-- extends read-committed so that reads are always repeatable
-- The only possible problems are due to phantoms
- Serializable isolation
-- guarantees that no problems will ever occur

Set through Connection.setTransactionIsolation(..)

This risk of incorrect results can be mitigated by a careful analysis of the client.

Phantoms and nonrepeatable reads will never be a problem in certain cases.  

I.e. if your transaction performs only insertions, or if it deletes specific existing records (“delete from STUDENT *where* SId=1”). 

In this case, an isolation level of read-committed will be fast and correct.

Or potential problems might not be an issue. 

Suppose that your transaction for every year calculates the average grade given .

You decide that even though grade changes can occur during the execution of the transaction, those changes are not likely to affect the resulting statistics significantly. 

You could reasonably choose the isolation level of read-committed or even read-uncommitted.

*The default isolation level for many database servers is read-committed.*

Appropriate for the simple queries posed by naïve users in autocommit mode. 

If your client programs perform critical tasks, then it is equally critical that you carefully determine the most appropriate isolation level. 

When you disable autocommit mode, you must be very careful to choose the proper isolation level of each transaction.

## 2.2.4 Prepared Statements

Many JDBC client programs are parameterized, i.e. they argument value from the user use it in an executed SQL statement

A parameterized statement is an SQL statement in which ‘?’ characters denote missing parameter values. 

A statement can have several parameters, all denoted by ‘?’.
Each parameter has an index value that corresponds to its position in the string.

The PreparedStatement class handles parameterized statements. 
How to:
- Create a PreparedStatement object for a specified parameterized SQL statement - Connection.prepareStatement(query)
- Assign values to the parameters
- Execute the prepared statement

executeQuery/executeUpdate are similar to the corresponding Statement methods, but don't require any arguments. 
setInt/setString assign values to parameters - takes an index and a value. 

*The more efficient option when statements are generated in a loop*: 

the database engine can compile a prepared statement without knowing its parameter values. 

It compiles the statement once and then executes it repeatedly inside of the loop without further recompilation.

## 2.2.5 Scrollable and Updatable Result Sets
Result sets in basic JDBC are forward-only and non-updatable.
 
*Full* JDBC allows result sets to be scrollable/updatable. 

Clients can position such result sets at arbitrary records, update the current record, and insert new records. 
- beforeFirst positions the result set before the first record
- afterLast positions the result set after the last record. 
- absolute positions the result set at exactly the specified record and returns false if there is no such record. 
- relative positions the result set a relative number of rows. 
-- relative(1) is identical to next
-- relative(-1) is identical to previous.
- updateInt/updateString modify the specified field of the current record on the client. 
-- not sent to the database until updateRow is called. 
--- allows JDBC to batch updates to several fields of a record into a single call to the engine

Insertions are done through an *insert row*. 

This row does not exist in the table (e.g., you cannot scroll to it). 

Serves as a staging area for new records. 

- The client calls moveToInsertRow to position the result set at the insert row, 
- updateXXX to set the values of its fields
- updateRow to insert the record into the database
- moveToCurrentRow to reposition the record set to where it was before the insertion.

By default, record sets are forward-only and non-updatable. 
If a client wants a more powerful record set, the createStatement method of Connection is overloaded to configure scrollability and updatability behavior. 

A scrollable result set has limited use, because most of the time the client knows what it wants to do with the output records and doesn’t need to examine them twice.
 
 A client typically needs a scrollable result set only if it allowed users to interact with the result of a query. 
 
 E.g. a client that wants to display the output of a query as a Swing JTable object.
 The JTable will display a scrollbar when there are too many output records to fit on the screen and allow the user to move back and forth through the records by clicking
on the scrollbar. 

Here the client needs to supply a scrollable result set to the JTable object, so that it can retrieve previous records when the user scrolls back.

## 2.2.6 Additional Data Types
In addition to int and strings JDBC contains methods to manipulate numerous other types. 

ResultSet has
- getFloat,
- getDouble
- getShort
- getTime
- getDate
- etc 

Reads the value from the specified field of the current record and converts it (if possible) to the indicated Java type. 

It makes most sense to convert between corresponding types, but JDBC will happily attempt to convert any SQL value to the Java type indicated by the method. 

*It is always possible to convert any SQL value to a Java string.*

## 2.3 Computing in Java vs. SQL
When implementing a JDBC client, it must be decided what computations should be performed by respectively the database engine and by the Java client.
 
Fig. 2.5. 
The engine performs all of the computation, by executing an SQL query to compute the join of the STUDENT and DEPT tables. 

```
...
String qry = "select SName, DName from DEPT, STUDENT "
ResultSet rs = stmt.executeQuery(qry)) {
...

}
```
The client only retrieves the query output and prints it.

Alternatively, the client could have done the work (Fig. 2.22)
```
...
ResultSet rs1 = stmt1.executeQuery("select * from STUDENT");
ResultSet rs2 = stmt2.executeQuery("select * from DEPARTMENT");
while (rs1.next()) {
     rs2.beforeFirst();
     while (rs2.next())

if (rs2.getInt("DId") == rs1.getInt("MajorId")) {
... 
}

``` 
The engine’s only responsibility is to create result sets for the STUDENT and DEPT tables.
The client does all the rest of the work, computing the join and printing the result.

The original version is more elegant. There's less code and its also easier to read. 
What about efficiency? 
*As a rule of thumb, it is always more efficient to do as little as possible in the client*
- Usually less data to transfer from engine to client
-- especially important if they are on different machines
- The engine contains detailed specialized knowledge about how each table is implemented and the possible ways to compute complex queries (such as joins) 
--It is highly unlikely that a client can compute a query as efficiently as the engine

Fig. 2.22 computes the join using two nested loops.

The outer loop iterates through the STUDENT records. 

For each student, the inner loop searches for the DEPT record matching that student’s major.

It is not efficient (see Ch. 13/14)

The code above exemplify extremes of really good and really bad JDBC code. 
Sometimes, it is not quite so clear if something is good or bad. 


Suppose that you know that executing a join can be time-consuming. 

After some serious thought, you realize that you can get the data you need without using a join (fig. 2.23)

The idea is to use two single-table queries. 

The first query scans the DEPT table for a record having the specified major name. 

The second query uses the department id returned from the first query to search the STUDENT records. 

```
String qry1 = "select DId from DEPT where DName = ?";
String qry2 = "select * from STUDENT where MajorId = ?";
....
PreparedStatement stmt1 = conn.prepareStatement(qry1);
stmt1.setString(1, major);
ResultSet rs1 = stmt1.executeQuery();
rs1.next();
int deptId = rs1.getInt("Did");
...
PreparedStatement stmt2 = conn.prepareStatement(qry2);
stmt2.setInt(1, deptid);
ResultSet rs2 = stmt2.executeQuery();
while (rs2.next()) {
     ...
}

```

This algorithm is simple, elegant, and efficient. 

Does a sequential scan through each of two tables and ought to be much faster than a join. 

Unfortunately, your effort is wasted. The algorithm is just a clever implementation of a join; a multibuffer product with a materialized inner table. 

A well-written database engine would already use this algorithm to compute the join if it turned out to be most efficient. 

All of your cleverness has thus been preempted by the database engine. 

*Letting the engine do the work tends to be the most efficient strategy (as well as the easiest one to code).*

Beginning JDBC programmers often try to do too much in the client.
- might think that he knows a clever way to implement a query in Java. 
- might not be sure how to express a query in SQL and feels more comfortable coding the query in Java. 

*The decision to code the query in Java is almost always wrong.*

The programmer must trust that the database engine will do its job.