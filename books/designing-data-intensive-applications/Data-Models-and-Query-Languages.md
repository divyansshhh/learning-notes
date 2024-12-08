# Data Models and Query Languages

Primarily discusses three types of data models - _Relational_, _Document_, _Graph_. Here's a breif summary on the three - 

## Relational Databases
1. Useful when different entities in the data (tables) as related.
2. Queried using a declarative language (eg. SQL). 
3. Since it's a declarative language, it can be independently optimized by a query optimizer.
4. A schema is enforced. 
5. Joins can be expensive to compute. 
6. Query optimizer is responsible for chosing the correct index/query execution order to get the result quickly.
7. Modern databases like Postgres also have support for JSON like structures. 


## Document Databases
1. Useful when entities in the data (documents) are not related. 
2. No schema is imposed.
3. Might be quicker to query than Relational Databased due to locality of related data. This means - some query which requires joins in relational databases might not require joins and hence a reponse is returned quickly.
4. Joins (though rare) can be expensive and need to be done at the application layer. 
5. Examples are - MongoDB


## Graph Databases
1. Useful when entities have a lot of many-to-many relationships.
2. Graph databases can be modelled using a relational database but this isn't scalable. 
3. They come with their own declarative query lagugages. One such example is - Cypher. 
4. They don't enforce a schema. 
5. Since the query language is declarative, it can be optimized using a query optimizer without change to application logic. 
