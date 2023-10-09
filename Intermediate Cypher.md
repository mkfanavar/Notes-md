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
// cypher has three different time data which ...
return date(), datetime(), time()
//"2023-10-08", "2023-10-08T04:28:14.937000000Z", "04:28:14.937000000Z"

// to test datetime features of neo4j we create a test node on database
merge (x:Test {id: 1})
set x.date = date()
	x.datetime = datetime()
	x.time = time()

// to check data types in database we use this command
call apoc.meta.nodeTypeProperties()
// we see there is thre data type of Date, DateTime and Time for label of Test
```

![image-20231008080818870](C:\Users\Mahdi\Documents\Notes\images\image-20231008080818870.png)

```cypher
// you can access components of date types
match (x:Test)
return x.date.day, x.date.year, x.datetime.year, x.datetime.hour, x.datetime.minute

// you can use string to set a value for a date or datetime
match (x: Test {id: 1})
set x.date1 = date("2023-09-11")
set x.datetime1 = datetime("2023-09-11T18:33:04")
// note that you can use iso format to seting them

// we have duration too
match (x: Text {id: 1})
return duration.between(x.date1, x.date)
// you can use .inDays, .inMonths, .inSeconds too
return duration.between(x.date1, x.date).minutes
// we can add or substract duration from a date
return x.date1 + duration({months: 6})

// the apoc library (an extenstion to neo4j)
match (x:Test {id: 1})
return apoc.temporal.format(x.datetime, "YYYY/MM/dd - HH:mm")
// convert to ISO8601
return apoc.date.toISO8601(x.datetime.epochMillis, 'ms')
```



# Graph Traversal

To write better and more performant queries you need to know how graph engine travers and run queries under the hood.

Anchor of a query: set of nodes where engine start with theme. this set of nodes choose based on match and where clauses and moved to memory. query planner tries to apply filters as soon as possible to reduce  cardinality (number of row in result)



```cypher
// anchor is the person nodes
match (p:Person)-[:ACTED_IN]->(m)
return p.name, m.title limit 100

// anchor is movie nodes because there are fewer than person nodes
match (p:Person)-[:ACTED_IN]->(m:Movie)
return p.name, m.title limit 100

// anchor is person nodes that satisfy the where filter. if the person node has an index on `name` then only one recored retreived. if there is no index it should check filter with all person node once and then retrive nodes.
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Eminem"
return p.name, m.title limit 100

// multiple anchors
// all p1, p2 nodes retrived as anchor
match (p1:Person)-[:ACTED_IN]->(m1)
match (m2)<-[:ACTED_IN]-(p2:Person)
where p1.name = "Tom Hanks"
and p2.name = "Meg Ryan"
and m1 = m2
return m1.title

```

![image-20231009103430323](C:\Users\Mahdi\Documents\Notes\images\image-20231009103430323.png)

```cypher
// anchor in this query is one node of clint eastwood and then expand for the relation 
match (p:Person)--(m:Movie)
where p.name = "Clint Eastwood"
return m.title
```

![image-20231009103223470](C:\Users\Mahdi\Documents\Notes\images\image-20231009103223470.png)

```cypher
// first node for emimem retrived then first relation and 8 mile retrived and then the next relation and movie (explain this better)
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Eminem"
return m.title
```

![image-20231009104158088](C:\Users\Mahdi\Documents\Notes\images\image-20231009104158088.png)



```cypher
// depth-first retrival (explain below code how runs in neo4j engine)
// first node of eminem retived
// relation to 8 mile and 8 mile node
// acted_in relation for movie node -> coActors each at seprate step
match (p:Person)-[:ACTED_IN]->(m:Movie), (coActors:Person)-[:ACTED_IN]->(m)
where p.name = "Eminem"
return m.title as Movie, collect(coActors.name) as CoActors
```



![image-20231009104147750](C:\Users\Mahdi\Documents\Notes\images\image-20231009104147750.png)



```cypher
// in this one we have two match clause, the diffrence is that the second match cluse run after getting movie nodes... so the allActors will be include 'Eminem' too. why? explain this
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = "Eminem"
match (allActors:Person)-[:ACTED_IN]->(m)
return m.title as Movie, collect(allActors.name) as AllActors
```

![image-20231009105002392](C:\Users\Mahdi\Documents\Notes\images\image-20231009105002392.png)

```cypher
// having a label for anchor node is good but
// the label for m (non-anchor node) force for label check which is unnecceray and make more db hits 
match (p:Person)-[:ACTED_IN]->(m:Movie)
where p.name = 'Tom Hanks'
return m.title as Movie

//  this is more performant
match (p:Person)-[:ACTED_IN]->(m)
where p.name = 'Tom Hanks'
return m.title as Movie
```

```cypher
// retrive as path with relation data
match p = ((person:Person)-[:ACTED_IN]->(movie))
where person.name = "Walt Disney"
return p
```

## Varying Length Traversal

```cypher
// finding shortest path between two node without any condition and bound
match p = shortestPath((p1:Person)-[*]-(p2:Person))
where p1.name = "Eminem" and p2.name = "Charlton Heston"
return p
```

![image-20231009111916385](C:\Users\Mahdi\Documents\Notes\images\image-20231009111916385.png)

```cypher
// finding shortest pass based on specific relation
match p = shortestPath((p1:Person)-[:ACTED_IN*]-(p2:Person))
where p1.name = "Eminem" and p2.name = "Charlton Heston"
return p
```

![image-20231009111843148](C:\Users\Mahdi\Documents\Notes\images\image-20231009111843148.png)

```cypher
// all person nodes who are exactly 2 hops away from eminem through acted_in relation
match (p:Person {name: "Eminem"})-[:ACTED_IN*2]-(others:Person)
return others.name as others
```

![image-20231009112904633](C:\Users\Mahdi\Documents\Notes\images\image-20231009112904633.png)  

```cypher
// exactly 4 hops away
match (p:Person {name: "Eminem"})-[:ACTED_IN*4]-(others:Person)
return others.name as others
```



![image-20231009113025142](C:\Users\Mahdi\Documents\Notes\images\image-20231009113025142.png)

```cypher
// up to 4 hops away ... if there was a person under 4 hops away it capture that too
match (p:Person {name: "Eminem"})-[:ACTED_IN*1..4]-(others:Person)
return others.name as others
```



