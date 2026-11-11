# Chapter 2: Data Models and Query Languages

## Relational Model

- Codd (1970): data stored as tables of rows
- Goal: hide implementation, so the query optimiser picks access paths instead of the developer

## NoSQL

- "Not Only SQL"
- Drivers: scalability, open source, specialised queries, frustration with rigid schemas
- **Polyglot persistence**: relational and non-relational stores used side by side
- **Impedance mismatch**: object-oriented code vs tables needs a translation layer

## Relationships

- Normalisation (storing IDs, not duplicated text) relies on many-to-one relationships
- Document DBs handle joins poorly, so joins move into application code

## Relational vs Document

- **Document**: schema flexibility, locality, closer to app objects. Best for tree-shaped data loaded all at once
- **Relational**: better joins, handles many-to-one and many-to-many
- **Schema-on-read** (document, like dynamic typing) vs **schema-on-write** (relational, like static typing)
  - "Schemaless" is misleading because there's always an implicit schema
- Locality downside: an update rewrites the whole document, so keep documents small
- The two models are converging (Postgres/MySQL support JSON; some document DBs support joins)

## Graph Models

- Vertices plus edges; best when anything can relate to anything

### Property graphs

- e.g. Neo4j: vertices and edges both have IDs, labels and properties
- No restriction on what connects to what, so it evolves easily
- Query language: **Cypher**, declarative pattern matching that supports variable-length traversal
- Graph queries in SQL are possible with `WITH RECURSIVE`, but clumsy

### Triple stores

- (subject, predicate, object)
- Associated with RDF and the semantic web; query language **SPARQL**

### Datalog

- Older foundation. Builds queries from rules that can be recursive

## Summary

- **Document**: self-contained data, few relationships
- **Graph**: everything potentially related
- Pick the model that fits your data
