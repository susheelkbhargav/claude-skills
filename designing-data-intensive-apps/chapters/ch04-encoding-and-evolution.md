# Chapter 4: Encoding and Evolution

## Core Idea

Code changes roll out gradually (rolling upgrades server-side, laggard clients elsewhere), so old and new code — and old and new data formats — coexist in one running system. Everything hinges on two compatibility properties: **backward compatibility** (newer code can read data written by older code — easy, you know the old format) and **forward compatibility** (older code can read data written by newer code — harder, old code must ignore additions it doesn't understand). Choose an encoding whose **schema evolution** rules give you both, and you can deploy any component in any order without downtime.

## Frameworks Introduced

**Thrift / Protocol Buffers** — schema-driven binary encodings with numeric **tag numbers**.
- *When to use*: internal service-to-service data (RPC, storage) in statically typed shops; you want compact encoding plus codegen.
- *How*: schema (IDL) assigns each field a tag number; encoded record is a concatenation of (tag, type, value) — no field names on the wire. Person record from the book: 59 bytes (Thrift BinaryProtocol), 34 (CompactProtocol, varints + packed tag/type bytes), 33 (Protobuf).
- *Why it works*: tags are stable aliases for names — rename freely, the wire never sees names. Old code skips unknown tags (type annotation tells it how many bytes to skip) → forward compatible; new code still understands old tags → backward compatible.
- *Failure mode*: reuse or change a tag number and all existing data is silently misinterpreted. Add a `required` field after initial deployment and new code fails reading old data — every post-v1 field must be optional or defaulted. Removed tags are burned forever.

**Avro** — schema-driven binary encoding with *no* tags: just values concatenated.
- *When to use*: Hadoop-style big files of records, and anywhere schemas are **dynamically generated** (e.g., dumping a relational DB — column names become field names automatically, no hand-assigned tags to manage).
- *How*: data is decoded by resolving the **writer's schema** (what the encoder used) against the **reader's schema** (what the decoder expects). They need only be *compatible*, not identical: fields matched by name, extra writer fields ignored, missing ones filled from the reader's default. Most compact of all (32 bytes for the same record).
- *Failure mode*: the reader must obtain the writer's schema somehow — object container file header (large files), schema version number per record + schema registry (databases; Espresso does this), or negotiation at connection setup (Avro RPC). Lose that and the bytes are unreadable — nothing on the wire says "this is a string." Fields without defaults break compatibility on add (backward) or remove (forward).

## Key Concepts

1. **Encoding/decoding** (serialization/marshalling): translating between in-memory structures and self-contained byte sequences. Kleppmann sticks to "encoding" to avoid the transaction sense of "serialization."
2. **Language-built-in serialization** (Java Serializable, pickle, Marshal): convenient, but language-locked, a security hole (decoding instantiates arbitrary classes → RCE), versioning-hostile, and inefficient. Only for transient purposes.
3. **Textual formats (JSON/XML/CSV)**: fine for cross-organization interchange where agreement matters more than efficiency. Pitfalls: number ambiguity (JSON can't distinguish int/float or guarantee precision — Twitter ships 64-bit tweet IDs as *strings* because JS floats corrupt ints > 2^53), no binary strings (Base64 hack), optional/ignored schemas.
4. **Binary JSON variants** (MessagePack etc.): modest savings (81 → 66 bytes) because field names still ride along. Schema-driven formats win by omitting names.
5. **Datatype changes**: possible but risky — int32 → int64 truncates when old code reads new data. Protobuf's `repeated` trick: optional scalar → repeated list is a legal evolution; Avro converts types per its resolution rules.
6. **Avro union types + defaults** replace optional/required: nullable must be explicit (`union { null, long }`), preventing accidental-null bugs.
7. **Merits of schemas**: compactness; schema-as-guaranteed-current documentation; ability to check forward/backward compatibility *before* deploying; codegen for static typing. Schema evolution gives schema-on-read flexibility with better guarantees.
8. **Dataflow modes** — who encodes, who decodes: **via databases** (writer encodes, a later reader — maybe your future self — decodes), **via service calls (REST/RPC)** (client encodes request, server decodes and encodes response), **via async message passing** (sender encodes, broker stores and forwards, consumer decodes).
9. **RPC's flaw**: it fakes local calls ("location transparency") but networks add timeouts-with-unknown-outcome, retries needing idempotence, wild latency variance, and no pass-by-reference. Modern frameworks (gRPC on Protobuf, Finagle/Thrift, Rest.li) embrace this with futures and streams. REST remains predominant for public APIs (curl-debuggable, no codegen); RPC dominates intra-datacenter.
10. **Message brokers** (Kafka, RabbitMQ…): asynchronous, one-way; buffer against unavailable consumers, redeliver, decouple sender from recipient's address. Broker doesn't enforce a data model — compatibility is on you, but if the encoding is both backward and forward compatible you can deploy publishers/consumers in any order.

## Mental Models

- **Data outlives code.** You replace all running code in minutes; five-year-old rows sit in their original encoding forever. Migrations are expensive, so databases fake it — add a nullable column, fill nulls on read — and schema evolution makes the whole DB *appear* single-schema.
- **Writing to a database is sending a message to your future self.** Backward compatibility is non-negotiable there; forward compatibility too, since during a rolling upgrade old readers see new writers' data.
- **Tags vs names is the Protobuf/Avro fork.** Tags = wire stability with manual bookkeeping; name-matching = friction-free dynamically generated schemas. Pick by whether a human or a program authors your schemas.
- **Services: servers first, clients second.** That deployment assumption means you need backward compatibility on *requests* and forward compatibility on *responses*.

## Anti-patterns

- Language-native serialization for anything persistent or cross-service (security + lock-in + no versioning).
- Adding a `required` field, or reusing a retired tag number, after v1 of a schema.
- Old code that reads a record, updates it, and writes it back **dropping unknown fields** (Figure 4-7): decode-to-model-object-and-re-encode silently deletes fields newer code wrote. Preserve unknown fields at the application level; same trap when a consumer re-publishes messages.
- Trusting JSON numbers for 64-bit IDs consumed by JavaScript.
- RPC frameworks that hide the network (CORBA, EJB/RMI, DCOM — no compatibility story, all dead). SOAP/WSDL: codegen-dependent, interop-hostile; avoid for new work.
- Distributed actor frameworks on default serialization: Akka's default (Java serialization) breaks rolling upgrades — swap in Protobuf.

## Worked Example

One record: `{userName: "Martin", favoriteNumber: 1337, interests: ["daydreaming","hacking"]}`.

- **JSON**: 81 bytes; field names in every record; MessagePack shaves to 66.
- **Thrift/Protobuf**: schema assigns tags 1/2/3 (`required string userName = 1; optional int64 favorite_number = 2; repeated string interests = 3`). Wire carries only tag+type+value: 33–59 bytes.
- **Avro**: schema has no tags; wire is pure concatenated values (32 bytes) — meaningless without the writer's schema.

*Add a field* `photoURL`: JSON just works if readers ignore extras — but nothing enforces it. Protobuf: new tag 4, must be optional/defaulted; old readers skip tag 4 (forward compat), new readers tolerate its absence (backward compat). Avro: add with a default value; readers on the old schema ignore it, readers on the new schema fill the default when reading old data. *Remove a field*: Protobuf — only if optional, and never reuse the tag. Avro — only if it had a default (else forward compatibility breaks). *Rename*: Protobuf — free, names aren't on the wire. Avro — reader-schema aliases make it backward but not forward compatible.

## Key Takeaways

1. Rolling upgrades force old/new code and data to coexist; design every encoding for backward AND forward compatibility from day one.
2. Schema-driven binary formats (Thrift/Protobuf/Avro) beat schemaless JSON internally: smaller, self-documenting, and compatibility is checkable pre-deploy.
3. Decision criteria: cross-org interchange → JSON/REST (agreement > efficiency); internal static-typed services → Protobuf/gRPC or Thrift; dynamically generated schemas and big analytic files → Avro.
4. Tag numbers are forever: never reuse, never make post-v1 fields required; in Avro, always give evolvable fields defaults.
5. Compatibility responsibilities differ by dataflow mode: databases need both directions (data outlives code); services need backward-compatible requests, forward-compatible responses; brokers need both to deploy in any order.
6. Don't let RPC pretend the network isn't there — retries need idempotence, timeouts mean unknown outcome.

## Problems This Solves (mapped to real system-design examples)

This is the lowest-leverage chapter across the repo's 28 examples — only one grounds it directly. Most designs don't surface schema evolution as a named concern even where it's implicitly present (any Kafka-based pipeline).

| Mechanism | Problem it solves | Example |
|---|---|---|
| Backward/forward-compatible message schema across producer, broker, and consumer | Producers and consumers deploy independently; neither side can assume the other is on the same schema version | Distributed message queue (explicitly requires schema compliance between producer/queue/consumer) |

## Connects To

- **Ch 1** evolvability — this chapter is its data-layer mechanics.
- **Ch 2** schema-on-read vs schema-on-write: evolution gives the flexibility of the former with the guarantees of the latter.
- **Ch 7** "serializability" (transactions) — unrelated to serialization here; **Ch 8** network faults behind RPC's problems; **Ch 10** Avro object container files in batch processing; **Ch 11** message brokers/Kafka in depth.
