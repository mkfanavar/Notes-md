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

![e](C:\Users\Mahdi\Documents\Notes\images\image-20231015092218581.png)

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

![image-20231015123725896](C:\Users\Mahdi\Documents\Notes\images\image-20231015123725896.png)

### Dropping constraints

```cypher
// drop a constraint
DROP CONSTRAINT <constraint_name>

// create a list of all constraints
show constraints yield name return collect('drop constraints' + name + ";") as statements

// droping all constraints
CALL apoc.schema.assert({}, {}, true)
```

