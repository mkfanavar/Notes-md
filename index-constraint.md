# Indexes and Constraints in Neo4j

### Type of constraints

- uniqueness of property key values: a property must be unique for that node and label
- existence of property key values: a property must have a value
- node key: existence and uniqueness for a property key value

### Type of Indexes

- RANGE: for nodes and relations
- LOOKUP: create by engine for nodes and relations by they label and types
- TEXT
- POINT
- FULL-TEXT

# Using Constraints

constraints are internally implemented as indexes

![e](.\images\image-20231015092218581.png)

- uniqueness constraints
  - best practice: Define a uniqueness constraints for every node label
- node key constraints:
  - property or properties are unique for a node with that label
  - property of properties exist for a node with that label

## Creating Uniqueness Constraints

```cypher
// create uniquness constraint
CREATE CONSTRAINT <constraint_name> IF NOT EXISTS
FOR (x:<node_label>)
REQUIRE x.<property_key> IS UNIQUE
         
// for example
create constraint tmbdid_uniquness  if not exists
for (x:Person)
require x.tmdbId is unique
     
//creating a uniqueness constraint for multiple properties
CREATE CONSTRAINT <constraint_name> IF NOT EXISTS
FOR (x:<node_label>)
REQUIRE (x.<property_key1>, x.<property_key2>)  IS UNIQUE

// the combination of title and relese date of movies should be unique
CREATE CONSTRAINT uniqe_title_release IF NOT EXISTS
FOR (x:Movie)
REQUIRE (x.release, x.title) IS UNIQUE

// show all constraints
SHOW CONSTRAINTS

// uniqueness in node creation
// if we have a movie with movieId of '1'
// then this code do nothing.
MERGE (m:Movie {movieId: '1'})


// Uniqueness constraint creation failure
// this will fail becuase current content break defined constraint
CREATE CONSTRAINT Movie_year_unique IF NOT EXISTS
FOR (x:Movie)
REQUIRE x.year IS UNIQUE
```

## Creating Existence Constraints on nodes properties (only for enterprise edition)

```cypher
CREATE CONSTRAINT <constraint_name> IF NOT EXISTS
FOR (x:<label>)
REQUIRE x.<property> is not null
```

## Creating Existence Constraints on Relationship Properties (only for enterprise edition)

```cypher
CREATE CONSTRAINT <constraint_name> IF NO EXISTS
FOR ()-[x:<relationship_type>]-()
REQUIRE x.<property_key> is not null
```

## Creating Node Key Constraints (only for enterprise edition)

```cypher
// uniqueness and existence
CREATE CONSTRAINT <constraint_name> IF NOT EXISTS
FOR (x:<node_label>)
REQUIRE x.<property_key> IS NODE KEY

// for multiple properteis
CREATE CONSTRAINT <constraint_name> IF NOT EXISTS
FOR (x:<node_label>)
REQUIRE (x.<property_key1>, x.<property_key2>)  IS NODE KEY
```

## Managing Constraints

![image-20231015123725896](.\images\image-20231015123725896.png)

### Dropping constraints

```cypher
// drop a constraint
DROP CONSTRAINT <constraint_name>

// create a list of all constraints
show constraints yield name return collect('drop constraints' + name + ";") as statements

// droping all constraints
CALL apoc.schema.assert({}, {}, true)
```



# Indexes

### Best practice for leading date

1. create constraints
2. load data
3. create indexes

### In query planning at most one index can be used

### Types for indexes

- range
- composite
- text
- point
- full-text

### Range Index

an implementation of b-tree'
#### use cases:

- equality checks: `=` (for string properties `text` index maybe perform better)
- range comparisons: `>`, `>=`, `<`, `<=`
- string comparisons: `STARTS WITH` (can use `text`  index but may not be performant as well as `range` index)
- existence checks: `IS NOT NULL`
- `ENDS WITH` and `CONTAINS`: may benefit slightly from `range` index but it's recommended to use `text` index for them

### Composite Indexes

Range index on multiple properties of a node or relationship

Properties can be of mixed types
we use this index when multiple properties always tested together in queries
we know that only one index can be used in query planning. so in queries like this:

```cypher
match (m:Movie) where m.year > 1999
and m.title contains "Toy"
return m.title, m.year, m.imdbRating
```

a composite index can make query more performant

### Text Indexes

for node or relationship of type string

use cases:

- equality checks: `=`

  string comparisons: `ENDS WITH` , `CONTAINS`

- List membership `x.prop in ["a", "b", "c"]`

### Lookup Indexes

- created automatically => index for matching node label or relationship type
  - you should never delete these indexes 

```cypher
// show all defined indexes
show indexes

// always we should have an `lookup` index which created by neo4j for us
// never drop this one
```

## Range Indexes

```cypher
create index <index_name> if not exists
for (x:<node_label>)
on x.<property_key>
```

consider this query
```cypher
profile
match (m:Movie)
where m.title starts with "Toy"
return m.title
// this query need 27000 db hits

create index movie_title if not exists
for (x:Movie)
on (x.title)

// after creating this range index
// total db hits of query reduce to 8!
// this expriment shows us whe always should profile important queries and create indexes for them
```

### creating a range index on relationship

```cypher
create index <index_name> if not exists
for ()-[x:<relationship_type>]-()
on (x.<property_key>)
```

 ### Drop index

```cypher
DROP INDEX <index_name>
```

## Composite Indexes

```cypher
CREATE INDEX <index_name> IF NOT EXISTS
FOR (x:<node_label>)
ON (x.<property_key1>, x.<property_key2>) 	
```

```cypher
match (m:Movie)
where m.year = 2000 and m.runtime <= 60
return m.title, m.year, m.runtime
// this query needs 28000 db hits
// then create an index
create index movie_year_title_index
for (x:Movie)
on (x.year, x.runtime)
// now the last query needs only 10 db hits to run
// i notice that the order of properties in index creation is influntial to quality of index
// for example `on (x.runtime, x.yaer)` is not performant as `on (x.year, x.runtime`)
// because in query .year comes first and then .runtime

// this index is not used when we check existstence of a property
// also when we query based on one of those properties (.title, .runtime), engine doesn't use index for matching nodes
```

```cypher
CREATE INDEX <index_name> IF NOT EXISTS
FOR ()-[x:<relationship_type>]-()
ON (x:<property_key1>, x.<property_key2>)
```

## Text Indexes

```cypher
CREATE TEXT INDEX <index_name> IF NOT EXISTS
FOR (x:<node_label>)
ON x.<property_key>
```

we know that  generally total db hits is a good measure to compare performance.
but in case of text indexes sometimes total db hits is high but elapsed time of running query is lower. when we have text index we should consider that lower elapsed time is more important.

### Why use a Text Index over a Range Index?

- text indexes perform better than range index when we have a lot of duplicate properties.
- text indexes take up less space in the graph

```cypher
CREATE TEXT INDEX <index_name> IF NOT EXISTS
FOR ()-[x:<relationship_type>]-()
ON x.<property_key>
```



## Full-Text Index

- useful for queries that must parse a string property value

- implemented to use Apache Lucene

- you can use Lucene's query language to retrieve data

- you need explicitly call to use the index


  ```cypher
  call db.index.fulltext.queryNodes
  ('Movie_plot_ft', 'murder AND drugs')
  YIELD node
  ```

  ```cypher
  // creating full-text index
  create fulltext index <index_name> if not exists
  for (x:<node_label>)
  on each [x.<property_key>]
  ```

  when you profile this type of procedure call, total db hits is wrong value. only elapsed time should be consider.

  ```cypher
  create fulltext index <index_name> if not exists
  for ()-[x:<relationship_type>]-()
  on each [x.<property_key>]
           
           
  // for multiple properties
  // you can combine multiple label and multiple nodes 
  create fulltext index <index_name> if not exists
  (x:<node_label1> | <node_label2> | ...)
  on each [x.<property_key1>, x.<property_key2>,...]
  ```

  ```cypher
  call db.index.fulltext.queryRelationships
  ("<index_name", "lucene_query")
  yield relationship
  return relationship.<property>
  ```

  example

  ```cypher
  create text index movie_plot_text if not exists
  for (x:Movie)
  on (x.plot)
  
  create text index movie_title_text if not exists
  for (x:Movie)
  on (x.text)
  
  create text index actor_bio_text if not exists
  for (x:Actor)
  on (x.bio)
  ```

  ```cypher
  profile
  match (m:Movie)
  where (m.plot contains 'british'
  or m.plot contains "British")
  and (m.title contains "death" or m.title contains "Death")
  return m.name as Name, m.bio as Bio, m.title as Title, m.plot as Plot
  
  Union All
  
  match (a:Actor)
  where (a.bio contains "British" or a.bio contains "british")
  and (a.bio contains "actress" or a.bio contains "Actress")
  ```

  

  
