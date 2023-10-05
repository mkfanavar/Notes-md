# Intermediate Cypher

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

<img src="C:\Users\Mahdi\Desktop\Notes\images\image-20231005101807111.png" style="zoom: 50%;" />



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