# Ch. 2 - Jaywalking

Using comma-separated lists instead of creating an intersection table for a many-to-many relationship. 

## Objective: Store Multivalue Attributes
E.g. evolving a product from having a single contact (refered to as Account) to multiple

### Antipattern: Format Comma-Separated Lists
```
CREATE TABLE Products (
account_id VARCHAR(100), -- comma-separated list
...
)
INSERT INTO Products (product_id, product_name, account_id)
VALUES (DEFAULT, 'Visual TurboBuilder', '12,34');
```

Pro:  no additional tables or columns; only changing the data type of one column
Con: performance and data integrity problems 

### Querying Products for a Specific Account

When all the foreign keys are combined into a single field you can't use equality in queries. 
Instead, you must test against some kind of pattern:
```
SELECT * FROM Products WHERE account_id REGEXP '\\b12\\b';
```
Cons
- Pattern-matching expressions may return false matches. 
- Matching can’t benefit from indexes => bad performance. 
- Lacks vendor neutrality; Pattern-matching syntax is different in each database brandIt 

### Querying Accounts for a Given Product

Awkward and slow to join a comma-separated list to matching rows in referenced table, e.g. query accounts table based on list from Products:
```
SELECT * FROM Products AS p JOIN Accounts AS a ON p.account_id REGEXP '\\b' || a.account_id || '\\b' 
WHERE p.product_id = 123;
```
Query must scan through both tables, generate a cross product, and evaluate the regular expression for every combination of rows.
Join can't make use of indexes. 

### Making Aggregate Queries
Aggregate functions work on groups of rows
To do something like a COUNT, you have to resort to tricks, like calculating the length of the string of comma-separated values minus the length of that string with the commas removed. 

Cons
- Time consuming to re-invent aggregate functions 
- Hard to understand and debug 
- Not all aggregate functions can be expressed alternatively 

### Updating Accounts for a Specific Product
Add:
Concatenate to the existing value 

Delete:
You have to run two SQL queries: 
- fetch the old list and remove the value from it
- save the updated list.


### Validating Product IDs
Since the list is just a string, users are free to concatenate invalid nonsense.
Even if we are concatenating a valid id, we have no way to check that it exists in the Account table as foreign key constraint only works for the whole column value; not a list element. 

#### Choosing a Separator Character
If using a list of strings, it cannot be prevented that an appended string contains the same character as is used for seperator.

#### List Length Limitations
A VARCHAR(x) type must be used for the column, but what value to set x to? How can we be sure that we're never gonna need to add more values than x allows? 
 
 
## How to Recognize the Antipattern

When people ask questions like
- “What is the greatest number of entries this list must support?”
- “Do you know how to match a word boundary in SQL?”
- “What character will never appear in any list entry?”

## Legitimate Uses of the Antipattern
The performance of certain queries can be improved by applying denormalization to your database. 
Storing lists as a comma-separated string is an example of denormalization.

There's no need to seperate values when the application:
-  may need the data in a comma-separated format and never need to access individual items in the list. 
- receives a comma-separated format from another source and you simply need to store the full list and retrieve it later in exactly the same format

Be conservative if you decide to employ denormalization: 
*Start by using a normalized database organization, because it permits your application code to be more flexible, and it allows your database to help preserve data integrity.*

Some brands of SQL database products extend SQL data types with some array types that may mitigate some of the issues.
E.g. you can specify the scalar data type of the array elements. 
Complex to use, and implemented in different ways in different database brands.  

## Solution: Create an Intersection Table
Store each Account id in a separate table to implement a many-to-many relationship between Product and Account. 
A table with foreign keys referencing two tables is called an *intersection* table (join table/many-to-many table/mapping table)
An intersection table resolves all the described problems.

### Querying Products by Account and the Other Way Around
Straightforward to query in a way that makes use of indexes: 
```
SELECT p.*
FROM
Products AS p JOIN Contacts AS c ON (p.product_id = c.product_id)
WHERE c.account_id = 34;

SELECT a.*
FROM
Accounts AS a JOIN Contacts AS c ON (a.account_id = c.account_id)
WHERE c.product_id = 123;
```


### Making Aggregate Queries

Aggregate queries are straightforward:
```
SELECT product_id, COUNT(*) AS accounts_per_product
FROM Contacts
GROUP BY product_id;

SELECT account_id, COUNT(*) AS products_per_account
FROM Contacts
GROUP BY account_id;
```

### Updating Contacts for a Specific Product

You add/remove entries by inserting/deleting rows in the intersection table. 

### Validating Product IDs

By using foreign keys for ids you rely on the database to enforce referential integrity; the intersection table will only contain account IDs that exist 
The type of the column can potentially restrict the valid values of the account ids, i.e. nonsense values are not possible (but obviously VARCHAR permits all kinds of things)

### Choosing a Separator Character
Not relevant since the seperation happens by being stored in different rows.

### List Length Limitations
As each entry is in a separate row, the "list" is limited only by the number of rows that can physically exist in a table. 
If the number of entries should be limited, it should be enforced on the application level. 

### Other Advantages of the Intersection Table

An index on Contacts.account_id makes performance better than matching a substring in a comma-separated list. 
In many database brands, declaring a foreign key on a column implicitly creates an index on that column.

You can create additional attributes for each entry by adding columns to the intersection table.  E.g. you could record the date a contact was added for a given product or an attribute noting who is the primary contact vs. the secondary contacts. 
You can’t do this in a comma-separated list.

### Mini-Antipattern: Splitting CSV Into Rows

You have a situation where data is stored as a list in a column where you want to respond to a query as if the data was in individual rows and it is not possible to migrate the data into actual rows. 

Some brands of SQL have a function to for this. 
E.g. PostgreSQL has *string_to_array()* that converts the comma-separated string into an array type that can then be input to  *unnest()* that expands an array into rows. 
Alternatively, the list can be joined with a predefined set of integers, one integer per row. 
For each of the resulting joined rows, use a substring expression to extract the Nth element from the comma-separated list
However, you need to get the integers from somewhere
- Dedicated table with integer series 
- Generate series dynamically w. UNION

Solutions may only work in certain brands/versions of SQL databases.

*The best solution, which works in any database, is to store data the way you need to use it.*