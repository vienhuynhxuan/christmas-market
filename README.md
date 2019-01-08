# Gaming the Christmas Market

## Introduction

The [Bath Christmas Market](https://bathchristmasmarket.co.uk) is a yearly extravaganza, when the city of Bath is 
transformed into a veritable Winter Wonterland with a selection of gift chalets for all your Christmas purchasing 
requirements. Or at least, that's one perspective. For me, long in the tooth and a little bit grumpy, it's not quite so
evocative. A log jam of people shuffling between chalets with their Christmas spirit disappearing faster than the mince 
pies and hot toddy. When it comes to Christmas shopping, a high focus on efficiency is what is required! 

This mini project uses the Neo4j Graph Database to determine the optimum route through the christmas market, given 
a set of mandatory chalets to purchase from. It also shows a user friendly visual of the route, using Neo4j Desktop and 
making use of APOC capabilities with virtual nodes and relationships.

**TODO** - add version of Neo4j

Full source code, licensed under Apache License 2.0 is [available](https://github.com/dbarton-uk/christmas-market).
 
**Use Cases**

1. Optimum Route: Given a set of chalets to visit, define an optimum travel route between the chalets such that each of 
the set is visited at least once.
 
2. Visualize Route: Given the optimum route, provide a user friendly visual of the route.

## Setting up the Data

### Overview

The christmas market is split into zones, each defined by a unique name. A zone hosts a number of chalets, each with a 
unique name and number. A chalet has a description and is categorized by the type of gift that it sells. Links between 
chalets have been manually defined, with a cost assigned to each link. The link with its cost is used to determine the 
optimal route to take when visiting a given set of chalets.
 
60 chalets across 8 zones are defined, with the data sourced originally sourced from [The Bath Christmas Market website](https://bathchristmasmarket.co.uk) 
The raw data is available in the linked [spreadsheet file](https://github.com/dbarton-uk/christmas-market/blob/master/ChristmasMarket.numbers), 
extracted to [csv](https://github.com/dbarton-uk/christmas-market/tree/master/data).

For reference, the original map of the market is here:

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/Bath-Christmas-Market-Map-2018.png?raw=true "Map")

### Create constraints and indexes

Run [create_constraints.cql](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/create_constraints.cql)
to setup constraints and indexes. 

```cypher
CREATE CONSTRAINT ON (z:Zone) ASSERT (z.name) IS NODE KEY;
CREATE CONSTRAINT ON (c:Chalet) ASSERT (c.number) IS NODE KEY;
CREATE CONSTRAINT ON (c:Chalet) ASSERT c.name IS UNIQUE;
CREATE CONSTRAINT ON (c:Chalet) ASSERT c.sequence IS UNIQUE;
```

### Loading the data

1. First run [load_chalets.cql](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/load_chalets.cql)

```cypher
LOAD CSV WITH HEADERS 
  FROM 'https://raw.githubusercontent.com/dbarton-uk/christmas-market/master/data/Chalets-Chalets.csv' 
  AS csv
CREATE (c :Chalet {
  sequence: toInteger(csv.Id),
  number: toInteger(csv.Number),
  name: csv.Name,
  description: csv.Description,
  zone: csv.Zone,
  category: csv.Category
})
MERGE (z:Zone { name: csv.Zone})
WITH c, csv.Category as category, z
CALL apoc.create.addLabels( id(c), 
		[apoc.text.capitalize(apoc.text.camelCase(category)), 
	 	 apoc.text.capitalize(apoc.text.camelCase(z.name))]) 
YIELD node
MERGE (z) -[:HOSTS]-> (c)
```
`Added 68 labels, created 68 nodes, set 308 properties, created 60 relationships, completed after 540 ms.`

The load chalet script does the following:

- Creates chalet nodes based on [chalet csv data](https://github.com/dbarton-uk/christmas-market/blob/master/data/Chalets-Chalets.csv)
in the github repository.

- Create zone nodes based on zone data, embedded in the chalets csv.

- Adds category labels to chalet nodes. 

- Adds zone labels to chalet nodes

- Links zones to chalets with a `:HOSTS` relationship.

The chalets are split into 5 categories: Clothing and Accessories, Food and Drink, Gifts and Homeware, Health and Beauty 
and Home and Garden. These categories were added as labels rather than attributes to improve the visualization options 
available in Neo4j Desktop. Zone names are also added as labels to enhance Neo4j Desktop visualization options.

2. Next run [load_links.cql](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/load_links.cql)

```cypher
LOAD CSV WITH HEADERS 
  FROM 'https://raw.githubusercontent.com/dbarton-uk/christmas-market/master/data/Links-Links.csv' 
  AS csv
MATCH (c1:Chalet {sequence: toInteger(csv.from)})
MATCH (c2:Chalet {sequence: toInteger(csv.to)})
MERGE (c1) -[:LINKS_TO {cost: toInteger(csv.cost)}]-> (c2)
```

`Set 87 properties, created 87 relationships, completed after 347 ms.`

The script creates the links between chalets based on the link csv data in the [repository](https://github.com/dbarton-uk/christmas-market/blob/master/data/Links-Links.csv)
A cost is defined for each link, which is used by the algorithm when calculating an optimal route.

And here is the schema. For clarity, category labels aren't shown.

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/schema.png?raw=true "Database Schema")

3. Check the data

Ok, so, let's see what we have. First the [chalets](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/chalets.cql).

```cypher
MATCH (c :Chalet)
RETURN c.number as Number, c.name as Name, c.description, c.category as Category, c.zone as Zone
  ORDER BY c.zone, c.category, c.number
```

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/chalets_table.png?raw=true "Table of Chalets")

Next, the [intra-zone links](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/intra-zone_links.cql).
```cypher
MATCH p = (c1:Chalet) -[:LINKS_TO]-> (c2:Chalet)
  WHERE c1.zone = c2.zone
RETURN p
```

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/intra-zone_links.png?raw=true "Intra-Zone Links")


And finally, the [inter-zone links](https://github.com/dbarton-uk/christmas-market/blob/master/scripts/inter-zone_links.cql)
```cypher
MATCH p = (c1:Chalet) -[:LINKS_TO]-> (c2:Chalet)
  WHERE c1.zone <> c2.zone
RETURN p
```

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/inter-zone_links.png?raw=true "Inter-Zone Links")

Using zone as a label, means that in Neo4j Desktop, we can colour each chalet by zone. :thumbsup:

### Optimizing the route

#### Choosing the gifts

So now that we are set up and ready to go, let's choose some gifts.

For Bro, something to share (109)
For Nan, something for the garden. (24)
For Grandpa, something tasty (169)
For Toby the dog, some doggy treats (89)
For the kids, something that won't get me in trouble (32, 184) 
and for the missus something to keep her warm and sweet (181, 19)

```cypher
WITH [{ number: 109, for: "Bro"},
       {number: 24, for: "Nan"},
       {number: 169, for: "Grandpa"},
       {number: 89, for: "Toby"},
       {number: 32, for: "The Girl"},
       {number: 184, for: "The Boy"},
       {number: 181, for: "The Missus"},
       {number: 19, for: "The Missus"}
     ] as gifts
UNWIND gifts as gift
MATCH (c:Chalet { number: gift.number})
RETURN
  gift.number as Number,
  gift.for as For,
  c.name as Name,
  c.description as Description,
  c.zone as Zone,
  c.category as Category
  ORDER by c.zone, c.number
```

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/selected_gifts.png?raw=true "Table of Selected Gifts")

#### The Algorithm

Now we know what we are going to purchase, lets find the optimal route around the market. Although the end result is
a single cypher statement, the algorithm should be considered in two parts. The original cypher is split into two for 
easier explaination.

The first part is shown below:

```cypher
WITH  [109, 24, 169, 89, 32, 184, 181, 19] AS selection
MATCH (c:Chalet) where c.number IN selection
WITH collect(c) as chalets
UNWIND chalets as c1
WITH c1,
     filter(c in chalets where c.number > c1.number) as c2s,
     chalets
UNWIND c2s as c2
CALL algo.shortestPath.stream(c1, c2, 'cost', {relationshipQuery: 'LINKS_TO'}) YIELD nodeId, cost
WITH c1,
     c2,
     max(cost) as totalCost,
     collect(nodeId) as shortestHopNodeIds,
     chalets
MERGE (c1) -[r:SHORTEST_ROUTE_TO]- (c2)
SET r.cost = totalCost
SET r.shortestHopNodeIds = shortestHopNodeIds
```

The shortest path between each of the chalets ,represented by selected chalet numbers, is calculated using the 
'out of the box' algo.shortestPath.stream function. New "SHORTEST_ROUTE_TO" relationships are created between each 
distinct pair of selected chalets, with the total cost of the shortest path and the route of the shortest path stored
on the new relationship. Some optimization is achieved by ensuring the shorted path between two nodes is only calculated
once. 

Let's take a look at the newly created shortest routes.

```cypher
WITH  [109, 24, 169, 89, 32, 184, 181, 19] AS selection
MATCH p = (c1) -[:SHORTEST_ROUTE_TO]- (c2)
  WHERE c1.number in selection and c2.number in selection
RETURN p
```

![alt text](https://github.com/dbarton-uk/christmas-market/blob/master/images/shortest_routes.png?raw=true "Mesh of Shortest Routes")

Part 2 of the algorithm will use the total cost of these shortest routes to calculate a path that includes each of the
chalets in the gift selection.


#### Visual of route

### Route visualisation
### Explanation of visual


