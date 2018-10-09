## openCypher Query Language

[*openCypher*](http://www.opencypher.org/) is a query language for querying
graph databases. It aims to be intuitive and easy to learn, while
providing a powerful interface for working with graph based data.

*Memgraph* supports most of the commonly used constructs of the language. This
chapter contains the details of implemented features. Additionally,
not yet supported features of the language are listed.

  * [Reading Existing Data](#reading-existing-data)
  * [Writing New Data](#writing-new-data)
  * [Reading and Writing](#reading-and-writing)
  * [Indexing](#indexing)
  * [Other Features](#other-features)

### Reading Existing Data

The simplest usage of the language is to find data stored in the
database. For that purpose, the following clauses are offered:

  * `MATCH`, which searches for patterns;
  * `WHERE`, for filtering the matched data and
  * `RETURN`, for defining what will be presented to the user in the result
    set.
  * `UNION` and `UNION ALL` for combining results from multiple queries.

#### MATCH

This clause is used to obtain data from Memgraph by matching it to a given
pattern. For example, to find each node in the database, you can use the
following query.

```opencypher
MATCH (node) RETURN node
```

Finding connected nodes can be achieved by using the query:

```opencypher
MATCH (node1)-[connection]-(node2) RETURN node1, connection, node2
```

In addition to general pattern matching, you can narrow the search down by
specifying node labels and properties. Similarly, edge types and properties
can also be specified. For example, finding each node labeled as `Person` and
with property `age` being 42, is done with the following query.

```opencypher
MATCH (n :Person {age: 42}) RETURN n
```

While their friends can be found with the following.

```opencypher
MATCH (n :Person {age: 42})-[:FriendOf]-(friend) RETURN friend
```

There are cases when a user needs to find data which is connected by
traversing a path of connections, but the user doesn't know how many
connections need to be traversed. openCypher allows for designating patterns
with *variable path lengths*. Matching such a path is achieved by using the
`*` (*asterisk*) symbol inside the edge element of a pattern. For example,
traversing from `node1` to `node2` by following any number of connections in a
single direction can be achieved with:

```opencypher
MATCH (node1)-[r*]->(node2) RETURN node1, r, node2
```

If paths are very long, finding them could take a long time. To prevent that,
a user can provide the minimum and maximum length of the path. For example,
paths of length between 2 and 4 can be obtained with a query like:

```opencypher
MATCH (node1)-[r*2..4]->(node2) RETURN node1, r, node2
```

It is possible to name patterns in the query and return the resulting paths.
This is especially useful when matching variable length paths:

```opencypher
MATCH path = ()-[r*2..4]->() RETURN path
```

More details on how `MATCH` works can be found
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/match/).

The `MATCH` clause can be modified by prepending the `OPTIONAL` keyword.
`OPTIONAL MATCH` clause behaves the same as a regular `MATCH`, but when it
fails to find the pattern, missing parts of the pattern will be filled with
`null` values. Examples can be found
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/optional-match/).

#### WHERE

You have already seen that simple filtering can be achieved by using labels
and properties in `MATCH` patterns. When more complex filtering is desired,
you can use `WHERE` paired with `MATCH` or `OPTIONAL MATCH`. For example,
finding each person older than 20 is done with the this query.

```opencypher
MATCH (n :Person) WHERE n.age > 20 RETURN n
```

Additional examples can be found
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/where/).

#### RETURN

The `RETURN` clause defines which data should be included in the resulting
set. Basic usage was already shown in the examples for `MATCH` and `WHERE`
clauses. Another feature of `RETURN` is renaming the results using the `AS`
keyword.

Example.

```opencypher
MATCH (n :Person) RETURN n AS people
```

That query would display all nodes under the header named `people` instead of
`n`.

When you want to get everything that was matched, you can use the `*`
(*asterisk*) symbol.

This query:

```opencypher
MATCH (node1)-[connection]-(node2) RETURN *
```

is equivalent to:

```opencypher
MATCH (node1)-[connection]-(node2) RETURN node1, connection, node2
```

`RETURN` can be followed by the `DISTINCT` operator, which will remove
duplicate results. For example, getting unique names of people can be achieved
with:

```opencypher
MATCH (n :Person) RETURN DISTINCT n.name
```

Besides choosing what will be the result and how it will be named, the
`RETURN` clause can also be used to:

  * limit results with `LIMIT` sub-clause;
  * skip results with `SKIP` sub-clause;
  * order results with `ORDER BY` sub-clause and
  * perform aggregations (such as `count`).

More details on `RETURN` can be found
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/return/).

##### SKIP & LIMIT

These sub-clauses take a number of how many results to skip or limit.
For example, to get the first 3 results you can use this query.

```opencypher
MATCH (n :Person) RETURN n LIMIT 3
```

If you want to get all the results after the first 3, you can use the
following.

```opencypher
MATCH (n :Person) RETURN n SKIP 3
```

The `SKIP` and `LIMIT` can be combined. So for example, to get the 2nd result,
you can do:

```opencypher
MATCH (n :Person) RETURN n SKIP 1 LIMIT 1
```

##### ORDER BY

Since the patterns which are matched can come in any order, it is very useful
to be able to enforce some ordering among the results. In such cases, you can
use the `ORDER BY` sub-clause.

For example, the following query will get all `:Person` nodes and order them
by their names.

```opencypher
MATCH (n :Person) RETURN n ORDER BY n.name
```

By default, ordering will be in the ascending order. To change the order to be
descending, you should append `DESC`.

For example, to order people by their name descending, you can use this query.

```opencypher
MATCH (n :Person) RETURN n ORDER BY n.name DESC
```

You can also order by multiple variables. The results will be sorted by the
first variable listed. If the values are equal, the results are sorted by the
second variable, and so on.

Example. Ordering by first name descending and last name ascending.

```opencypher
MATCH (n :Person) RETURN n ORDER BY n.name DESC, n.lastName
```

Note that `ORDER BY` sees only the variable names as carried over by `RETURN`.
This means that the following will result in an error.

```opencypher
MATCH (n :Person) RETURN old AS new ORDER BY old.name
```

Instead, the `new` variable must be used:

```opencypher
MATCH (n: Person) RETURN old AS new ORDER BY new.name
```

The `ORDER BY` sub-clause may come in handy with `SKIP` and/or `LIMIT`
sub-clauses. For example, to get the oldest person you can use the following.

```opencypher
MATCH (n :Person) RETURN n ORDER BY n.age DESC LIMIT 1
```

##### Aggregating

openCypher has functions for aggregating data. Memgraph currently supports
the following aggregating functions.

  * `avg`, for calculating the average.
  * `collect`, for collecting multiple values into a single list or map. If given a single expression values are collected into a list. If given two expressions, values are collected into a map where the first expression denotes map keys (must be string values) and the second expression denotes map values.
  * `count`, for counting the resulting values.
  * `max`, for calculating the maximum result.
  * `min`, for calculating the minimum result.
  * `sum`, for getting the sum of numeric results.

Example, calculating the average age:

```opencypher
MATCH (n :Person) RETURN avg(n.age) AS averageAge
```

Collecting items into a list:

```opencypher
MATCH (n :Person) RETURN collect(n.name) AS list_of_names
```

Collecting items into a map:

```opencypher
MATCH (n :Person) RETURN collect(n.name, n.age) AS map_name_to_age
```

Click
[here](https://neo4j.com/docs/developer-manual/current/cypher/functions/aggregating/)
for additional details on how aggregations work.

#### UNION and UNION ALL

openCypher supports combining results from multiple queries into a single result
set. That result will contain rows that belong to queries in the union
respecting the union type.

Using `UNION` will contain only distinct rows while `UNION ALL` will keep all
rows from all given queries.

Restrictions when using `UNION` or `UNION ALL`:
  * The number and the names of columns returned by queries must be the same
    for all of them.
  * There can be only one union type between single queries, i.e. a query can't
    contain both `UNION` and `UNION ALL`.

Example, get distinct names that are shared between persons and movies:

```opencypher
MATCH(n: Person) RETURN n.name AS name UNION MATCH(n: Movie) RETURN n.name AS name
```

Example, get all names that are shared between persons and movies (including duplicates):

```opencypher
MATCH(n: Person) RETURN n.name AS name UNION ALL MATCH(n: Movie) RETURN n.name AS name
```

### Writing New Data

For adding new data, you can use the following clauses.

  * `CREATE`, for creating new nodes and edges.
  * `SET`, for adding new or updating existing labels and properties.
  * `DELETE`, for deleting nodes and edges.
  * `REMOVE`, for removing labels and properties.

You can still use the `RETURN` clause to produce results after writing, but it
is not mandatory.

Details on which kind of data can be stored in *Memgraph* can be found in
[Data Storage](storage.md) chapter.

#### CREATE

This clause is used to add new nodes and edges to the database. The creation
is done by providing a pattern, similarly to `MATCH` clause.

For example, to create 2 new nodes connected with a new edge, use this query.

```opencypher
CREATE (node1)-[:edge_type]->(node2)
```

Labels and properties can be set during creation using the same syntax as in
[MATCH](#match) patterns. For example, creating a node with a label and a
property:

```opencypher
CREATE (node :Label {property: "my property value"}
```

Additional information on `CREATE` is
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/create/).

#### SET

The `SET` clause is used to update labels and properties of already existing
data.

Example. Incrementing everyone's age by 1.

```opencypher
MATCH (n :Person) SET n.age = n.age + 1
```

Click
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/create/)
for a more detailed explanation on what can be done with `SET`.

#### DELETE

This clause is used to delete nodes and edges from the database.

Example. Removing all edges of a single type.

```opencypher
MATCH ()-[edge :type]-() DELETE edge
```

When testing the database, you want to often have a clean start by deleting
every node and edge in the database. It is reasonable that deleting each node
should delete all edges coming into or out of that node.

```opencypher
MATCH (node) DELETE node
```

But, openCypher prevents accidental deletion of edges. Therefore, the above
query will report an error. Instead, you need to use the `DETACH` keyword,
which will remove edges from a node you are deleting. The following should
work and *delete everything* in the database.

```opencypher
MATCH (node) DETACH DELETE node
```

More examples are
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/delete/).

#### REMOVE

The `REMOVE` clause is used to remove labels and properties from nodes and
edges.

Example.

```opencypher
MATCH (n :WrongLabel) REMOVE n :WrongLabel, n.property
```

### Reading and Writing

OpenCypher supports combining multiple reads and writes using the
`WITH` clause. In addition to combining, the `MERGE` clause is provided which
may create patterns if they do not exist.

#### WITH

The write part of the query cannot be simply followed by another read part. In
order to combine them, `WITH` clause must be used. The names this clause
establishes are transferred from one part to another.

For example, creating a node and finding all nodes with the same property.

```opencypher
CREATE (node {property: 42}) WITH node.property AS propValue
MATCH (n {property: propValue}) RETURN n
```

Note that the `node` is not visible after `WITH`, since only `node.property`
was carried over.

This clause behaves very much like `RETURN`, so you should refer to features
of `RETURN`.

#### MERGE

The `MERGE` clause is used to ensure that a pattern you are looking for exists
in the database. This means that if the pattern is not found, it will be
created. In a way, this clause is like a combination of `MATCH` and `CREATE`.


Example. Ensure that a person has at least one friend.

```opencypher
MATCH (n :Person) MERGE (n)-[:FriendOf]->(m)
```

The clause also provides additional features for updating the values depending
on whether the pattern was created or matched. This is achieved with `ON
CREATE` and `ON MATCH` sub clauses.

Example. Set a different properties depending on what `MERGE` did.

```opencypher
MATCH (n :Person) MERGE (n)-[:FriendOf]->(m)
ON CREATE SET m.prop = "created" ON MATCH SET m.prop = "existed"
```

For more details, click [this
link](https://neo4j.com/docs/developer-manual/current/cypher/clauses/merge/).

### Indexing

An index stores additional information on certain types of data, so that
retrieving said data becomes more efficient. Downsides of indexing are:

  * requiring extra storage for each index and
  * slowing down writes to the database.

Carefully choosing which data to index can tremendously improve data retrieval
efficiency, and thus make index downsides negligible.

Memgraph automatically indexes labeled data. This improves queries
which fetch nodes by label:

```opencypher
MATCH (n :Label) ... RETURN n
```

Indexing can also be applied to data with a specific combination of label and
property. These are not automatically created, instead a user needs to create
them explicitly. Creation is done using a special
`CREATE INDEX ON :Label(property)` language construct.

For example, to index nodes which is labeled as `:Person` and has a property
named `age`:

```opencypher
CREATE INDEX ON :Person(age)
```

After the index is created, retrieving those nodes will become more efficient.
For example, the following query will retrieve all nodes which have an `age`
property, instead of fetching each `:Person` node and checking whether the
property exists.

```opencypher
MATCH (n :Person {age: 42}) RETURN n
```

Using index based retrieval also works when filtering labels and properties
with `WHERE`. For example, the same effect as in the previous example can be
done with:

```opencypher
MATCH (n) WHERE n:Person AND n.age = 42 RETURN n
```

Since the filter inside `WHERE` can contain any kind of an expression, the
expression can be complicated enough so that the index does not get used. We
are continuously improving the recognition of index usage opportunities from a
`WHERE` expression. If there is any suspicion that an index may not be used,
we recommend putting properties and labels inside the `MATCH` pattern.

Currently, once an index is created it cannot be deleted. This feature will be
implemented very soon. The expected syntax for removing an index will be `DROP
INDEX ON :Label(property)`.

### Other Features

The following sections describe some of the other supported features.

#### Filtering Variable Length Paths

OpenCypher supports only simple filtering when matching variable length paths.
For example:

```opencypher
MATCH (n)-[edge_list:Type * {x: 42}]-(m)
```

This will produce only those paths whose edges have the required `Type` and `x`
property value. Edges that compose the produced paths are stored in a symbol
named `edge_list`. Naturally, the user could have specified any other symbol
name.

Memgraph extends openCypher with a syntax for arbitrary filter expressions
during path matching. The next example filters edges which have property `x`
between `0` and `10`.

```opencypher
MATCH (n)-[edge_list * (edge, node | 0 < edge.x < 10)]-(m)
```

Here we introduce a lambda function with parentheses, where the first two
arguments, `edge` and `node`, correspond to each edge and node during path
matching. `node` is the destination node we are moving to across the current
`edge`. The last `node` value will be the same value as `m`. Following the
pipe (`|`) character is an arbitrary expression which must produce a boolean
value.  If `True`, matching continues, otherwise the path is discarded.

The previous example can be written using the `all` function:

```opencypher
MATCH (n)-[edge_list *]-(m) WHERE all(edge IN r WHERE 0 < edge.x < 10)
```

However, filtering using a lambda function is more efficient because paths
may be discarded earlier in the traversal. Furthermore, it provides more
flexibility for deciding what kind of paths are matched due to more expressive
filtering capabilities. Therefore, filtering through lambda functions should
be preferred whenever possible.

#### Breadth First Search

A typical graph use-case is searching for the shortest path between nodes.
The openCypher standard does not define this feature, so Memgraph provides
a custom implementation, based on the edge expansion syntax.

Finding the shortest path between nodes can be done using breadth-first
expansion:

```opencypher
MATCH (a {id: 723})-[edge_list:Type *bfs..10]-(b {id: 882}) RETURN *
```

The above query will find all paths of length up to 10 between nodes `a` and `b`.
The edge type and maximum path length are used in the same way like in variable
length expansion.

To find only the shortest path, simply append `LIMIT 1` to the `RETURN` clause.

```opencypher
MATCH (a {id: 723})-[edge_list:Type *bfs..10]-(b {id: 882}) RETURN * LIMIT 1
```

Breadth-first expansion allows an arbitrary expression filter that determines
if an expansion is allowed. Following is an example in which expansion is
allowed only over edges whose `x` property is greater than `12` and nodes `y`
whose property is less than `3`:

```opencypher
MATCH (a {id: 723})-[*bfs..10 (e, n | e.x > 12 AND n.y < 3)]-() RETURN *
```

The filter is defined as a lambda function over `e` and `n`, which denote the edge
and node being expanded over in the breadth first search. Note that if the user
omits the edge list symbol (`edge_list` in previous examples) it will not be included
in the result.

There are a few benefits of the breadth-first expansion approach, as opposed to
a specialized `shortestPath` function. For one, it is possible to inject
expressions that filter on nodes and edges along the path itself, not just the final
destination node. Furthermore, it's possible to find multiple paths to multiple destination
nodes regardless of their length. Also, it is possible to simply go through a node's
neighbourhood in breadth-first manner.

Currently, it isn't possible to get all shortest paths to a single node using
Memgraph's breadth-first expansion.

#### Weighted Shortest Path

Another standard use-case in a graph is searching for the weighted shortest
path between nodes. The openCypher standard does not define this feature, so
Memgraph provides a custom implementation, based on the edge expansion syntax.

Finding the weighted shortest path between nodes is done using the weighted
shortest path expansion:

```opencypher
MATCH (a {id: 723})-[
        edge_list *wShortest 10 (e, n | e.weight) total_weight
    ]-(b {id: 882})
RETURN *
```

The above query will find the shortest path of length up to 10 nodes between
nodes `a`  and `b`. The length restriction parameter is optional.

Weighted Shortest Path expansion allows an arbitrary expression that determines
the weight for the current expansion. Total weight of a path is calculated as
the sum of all weights on the path between two nodes. Following is an example in
which the weight between nodes is defined as the product of edge weights
(instead of sum), assuming all weights are greater than '1':

```opencypher
MATCH (a {id: 723})-[
        edge_list *wShortest 10 (e, n | log(e.weight)) total_weight
    ]-(b {id: 882})
RETURN exp(total_weight)
```

Weighted Shortest Path expansions also allows an arbitrary expression filter
that determines if an expansion is allowed. Following is an example in which
expansion is allowed only over edges whose `x` property is greater than `12`
and nodes `y` whose property is less than `3`:

```opencypher
MATCH (a {id: 723})-[
        edge_list *wShortest 10 (e, n | e.weight) total_weight (e, n | e.x > 12 AND n.y < 3)
    ]-(b {id: 882})
RETURN exp(total_weight)
```

Both weight and filter expression are defined as lambda functions over `e` and
`n`, which denote the edge and the node being expanded over in the weighted
shortest path search.

#### UNWIND

The `UNWIND` clause is used to unwind a list of values as individual rows.

Example. Produce rows out of a single list.

```opencypher
UNWIND [1,2,3] AS listElement RETURN listElement
```

More examples are
[here](https://neo4j.com/docs/developer-manual/current/cypher/clauses/unwind/).

#### Functions

You have already been introduced to one type of functions, [aggregating
functions](#aggregating). This section contains the list of other supported
functions.

 Name            | Description
-----------------|------------
 `coalesce`      | Returns the first non null argument.
 `startNode`     | Returns the starting node of an edge.
 `endNode`       | Returns the destination node of an edge.
 `degree`        | Returns the number of edges (both incoming and outgoing) of a node.
 `head`          | Returns the first element of a list.
 `last`          | Returns the last element of a list.
 `properties`    | Returns the properties of a node or an edge.
 `size`          | Returns the number of elements in a list or a map. When given a string it returns the number of characters. When given a path it returns the number of expansions (edges) in that path.
 `toBoolean`     | Converts the argument to a boolean.
 `toFloat`       | Converts the argument to a floating point number.
 `toInteger`     | Converts the argument to an integer.
 `type`          | Returns the type of an edge as a character string.
 `keys`          | Returns a list keys of properties from an edge or a node. Each key is represented as a string of characters.
 `labels`        | Returns a list of labels from a node. Each label is represented as a character string.
 `nodes`         | Returns a list of nodes from a path.
 `relationships` | Returns a list of relationships from a path.
 `range`         | Constructs a list of value in given range.
 `tail`          | Returns all elements after the first of a given list.
 `abs`           | Returns the absolute value of a number.
 `ceil`          | Returns the smallest integer greater than or equal to given number.
 `floor`         | Returns the largest integer smaller than or equal to given number.
 `round`         | Returns the number, rounded to the nearest integer. Tie-breaking is done using the *commercial rounding*,  where -1.5 produces -2 and 1.5 produces 2.
 `exp`           | Calculates `e^n` where `e` is the base of the natural logarithm, and `n` is the given number.
 `log`           | Calculates the natural logarithm of a given number.
 `log10`         | Calculates the logarithm (base 10) of a given number.
 `sqrt`          | Calculates the square root of a given number.
 `acos`          | Calculates the arccosine of a given number.
 `asin`          | Calculates the arcsine of a given number.
 `atan`          | Calculates the arctangent of a given number.
 `atan2`         | Calculates the arctangent2 of a given number.
 `cos`           | Calculates the cosine of a given number.
 `sin`           | Calculates the sine of a given number.
 `tan`           | Calculates the tangent of a given number.
 `sign`          | Applies the signum function to a given number and returns the result. The signum of positive numbers is 1, of negative -1 and for 0 returns 0.
 `e`             | Returns the base of the natural logarithm.
 `pi`            | Returns the constant *pi*.
 `rand`          | Returns a random floating point number between 0 (inclusive) and 1 (exclusive).
 `startsWith`    | Check if the first argument starts with the second.
 `endsWith`      | Check if the first argument ends with the second.
 `contains`      | Check if the first argument has an element which is equal to the second argument.
 `left`          | Returns a string containing the specified number of leftmost characters of the original string.
 `lTrim`         | Returns the original string with leading whitespace removed.
 `replace`       | Returns a string in which all occurrences of a specified string in the original string have been replaced by another (specified) string.
 `reverse`       | Returns a string in which the order of all characters in the original string have been reversed.
 `right`         | Returns a string containing the specified number of rightmost characters of the original string.
 `rTrim`         | Returns the original string with trailing whitespace removed.
 `split`         | Returns a list of strings resulting from the splitting of the original string around matches of the given delimiter.
 `substring`     | Returns a substring of the original string, beginning with a 0-based index start and length.
 `toLower`       | Returns the original string in lowercase.
 `toString`      | Converts an integer, float or boolean value to a string.
 `toUpper`       | Returns the original string in uppercase.
 `trim`          | Returns the original string with leading and trailing whitespace removed.
 `all`           | Check if all elements of a list satisfy a predicate.<br/>The syntax is: `all(variable IN list WHERE predicate)`.<br/> NOTE: Whenever possible, use Memgraph's lambda functions when [matching](#filtering-variable-length-paths) instead.
 `single`        | Check if only one element of a list satisfies a predicate.<br/>The syntax is: `single(variable IN list WHERE predicate)`.
 `reduce`        | Accumulate list elements into a single result by applying an expression. The syntax is:<br/>`reduce(accumulator = initial_value, variable IN list | expression)`.
 `extract`       | A list of values obtained by evaluating an expression for each element in list. The syntax is:<br>`extract(variable IN list | expression)`.
 `assert`        | Raises an exception reported to the client if the given argument is not `true`.
 `counter`       | Generates integers that are guaranteed to be unique on the database level, for the given counter name.
 `counterSet`    | Sets the counter with the given name to the given value.
 `indexInfo`     | Returns a list of all the indexes available in the database. The list includes indexes that are not yet ready for use (they are concurrently being built by another transaction).
 `timestamp`     | Returns the difference, measured in milliseconds, between the current time and midnight, January 1, 1970 UTC.
 `id`            | Returns identifier for a given node or edge. The identifier is generated during the initialization of node or edge and will be persisted through the durability mechanism.

#### String Operators

Apart from comparison and concatenation operators openCypher provides special
string operators for easier matching of substrings:

Operator           | Description
-------------------|------------
 `a STARTS WITH b` | Returns true if prefix of string a is equal to string b.
 `a ENDS WITH b`   | Returns true if suffix of string a is equal to string b.
 `a CONTAINS b`    | Returns true if some substring of string a is equal to string b.

#### Parameters

When automating the queries for Memgraph, it comes in handy to change only
some parts of the query. Usually, these parts are values which are used for
filtering results or similar, while the rest of the query remains the same.

Parameters allow reusing the same query, but with different parameter values.
The syntax uses the `$` symbol to designate a parameter name. We don't allow
old Cypher parameter syntax using curly braces. For example, you can parameterize
filtering a node property:

```opencypher
MATCH (node1 {property: $propertyValue}) RETURN node1
```

You can use parameters instead of any literal in the query, but not instead of
property maps even though that is allowed in standard openCypher. Following
example is illegal in Memgraph:

```opencypher
MATCH (node1 $propertyValue) RETURN node1
```

To use parameters with Python driver use following syntax:

```python
session.run('CREATE (alice:Person {name: $name, age: $ageValue}',
            name='Alice', ageValue=22)).consume()
```

To use parameters which names are integers you will need to wrap parameters in
a dictionary and convert them to strings before running a query:

```python
session.run('CREATE (alice:Person {name: $0, age: $1}',
            {'0': "Alice", '1': 22})).consume()
```

To use parameters with some other driver please consult appropriate
documentation.

#### CASE

Conditional expressions can be expressed in openCypher language by simple and
generic form of `CASE` expression. A simple form is used to compare an expression
against multiple predicates. For the first matched predicate result of the
expression provided after the `THEN` keyword is returned.  If no expression is
matched value following `ELSE` is returned is provided, or `null` if `ELSE` is not
used:

```opencypher
MATCH (n)
RETURN CASE n.currency WHEN "DOLLAR" THEN "$" WHEN "EURO" THEN "€" ELSE "UNKNOWN" END
```

In generic form, you don't need to provide an expression whose value is compared to
predicates, but you can list multiple predicates and the first one that evaluates
to true is matched:

```opencypher
MATCH (n)
RETURN CASE WHEN n.height < 30 THEN "short" WHEN n.height > 300 THEN "tall" END
```

### Differences

Although we try to implement openCypher query language as closely to the
language reference as possible, we had to make some changes to enhance the
user experience.

#### Unicode Codepoints in String Literal

Use `\u` followed by 4 hex digits in string literal for UTF-16 codepoint and
`\U` with 8 hex digits for UTF-32 codepoint in Memgraph.


### Difference from Neo4j's Cypher Implementation

The openCypher initiative stems from Neo4j's Cypher query language. Following is a list
of most important differences between Neo's Cypher and Memgraph's openCypher implementation,
for users that are already familiar with Neo4j. There might be other differences not documented
here (especially subtle semantic ones).

#### Unsupported Constructs

* Data importing. Memgraph doesn't support Cypher's CSV importing capabilities.
* The `FOREACH` language construct for performing an operation on every list element.
* The `CALL` construct for a standalone function call. This can be expressed using
  `RETURN functioncall()`. For example, with Memgraph you can get information about
  the indexes present in the database using the `RETURN indexinfo()` openCypher query.
* Stored procedures.
* Regular expressions for string matching.
* `shortestPath` and `allShortestPaths` functions. `shortestPath` can be expressed using
  Memgraph's breadth-first expansion syntax already described in this document.
* Patterns in expressions. For example, Memgraph doesn't support `size((n)-->())`. Most of the time
  the same functionalities can be expressed differently in Memgraph using `OPTIONAL` expansions,
  function calls etc.
* Map projections such as `MATCH (n) RETURN n {.property1, .property2}`.

#### Unsupported Functions

General purpose functions:

* `exists(n.property)` - This can be expressed using `n.property IS NOT NULL`.
* `length()` is named `size()` in Memgraph.

Aggregation functions:

* `count(DISTINCT variable)` - This can be expressed using `WITH DISTINCT variable RETURN count(variable)`.

Mathematical functions:

* `percentileDisc()`
* `stDev()`
* `point()`
* `distance()`
* `degrees()`

List functions:

* `any()`
* `none()`
