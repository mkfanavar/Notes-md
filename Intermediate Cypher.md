# Intermediate Cypher

# Filtering Queries

## some useful commands to understand your database better

```cypher
CALL db.schema.visualization() // to check database data model and schema
CALL db.schema.nodeTypeProperties() // to check node properties
CALL db.schema.relTypeProperties() // to check relations properties
SHOW CONSTRAINTS // to check all constraints, the properties value in the result is the primary key of that type of node.
```



## basic query filtering

```cypher
// ------ rename a value 
match (m:Movie) where m.title = "Toy Story"
return
    m.year as production_year
	m.year >= 2000 as is2000s

// ------ all actors who had been colluages with "Tom Hanks" in a movie
match (p:Person)-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(:Person {name: "Tom Hanks"})
return p

//------- all movies Tom Hanks acted in between 2005 and 2010
match (p:Person {name: "Tom Hanks"})-[:ACTED_IN]->(m:Movie)
where 2005 <= m.year <= 2010
return m
                                                   
//------ usage of is not null
match (p:Person)
where p.died is not null
and p.born.year > 1985
return p.name, p.born, p.died
    
//----- check for existance of a label on a node
match (p:Person)
where p.born.year > 1970
and p:Actor
and p:Director
return p.name, p.born, labels(p)
//----- query person who had acted in and directed a movie (acted and directed same one not to diffrent one)
match (p:Person)-[:ACTED_IN]->(m:Movie)<-[:DIRECTED]-(p)
where p.born.year > 1970
return p.name, p.born, labels(p)
                              
// ----- match a value in list 
match (m:Movie)
where "Iran" in m.countries
return m.title, m.languages

// all directors of horror movies in year 2000
match (p:Person)-[:DIRECTED]->(m:Movie)-[:IN_GENRE]->(g:Genre {name: "Horror"})
where m.year = 2000
return p.name

// all persons who are Actor and Director and born in fifties
match (p:Person)
where p:Actor and p:Director and 1950 <= p.born.year <= 1959
return count(p)
```

## filtering queries by testing strings

```cypher
// some operators for testing strings
where m.title starts with "Toy Story"
where m.title ends with "I"
where m.title contains "River"
// cuation: usage of toLower and toUpper disable usage of index
where toLower(m.title) contains "river"
where toUpper(m.title) contains "RIVER" 
```

## Explain how a query runs

```cypher
Explain <Query>
// this command show you a visual explaination of how the query runs
// this show paln
```

<img src=".\images\image-20231005101807111.png" style="zoom: 50%;" />



## Performance

```cypher
Profile <query>
// explain keyword estimate query steps and plan but profile show the exactly what happend
// when you want profile (check it performance) you should run it twice. first time the plan of query cached and the values from the second run are valuable for you.

// a query could access through diffrent approach
profile match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Tom Hanks"
and exists {(p)-[:DIRECTED]->(m)}
return p.name, labels(p), m.title

// a good mesurment for query performance is db hits
// the other mesurment could be sum of number of rows each step fetch from db
// the above query db hits is 1107
// this is a better soluation
profile match (p:Person)-[:ACTED_IN]->(m:Movie)<-[:DIRECTED]-(p)
where p.name = "Tom Hanks"
return p.name, labels(p), m.title
// in this solution db hits is 105
```



# Controlling Results Returned

## Ordering returned values

by default text order alphabetical, numeric values order numerically , ...

```cypher
// order based on date
match (p:Person)
where p.born.year = 2000
return p.name as name, p.born.year as born
order by p.born desc

// order by multiple value
match (p:Person)-[:ACTED_IN|DIRECTED]->(m:Movie)
where p.name = 'Tom Hanks'
or p.name = 'Tom Cruise'
return m.year, m.title
order by m.year desc, m.title
    
// all movie titles order by imdb rating
MATCH (m:Movie)
WHERE m.imdbRating IS NOT NULL
RETURN m.title, m.imdbRating
ORDER BY m.imdbRating DESC
```

## Limiting Results

```cypher
// `limit` returned results 
match (m:Movie)
where m.released is not null
return m.title as title, m.released as releasedAt
order by m.released limit 100

// use `skip` for pagination
match (p:Person)
where p.born.year = 1990
return p.name as name, p.born as born
order by p.born skip 40 limit 10

// eliminate duplicate records with `distinct`
match (p:Person)-[:ACTED_IN|DIRECTED]->(m:Movie)
where p.name = "Tom Hanks"
return distinct m.title, m.released

//
match (m:Movie)
return distinct m.year
order by m.year

// distinct nodes
match (p:Person)-[:DIRECTED|ACTED_IN]->(m:Movie)
where p.name = "Tom Hanks"
return distinct m

// find lowest imdbRating value
match (m:Movie)
where m.imdbRating is not null
return m.imdbRating
order by m.imdbRating limit 1

// 
MATCH (p:Person)-[:ACTED_IN| DIRECTED]->(m)
WHERE m.title = 'Toy Story'
MATCH (p)-[:ACTED_IN]->()<-[:ACTED_IN]-(p2:Person)
RETURN distinct  p.name, p2.name
```

## Map Projections

when you return a node from cypher it will return some internal values as well as node properties.
you can specify what properties of node should be in result of query

```cypher
// Maps in cypher
// a list of key value pairs...
return {sat: 23, sun: 12, mon: 18,tue: 10, wed: 22, thu: 8, fri: 21} as days
// select specified key
return {sat: 23, sun: 12, mon: 18,tue: 10, wed: 22, thu: 8, fri: 21}["mon"] as days
return {sat: 23, sun: 12, mon: 18,tue: 10, wed: 22, thu: 8, fri: 21}.mon as days
// return a list of keys
return keys({sat: 23, sun: 12, mon: 18,tue: 10, wed: 22, thu: 8, fri: 21}) as days
// when we return a node, values returned as map

// lets look at maps projection with some examples
// this will just return node properties values
match (p:Person)
return p {.*} as person
limit 10

// return only some properties
match (p:Person)
return p {.bornIn, .name} as person
limit 10

// return arbitrary values in object
match (p:Person)-[:ACTED_IN]->(m:Movie)-[:IN_GENRE]->(g:Genre)
where g.name = "Comedy" and p.name = "Tom Hanks"
return m {.*, favorite: true}
```

## Changing Result Returned

```cypher
// using date()
match (p:Person)
where p.born.year < 1930 and p.died is null
return p.name, date().year - p.born.year as age

//concatnation
match (p:Person)
where p.born.year < 1930 and p.died is null
return "Actor: " + p.name as name, date().year - p.born.year as age

// conditional properties
match (p:Person)
where p:Actor
return p.name,p.born,
case
when p.born.year < 1970 then "oldies"
when 1970 <= p.born.year <= 2000 then "bommers"
when 2000 <= p.born.year then "gen-z"
else "unidentified"
end
as generation
order by p.name limit 500
```

# Working With Cypher Data

## Aggregating Data

```cypher
// aggregation function count()
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Tom Hanks"
return p.name as actorName,
count (*) as numMovies

// find actor, director pair with most work together
match (p:Person)-[:ACTED_IN]->(:Movie)<-[:DIRECTED]-(d:Director)
where p <> d
return p.name as actor, d.name as director, count (*) as numMovies
order by numMovies desc
limit 10

// count(n) vs count(*)
// count (n) : the graph engine calculates the number of non-null occurrences of n.
// count (*) : the graph engine calculates the number of rows retrieved, including those with null values.

// return a list
match (p:Person)
return p.name , [p.born, p.died] as lifeTime
limit 10

// aggregation function collect()
match (p:Person)-[:ACTED_IN]->(m:Movie)
return p.name as actor,
count(*) as total,
collect(m.title) as movies
order by total desc limit 10

// usage of distinct in collect function
match (p:Person)-[:ACTED_IN]->(m:Movie)
where m.year = 1920
return collect(distinct m.title) as movies,
collect(p.name) as actors

// use collect function on nodes + usage of prorperty selection in collect function
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Tom Cruise"
return collect(m {.title, .year}) as movies

// usage of index + size function
match (p:Person)-[:ACTED_IN]->(m:Movie)
return m.title as movie,
collect(p.name)[0] as fistCast,
size(collect(p.name)) as castSize

// return slice of collection
match (p:Person)-[:ACTED_IN]->(m:Movie)
return m.title as movie,
collect(p.name)[2..] as actors,
size(collect(p.name)) as castSize

// other aagregation functions are (add explanation and examples)
// min()
// max()
// avg()
// stddev()
// sum


// count() vs size()
// use count to count rows and size for size of collected results
// in most cases count is more performant... why?


// list comprehension...explain more.
match (m:Movie)
return m.title as movie,
[x in m.countries where x = "USA" or x = "Germany"]
as country limit 100

// pattern comprehension [<pattern>|<return-value>]
match (p:Person {name: "Tom Hanks"})
return [(p)-->(m:Movie) where m.title contains "Toy" | m.title + ": "+ m.year] as movies
// result: ["Toy Story of Terror: 2013", "Toy Story 3: 2010", "Toy Story 2: 1999", "Toy Story: 1995"]

// another example
match (m:Movie)
where m.year = 2015
return m.title,
[(dir:Person)-[:DIRECTED]->(m) | dir.name] as directors,
[(act:Person)-[:ACTED_IN]->(m) | act.name] as actors

// explain it:
match (p:Person)-[:DIRECTED]->(m:Movie)
return p.name, count(*)
order by count(*) desc
```

## Working with Date and Time

```cypher
```

