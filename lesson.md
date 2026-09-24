# Neo4j Graph Database CRUD

## Agenda

1.  Why graph databases?
2.  Graph data model: nodes, relationships, and properties
3.  Cypher basics
4.  CRUD: Create, Read, Update, Delete
5.  Importing CSV data and designing a graph model
6.  Hands-on CRUD in Neo4j Aura
7.  Hands-on CRUD from Python in Google Colab

## Lesson Outcomes

By the end of this lesson, you should be able to:

-   Explain how a graph database differs from a relational database.
-   Identify **nodes, relationships, labels, and properties**.
-   Read simple Cypher patterns.
-   Perform basic **Create, Read, Update, and Delete (CRUD)**
    operations.
-   Turn tabular CSV data into a simple graph data model.
-   Query Neo4j Aura directly and connect to it from Python.

------------------------------------------------------------------------

## 1. Why Graph Databases?

A relational database stores information mainly in **tables, rows,
columns, primary keys, and foreign keys**.

A graph database focuses on **things and how those things are
connected**.

### Relational thinking

  person_id   person_name   movie_id   movie_title
  ----------- ------------- ---------- -------------
  P001        Tom Hanks     M001       Apollo 13
  P002        Kevin Bacon   M001       Apollo 13

To understand connections, an RDBMS normally joins tables using keys.

### Graph thinking

``` text
(:Person {name: "Tom Hanks"})
          |
       ACTED_IN
          |
          v
(:Movie {title: "Apollo 13"})
```

The connection is stored explicitly as part of the graph.

A useful beginner rule is:

> **Nouns → Nodes \| Verbs → Relationships \| Details → Properties**

------------------------------------------------------------------------

## 2. Real-World Applications

### Fraud and financial crime

``` text
Customer → OWNS → Account → TRANSFERRED_TO → Account
```

Graph analysis can reveal chains, shared accounts, circular transfers,
and other suspicious connections that may be difficult to see across
many relational joins.

### Recommendations and connected experiences

``` text
Customer → PURCHASED → Product
Customer → VIEWED → Product
Person → FOLLOWS → Person
```

The connections can be used to find related products, people, content,
or behaviour.

------------------------------------------------------------------------

## 3. The Graph Data Model

We will use the Neo4j **Movie** graph throughout the lesson.

Two important node types are:

``` text
(:Person)
(:Movie)
```

Examples of relationships include:

``` text
(:Person)-[:ACTED_IN]->(:Movie)
(:Person)-[:DIRECTED]->(:Movie)
(:Person)-[:PRODUCED]->(:Movie)
(:Person)-[:WROTE]->(:Movie)
(:Person)-[:REVIEWED]->(:Movie)
(:Person)-[:FOLLOWS]->(:Person)
```

### The four building blocks

**Node** --- a thing or entity.

``` cypher
(:Person)
```

**Label** --- the category/type of a node.

``` cypher
:Person
:Movie
```

**Relationship** --- how two nodes are connected.

``` cypher
-[:ACTED_IN]->
```

**Property** --- information describing a node or relationship.

``` cypher
(:Person {name: "Tom Hanks", born: 1956})
```

A relationship can also have properties:

``` cypher
(:Person)-[:ACTED_IN {roles: ["Jim Lovell"]}]->(:Movie)
```

------------------------------------------------------------------------

## 4. Reading Cypher Like a Sentence

Cypher uses patterns that visually resemble the graph.

``` cypher
MATCH (person:Person)-[:ACTED_IN]->(movie:Movie)
WHERE person.name = "Tom Hanks"
RETURN movie.title
```

Read it as:

> Find a **Person** who **ACTED_IN** a **Movie**, keep the person named
> **Tom Hanks**, and return the movie title.

### Variables are names you choose

In:

``` cypher
MATCH (a:Person)-[:ACTED_IN]->(anyMovies)
RETURN anyMovies
```

`a` and `anyMovies` are temporary **variable names chosen by the query
writer**. They are not labels or fields stored in Neo4j.

A clearer version is:

``` cypher
MATCH (person:Person)-[:ACTED_IN]->(movie:Movie)
RETURN movie
```

Compare:

-   `person` → variable you choose
-   `Person` → node label in the graph
-   `ACTED_IN` → relationship type in the graph
-   `movie` → variable you choose
-   `Movie` → node label in the graph
-   `name`, `title` → properties

------------------------------------------------------------------------

# 5. CRUD in a Graph Database

CRUD stands for:

  Operation    Meaning              Common Cypher
  ------------ -------------------- ----------------------------
  **Create**   Add data             `CREATE`, `MERGE`
  **Read**     Find/traverse data   `MATCH`, `WHERE`, `RETURN`
  **Update**   Change data          `SET`, `REMOVE`
  **Delete**   Remove data          `DELETE`, `DETACH DELETE`

------------------------------------------------------------------------

## 6. CREATE --- Add Nodes and Relationships

### Create a node

``` cypher
CREATE (p:Person {name: "Example Actor", born: 1990})
RETURN p
```

### Create a movie

``` cypher
CREATE (m:Movie {title: "Example Movie", released: 2026})
RETURN m
```

### Connect existing nodes

``` cypher
MATCH (p:Person {name: "Example Actor"})
MATCH (m:Movie {title: "Example Movie"})
CREATE (p)-[:ACTED_IN]->(m)
```

The important idea is:

``` text
CREATE node
CREATE node
CONNECT them with a relationship
```

### `CREATE` vs `MERGE`

`CREATE` creates a new pattern.

`MERGE` first tries to find the requested pattern and creates it when it
does not exist.

``` cypher
MERGE (p:Person {name: "Example Actor"})
RETURN p
```

`MERGE` is useful when you want to avoid unintentionally creating the
same logical entity repeatedly.

------------------------------------------------------------------------

## 7. CREATE from CSV: From Tables to a Graph

Before importing data, decide the **graph model first**.

Suppose the source data describes people, movies, and who acted in each
movie.

### Recommended files

**people.csv**

``` csv
person_id,name,born
P001,Tom Hanks,1956
P002,Kevin Bacon,1958
P003,Meg Ryan,1961
```

**movies.csv**

``` csv
movie_id,title,released
M001,Apollo 13,1995
M002,You've Got Mail,1998
```

**acted_in.csv**

``` csv
person_id,movie_id,role
P001,M001,Jim Lovell
P002,M001,Jack Swigert
P001,M002,Joe Fox
P003,M002,Kathleen Kelly
```

The IDs connect the files:

``` text
people.csv                         movies.csv
    |                                  |
    v                                  v
(:Person) -------[:ACTED_IN]------> (:Movie)
                       ^
                       |
                 acted_in.csv
```

`role` can become a property of the `ACTED_IN` relationship.

Other relationships can use separate files:

**directed.csv**

``` csv
person_id,movie_id
P010,M001
```

**follows.csv**

``` csv
follower_id,followed_id
P001,P002
P002,P003
```

This can produce:

``` text
(:Person)-[:DIRECTED]->(:Movie)
(:Person)-[:FOLLOWS]->(:Person)
```

### CSV preparation checklist

For the Aura Import interface:

-   Use a header row.
-   Give every column a unique name.
-   Include at least one data row.
-   Keep stable IDs for entities that must be matched across files.
-   Separate nodes and relationships where practical.
-   Decide which columns become IDs, labels, properties, and
    relationship properties.

### Build the model in Aura Import

Conceptually, the workflow is:

``` text
CSV / tabular source
        ↓
Identify entities
        ↓
Create node types
        ↓
Identify connections
        ↓
Create relationship types
        ↓
Map IDs and properties
        ↓
Preview
        ↓
Import
```

For the movie example:

``` text
Person                         Movie
person_id                      movie_id
name                           title
born                           released
    \                         /
     \------ ACTED_IN -------/
               role
```

------------------------------------------------------------------------

## 8. READ --- Find and Traverse the Graph

### Find nodes

``` cypher
MATCH (p:Person)
RETURN p.name
```

### Filter

``` cypher
MATCH (p:Person)
WHERE p.name = "Tom Hanks"
RETURN p
```

### Traverse a relationship

``` cypher
MATCH (person:Person)-[:ACTED_IN]->(movie:Movie)
WHERE person.name = "Tom Hanks"
RETURN movie.title
```

### Reverse the question

Instead of:

> Which movies did Tom Hanks act in?

ask:

> Who acted in Apollo 13?

``` cypher
MATCH (person:Person)-[:ACTED_IN]->(movie:Movie)
WHERE movie.title = "Apollo 13"
RETURN person.name
```

The graph pattern is the same. What changes is **which node you filter
and which result you return**.

### Variable-length paths

``` cypher
MATCH (:Person {name: "Kevin Bacon"})-[*1..6]-(n)
RETURN DISTINCT n
```

`*1..6` means traverse paths from **1 to 6 relationship hops**.

An unbounded pattern such as:

``` cypher
-[*]-
```

has no maximum hop count and can become expensive on large graphs.
Prefer a sensible upper bound when the business question provides one.

### Nodes returned vs paths returned

This:

``` cypher
MATCH (:Person {name: "Kevin Bacon"})-[*1..6]-(n)
RETURN DISTINCT n
```

returns reachable **nodes**.

This:

``` cypher
MATCH p = (:Person {name: "Kevin Bacon"})-[*1..3]-(n)
RETURN p
```

returns the **paths**, allowing you to inspect the connections used to
reach the nodes.

### 6 Degrees of Separation

Six degrees of separation is the theory that any person on Earth is connected to any other person by a chain of no more than five intermediaries or six steps.
Let's see if it applies for **Kevin Bacon**. For a start, in the movie database, since every person connected to another person through a movie, 
we can define **1 degree (i.e. 1st connection) = 2 hops (i.e. person -> movie <- another person)**. 

Let's see who is farthest connection from Kevin Bacon and how many degrees away this person is from him. Run the following query to find out:

``` cypher
MATCH p = shortestPath(
(:Person {name:"Kevin Bacon"})-[*]-(actor:Person)
)
WHERE actor.name <> "Kevin Bacon"
RETURN actor.name AS Actor,
length(p) AS Hops,
length(p) / 2 AS Degrees
ORDER BY Degrees DESC, Actor
```

------------------------------------------------------------------------

## 9. UPDATE --- Change Existing Graph Data

Find the node first, then update it.

``` cypher
MATCH (p:Person {name: "Example Actor"})
SET p.born = 1991
RETURN p
```

Add another property:

``` cypher
MATCH (m:Movie {title: "Example Movie"})
SET m.tagline = "An example movie"
RETURN m
```

Properties can also be removed:

``` cypher
MATCH (m:Movie {title: "Example Movie"})
REMOVE m.tagline
RETURN m
```

The pattern is:

``` text
MATCH → SET / REMOVE → RETURN
```

------------------------------------------------------------------------

## 10. DELETE --- Remove Graph Data

Delete a relationship:

``` cypher
MATCH (:Person {name: "Example Actor"})-[r:ACTED_IN]->
      (:Movie {title: "Example Movie"})
DELETE r
```

Delete a node with no remaining relationships:

``` cypher
MATCH (m:Movie {title: "Example Movie"})
DELETE m
```

If the node still has relationships, Neo4j protects the graph from
deleting it directly.

To remove the node **and its relationships**:

``` cypher
MATCH (m:Movie {title: "Example Movie"})
DETACH DELETE m
```

Use `DETACH DELETE` carefully because it also removes the node's
connected relationships.

------------------------------------------------------------------------

## 11. CRUD Mental Model

For learners coming from SQL:

  SQL / RDBMS idea            Graph / Neo4j idea
  --------------------------- -----------------------------------
  Table                       Usually a node label
  Row                         Node
  Column                      Property
  Primary/unique identifier   Identifying property + constraint
  Foreign-key connection      Relationship
  `INSERT`                    `CREATE` / `MERGE`
  `SELECT`                    `MATCH ... RETURN`
  `UPDATE`                    `MATCH ... SET`
  `DELETE`                    `MATCH ... DELETE`
  Multiple joins              Graph pattern/traversal

Do not force every relational table to become a node. Model the
**business meaning**:

> What are the things?\
> How are they connected?\
> What information describes the things or connections?

------------------------------------------------------------------------

# 12. Hands-On: CRUD in Neo4j Aura

Open the Neo4j Aura console and complete the instructor-led CRUD
exercise using the Movie database.

**Neo4j Aura:** https://console.neo4j.io/

You will practise:

``` text
CREATE → READ → UPDATE → DELETE
```

and explore the graph through Cypher patterns.

------------------------------------------------------------------------

# 13. Hands-On: CRUD from Python in Google Colab

Next, perform Neo4j operations programmatically using Python and the
official Neo4j driver.

Open the notebook in this repository:

**[notebook/Integrate_Data_Neo4j.ipynb](notebook/Integrate_Data_Neo4j.ipynb)**

The notebook demonstrates connecting to Neo4j Aura and executing Cypher
from Python, including Movie graph queries such as:

``` cypher
MATCH (a:Person)-[:ACTED_IN]->(movie:Movie)
WHERE a.name = $name
RETURN movie.title
```

> **Credential safety:** Treat the Neo4j URI, username, and password as
> credentials. Do not commit passwords or credential files to GitHub.
> Upload credentials only into your private Colab runtime or use
> environment/secrets management.

------------------------------------------------------------------------

## 14. Key Takeaways

1.  A graph database stores **entities and their connections**.
2.  **Nodes** represent things; **relationships** represent how things
    connect; **properties** describe them.
3.  Cypher patterns resemble the graph you want to find or create.
4.  CRUD maps naturally to `CREATE/MERGE`, `MATCH`, `SET`, and `DELETE`.
5.  For imports, design the **graph model before mapping the CSV
    files**.
6.  Stable IDs help Neo4j connect records from different files.
7.  Graph queries become especially useful when the question is about
    **connections, paths, and degrees of separation**.

## References

-   Neo4j Aura Console: https://console.neo4j.io/
-   Neo4j Aura Import documentation:
    https://neo4j.com/docs/aura/import/introduction/
-   Neo4j Data Importer: https://neo4j.com/docs/data-importer/current/
-   Neo4j CSV import guide:
    https://neo4j.com/docs/getting-started/data-import/csv-import/
-   Neo4j Python Driver manual:
    https://neo4j.com/docs/python-manual/current/
