# Chapter 2: Data Models and Query Languages

## Core Idea
Every data model embodies assumptions about how data will be used — some operations become easy and fast, others awkward and slow. Applications layer models (app objects → general-purpose model like relational/document/graph → bytes on disk), and each layer hides the one below. The single most important discriminator between models is how they handle **relationships**: one-to-many (trees), many-to-one, and many-to-many. Pick the model that matches the relationship structure of your data, because emulating one model in another is always awkward.

## Frameworks Introduced

- **Relationship-driven model selection**: Choose the data model by the dominant relationship type in your data.
  - **When to use document**: data is a self-contained tree of one-to-many relationships (a résumé, an event log), the whole tree is typically loaded at once, relationships between documents are rare, joins rarely needed.
  - **When to use relational**: many-to-one and many-to-many relationships are common; you need joins, normalization, and ad-hoc queries across entities.
  - **When to use graph**: "anything is potentially related to everything" — many-to-many relationships dominate and connections are complex/heterogeneous (social graphs, web graph, road networks).
  - **How**: ask (1) is the data tree-shaped and loaded whole? (2) do many-to-many relationships exist now, or will they as features are added (data tends to become more interconnected over time)? (3) do queries need variable-length traversals?
  - **Why it works / failure mode**: matching model to relationship shape keeps application code simple — shredding a document into tables is cumbersome, but emulating joins in app code atop a document store is slower and more complex than database joins. Failure mode: choosing documents for join-free simplicity, then features (entities-as-references, recommendations) introduce many-to-many relationships and you're rebuilding joins by hand — repeating IMS's 1970s hierarchical-model problems.

- **Schema-on-read vs schema-on-write**: Document databases are not "schemaless" — the reading code assumes an implicit schema, just unenforced (schema-on-read, like dynamic typing); relational databases enforce an explicit schema at write time (schema-on-write, like static typing).
  - **When to use schema-on-read**: heterogeneous data — many object types that can't each get a table, or structure dictated by external systems you don't control. Format changes handled by app code branching on old/new documents.
  - **When to use schema-on-write**: all records share a structure; the schema documents and enforces it. Format changes via migration (`ALTER TABLE` is milliseconds on most DBs — MySQL copies the whole table; a full-table `UPDATE` is slow anywhere, so you can backfill lazily at read time instead).

- **Declarative over imperative querying**: Declare the pattern of data you want (conditions, transformations), not the algorithm. The query optimizer picks indexes, join methods, and execution order — the "access path" chosen automatically instead of hand-coded by the developer (CODASYL's fatal flaw).
  - **Why it works**: build the optimizer once, every application benefits; declarative queries are order-independent, so the engine can reorganize storage, add indexes, and parallelize across cores without breaking queries. Imperative code pins an ordering and must be rewritten to exploit new APIs — same lesson as CSS selectors vs imperative DOM manipulation.

## Key Concepts
- **Impedance mismatch**: the awkward translation layer between OO application objects and relational tables/rows/columns; ORMs reduce boilerplate but can't eliminate it.
- **Normalization**: removing duplication by storing human-meaningful values once and referencing them by ID (IDs never need to change); denormalization duplicates for read convenience at the cost of write overhead and inconsistency risk.
- **Locality**: a document stored as one contiguous string (JSON/BSON) is fetched in one read — an advantage only if you need most of the document at once; updates typically rewrite the whole document, so keep documents small.
- **CODASYL / network model**: 1970s generalization of the hierarchical model where records have multiple parents linked by pointers; querying meant manually navigating access paths, making code inflexible.
- **Hierarchical model**: IMS's tree of nested records — essentially JSON documents circa 1968; handled one-to-many well, many-to-many badly, no joins.
- **Property graph**: vertices and edges, each with an ID and key-value properties; edges have a label plus tail/head vertices; any vertex can connect to any other (Neo4j, Cypher).
- **Triple-store**: all data as (subject, predicate, object) statements; equivalent to the property graph with different vocabulary (Datomic, SPARQL, RDF/Turtle).
- **Datalog**: rule-based query language (subset of Prolog) writing facts as predicate(subject, object); rules define derived predicates that compose and recurse — the foundation later graph languages build on.
- **Polyglot persistence**: relational and non-relational stores used side by side, each where it fits.
- **MapReduce querying**: neither declarative nor fully imperative — pure map/reduce function snippets the framework calls repeatedly; MongoDB later added the declarative aggregation pipeline because two coordinated functions are harder to write and optimize ("a NoSQL system may find itself accidentally reinventing SQL").

## Mental Models
- Use **document** when your data is a self-contained tree loaded whole; use **relational** when many-to-one/many-to-many links need joins; use **graph** when the data is highly interconnected — "for highly interconnected data, the document model is awkward, the relational model acceptable, graph models the most natural."
- Use **schema-on-read** when records are heterogeneous or externally controlled; use **schema-on-write** when uniform structure should be enforced — it's the static vs dynamic typing debate for databases.
- Use **IDs over strings** whenever a value might change or be shared: IDs have no human meaning, so they never need updating; duplicated human-readable values must be updated everywhere (the case for normalization).
- Graph databases are **not CODASYL redux**: no nesting schema (any vertex links to any vertex), direct access by ID or index instead of access paths, no maintained ordering, and declarative queries (Cypher/SPARQL) instead of imperative traversal.

## Anti-patterns
- Denormalizing into documents to avoid joins, then hand-rolling join logic and consistency maintenance in application code — slower and more complex than database joins.
- Treating "schemaless" as no schema: the schema moves into reading code, implicit and unenforced.
- Storing mutable human-meaningful strings ("Greater Seattle Area") duplicated across records instead of an ID reference.
- Large, growing documents: engines load and rewrite whole documents, so size growth destroys the locality advantage.
- Imperative query code that depends on record ordering — blocks storage reorganization, optimization, and parallelism.
- Deeply nested documents: you can't reference a nested item directly, only via access-path-like positions ("second item in positions of user 251").

## Worked Example
**The LinkedIn résumé (Bill Gates, user_id 251).** Relational form: a `users` row (first_name, last_name, summary, region_id, industry_id) plus separate `positions`, `education`, `contact_info` tables with `user_id` foreign keys, and normalized `regions`/`industries` lookup tables. Fetching a profile needs a multi-way join or several queries. Document form: one JSON document with `positions` and `education` as embedded arrays — the one-to-many tree is explicit, locality is great, one query fetches everything. This favors documents.

**The escalation**: add features and the tree breaks down. Make organizations and schools first-class entities (with their own pages/logos) — job entries must now *reference* org entities, not embed strings. Add recommendations: a recommendation shows the recommender's current name and photo, so it must reference the author's profile (if they update their photo, every recommendation reflects it). These are many-to-many relationships: the dotted-box "document" parts remain, but the cross-links require references resolved by joins at read time. In both models the mechanism is the same — a foreign key (relational) or document reference (document) — but relational databases make those joins easy while document databases push the work into application code. Interconnectedness grows with features; the join-free initial fit was temporary.

**The query-language contrast**: "find people who emigrated from the US to Europe" is 4 lines in Cypher (`(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'})` …) but ~29 lines of SQL with `WITH RECURSIVE` CTEs — variable-length traversal (unknown number of joins) is native to graph languages and clumsy in SQL. Different models are designed for different use cases.

## Key Takeaways
1. Data models shape not just storage but how you think about the problem; each layer's model hides the complexity below.
2. Relationship structure decides the model: trees → document, joins/many-to-many → relational, dense interconnection → graph. All three are widely used; emulating one in another is awkward.
3. Document databases are the hierarchical model (IMS, 1968) reborn — great locality and schema flexibility, weak joins and many-to-many. History's fix was the relational model; know why before opting out of it.
4. Schema-on-read vs schema-on-write is a tradeoff, not a right answer — heterogeneous data favors the former, uniform data the latter.
5. Declarative languages (SQL, Cypher, SPARQL, CSS) win because they hide implementation, enable automatic optimization, and parallelize; imperative access-path code (CODASYL) is what they replaced.
6. Data becomes more interconnected as applications grow — evaluate models against future relationships, not just the initial shape.
7. Relational and document databases are converging (JSON in PostgreSQL/DB2, joins in RethinkDB); a hybrid is a good route — use the combination that fits.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Document model for a self-contained one-to-many tree | A trie node, or any structure that's always read/written whole | Search autocomplete (trie serialized into a document store) |
| Graph model — "anything can relate to anything" | Dense many-to-many relationships where joins would be unbounded-depth | News feed system (social graph for friend IDs), Google Maps (road network as nodes/edges for A*) |
| Relational model chosen deliberately over NoSQL | Data has a strict invariant (sums to zero) that needs real joins/constraints, not app-level enforcement | Payment system (double-entry ledger on ACID SQL) |

## Connects To
- **Chapter 3**: how these models are implemented in storage engines; locality on disk.
- **Chapter 4**: encoding formats, JSON's problems, schemas and schema evolution.
- **Chapters 5 & 7**: fault tolerance and concurrency differences between relational and document stores.
- **Chapter 10**: MapReduce in full; graph processing frameworks (Pregel); SQL as a pipeline of MapReduce operations.
- **Part III**: systematic treatment of caching, denormalization, and derived data.
