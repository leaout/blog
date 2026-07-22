---
title: pgvector 操作手册
date: 2024-12-17
tags: 数据库
---
## pg_vector 函数说明
Supported distance functions are:

- <-> - L2 distance
- <#> - (negative) inner product
- <=> - cosine distance
- <+> - L1 distance (added in 0.7.0)
- <~> - Hamming distance (binary vectors, added in 0.7.0)
- <%> - Jaccard distance (binary vectors, added in 0.7.0)
```
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
SELECT * FROM items WHERE id != 1 ORDER BY embedding <-> (SELECT embedding FROM items WHERE id = 1) LIMIT 5;
Distances
Get the distance

SELECT embedding <-> '[3,1,2]' AS distance FROM items;
For inner product, multiply by -1 (since <#> returns the negative inner product)

SELECT (embedding <#> '[3,1,2]') * -1 AS inner_product FROM items;
For cosine similarity, use 1 - cosine distance

SELECT 1 - (embedding <=> '[3,1,2]') AS cosine_similarity FROM items;

Aggregates
Average vectors

SELECT AVG(embedding) FROM items;
Average groups of vectors

SELECT category_id, AVG(embedding) FROM items GROUP BY category_id;
```
## index
###  HNSW
An HNSW index creates a multilayer graph. It has better query performance than IVFFlat (in terms of speed-recall tradeoff), but has slower build times and uses more memory. Also, an index can be created without any data in the table since there isn’t a training step like IVFFlat.
```
Add an index for each distance function you want to use.

L2 distance
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops);

Note: Use halfvec_l2_ops for halfvec and sparsevec_l2_ops for sparsevec (and similar with the other distance functions)

Inner product
CREATE INDEX ON items USING hnsw (embedding vector_ip_ops);
Cosine distance

CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);
L1 distance - added in 0.7.0

CREATE INDEX ON items USING hnsw (embedding vector_l1_ops);
Hamming distance - added in 0.7.0

CREATE INDEX ON items USING hnsw (embedding bit_hamming_ops);
Jaccard distance - added in 0.7.0

CREATE INDEX ON items USING hnsw (embedding bit_jaccard_ops);
Supported types are:

vector - up to 2,000 dimensions
halfvec - up to 4,000 dimensions (added in 0.7.0)
bit - up to 64,000 dimensions (added in 0.7.0)
sparsevec - up to 1,000 non-zero elements (added in 0.7.0)

Index Options
Specify HNSW parameters

m - the max number of connections per layer (16 by default)
ef_construction - the size of the dynamic candidate list for constructing the graph (64 by default)
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WITH (m = 16, ef_construction = 64);
A higher value of ef_construction provides better recall at the cost of index build time / insert speed.

Query Options
Specify the size of the dynamic candidate list for search (40 by default)

SET hnsw.ef_search = 100;
A higher value provides better recall at the cost of speed.

Use SET LOCAL inside a transaction to set it for a single query

BEGIN;
SET LOCAL hnsw.ef_search = 100;
SELECT ...
COMMIT;
Index Build Time
Indexes build significantly faster when the graph fits into maintenance_work_mem

SET maintenance_work_mem = '8GB';
A notice is shown when the graph no longer fits

NOTICE:  hnsw graph no longer fits into maintenance_work_mem after 100000 tuples
DETAIL:  Building will take significantly more time.
HINT:  Increase maintenance_work_mem to speed up builds.
Note: Do not set maintenance_work_mem so high that it exhausts the memory on the server

Like other index types, it’s faster to create an index after loading your initial data

Starting with 0.6.0, you can also speed up index creation by increasing the number of parallel workers (2 by default)

SET max_parallel_maintenance_workers = 7; -- plus leader
For a large number of workers, you may also need to increase max_parallel_workers (8 by default)
```

#### Indexing Progress
SELECT phase, round(100.0 * blocks_done / nullif(blocks_total, 0), 1) AS "%" FROM pg_stat_progress_create_index;
The phases for HNSW are:

initializing
loading tuples

## IVFFlat
An IVFFlat index divides vectors into lists, and then searches a subset of those lists that are closest to the query vector. It has faster build times and uses less memory than HNSW, but has lower query performance (in terms of speed-recall tradeoff).

Three keys to achieving good recall are:

Create the index after the table has some data
Choose an appropriate number of lists - a good place to start is rows / 1000 for up to 1M rows and sqrt(rows) for over 1M rows
When querying, specify an appropriate number of probes (higher is better for recall, lower is better for speed) - a good place to start is sqrt(lists)
Add an index for each distance function you want to use.

L2 distance

CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
Note: Use halfvec_l2_ops for halfvec (and similar with the other distance functions)

Inner product

CREATE INDEX ON items USING ivfflat (embedding vector_ip_ops) WITH (lists = 100);
Cosine distance

CREATE INDEX ON items USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
Hamming distance - added in 0.7.0

CREATE INDEX ON items USING ivfflat (embedding bit_hamming_ops) WITH (lists = 100);
Supported types are:

vector - up to 2,000 dimensions
halfvec - up to 4,000 dimensions (added in 0.7.0)
bit - up to 64,000 dimensions (added in 0.7.0)
Query Options
Specify the number of probes (1 by default)

SET ivfflat.probes = 10;
A higher value provides better recall at the cost of speed, and it can be set to the number of lists for exact nearest neighbor search (at which point the planner won’t use the index)

Use SET LOCAL inside a transaction to set it for a single query

BEGIN;
SET LOCAL ivfflat.probes = 10;
SELECT ...
COMMIT;
Index Build Time
Speed up index creation on large tables by increasing the number of parallel workers (2 by default)

SET max_parallel_maintenance_workers = 7; -- plus leader
For a large number of workers, you may also need to increase max_parallel_workers (8 by default)

Indexing Progress
Check indexing progress

SELECT phase, round(100.0 * tuples_done / nullif(tuples_total, 0), 1) AS "%" FROM pg_stat_progress_create_index;
The phases for IVFFlat are:

initializing
performing k-means
assigning tuples
loading tuples
Note: % is only populated during the loading tuples phase

## Filtering
There are a few ways to index nearest neighbor queries with a WHERE clause.

SELECT * FROM items WHERE category_id = 123 ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
A good place to start is creating an index on the filter column. This can provide fast, exact nearest neighbor search in many cases. Postgres has a number of index types for this: B-tree (default), hash, GiST, SP-GiST, GIN, and BRIN.

CREATE INDEX ON items (category_id);
For multiple columns, consider a multicolumn index.

CREATE INDEX ON items (location_id, category_id);
Exact indexes work well for conditions that match a low percentage of rows. Otherwise, approximate indexes can work better.

CREATE INDEX ON items USING hnsw (embedding vector_l2_ops);
With approximate indexes, filtering is applied after the index is scanned. If a condition matches 10% of rows, with HNSW and the default hnsw.ef_search of 40, only 4 rows will match on average. For more rows, increase hnsw.ef_search.

SET hnsw.ef_search = 200;
Starting with 0.8.0, you can enable iterative index scans, which will automatically scan more of the index when needed.

SET hnsw.iterative_scan = strict_order;
If filtering by only a few distinct values, consider partial indexing.

CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WHERE (category_id = 123);
If filtering by many different values, consider partitioning.

CREATE TABLE items (embedding vector(3), category_id int) PARTITION BY LIST(category_id);


|架构模式 |  名称  |    结果数据类型    |  参数数据类型    | 类型| 
| ------ | ---- | -------- | -------- | -------- | 
 public   | array_to_halfvec                 | halfvec            | double precision[], integer, boolean   | 函数|
 public   | array_to_halfvec                 | halfvec            | integer[], integer, boolean            | 函数|
 public   | array_to_halfvec                 | halfvec            | numeric[], integer, boolean            | 函数|
 public   | array_to_halfvec                 | halfvec            | real[], integer, boolean               | 函数|
 public   | array_to_sparsevec               | sparsevec          | double precision[], integer, boolean   | 函数|
 public   | array_to_sparsevec               | sparsevec          | integer[], integer, boolean            | 函数|
 public   | array_to_sparsevec               | sparsevec          | numeric[], integer, boolean            | 函数|
 public   | array_to_sparsevec               | sparsevec          | real[], integer, boolean               | 函数|
 public   | array_to_vector                  | vector             | double precision[], integer, boolean   | 函数|
 public   | array_to_vector                  | vector             | integer[], integer, boolean            | 函数|
 public   | array_to_vector                  | vector             | numeric[], integer, boolean            | 函数|
 public   | array_to_vector                  | vector             | real[], integer, boolean               | 函数|
 public   | avg                              | halfvec            | halfvec                                | agg|
 public   | avg                              | vector             | vector                                 | agg|
 public   | binary_quantize                  | bit                | halfvec                                | 函数|
 public   | binary_quantize                  | bit                | vector                                 | 函数|
 public   | cosine_distance                  | double precision   | halfvec, halfvec                       | 函数|
 public   | cosine_distance                  | double precision   | sparsevec, sparsevec                   | 函数
 public   | cosine_distance                  | double precision   | vector, vector                         | 函数
 public   | halfvec                          | halfvec            | halfvec, integer, boolean              | 函数|
 public   | halfvec_accum                    | double precision[] | double precision[], halfvec            | 函数|
 public   | halfvec_add                      | halfvec            | halfvec, halfvec                       | 函数|
 public   | halfvec_avg                      | halfvec            | double precision[]                     | 函数|
 public   | halfvec_cmp                      | integer            | halfvec, halfvec                       | 函数|
 public   | halfvec_combine                  | double precision[] | double precision[], double precision[] | 函数|
 public   | halfvec_concat                   | halfvec            | halfvec, halfvec                       | 函数|
 public   | halfvec_eq                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_ge                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_gt                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_in                       | halfvec            | cstring, oid, integer                  | 函数|
 public   | halfvec_l2_squared_distance      | double precision   | halfvec, halfvec                       | 函数|
 public   | halfvec_le                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_lt                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_mul                      | halfvec            | halfvec, halfvec                       | 函数|
 public   | halfvec_ne                       | boolean            | halfvec, halfvec                       | 函数|
 public   | halfvec_negative_inner_product   | double precision   | halfvec, halfvec                       | 函数|
 public   | halfvec_out                      | cstring            | halfvec                                | 函数|
 public   | halfvec_recv                     | halfvec            | internal, oid, integer                 | 函数|
 public   | halfvec_send                     | bytea              | halfvec                                | 函数|
 public   | halfvec_spherical_distance       | double precision   | halfvec, halfvec                       | 函数|
 public   | halfvec_sub                      | halfvec            | halfvec, halfvec                       | 函数|
 public   | halfvec_to_float4                | real[]             | halfvec, integer, boolean              | 函数|
 public   | halfvec_to_sparsevec             | sparsevec          | halfvec, integer, boolean              | 函数|
 public   | halfvec_to_vector                | vector             | halfvec, integer, boolean              | 函数|
 public   | halfvec_typmod_in                | integer            | cstring[]                              | 函数|
 public   | hamming_distance                 | double precision   | bit, bit                               | 函数
 public   | hnsw_bit_support                 | internal           | internal                               | 函数
 public   | hnsw_halfvec_support             | internal           | internal                               | 函数
 public   | hnsw_sparsevec_support           | internal           | internal                               | 函数
 public   | hnswhandler                      | index_am_handler   | internal                               | 函数
 public   | inner_product                    | double precision   | halfvec, halfvec                       | 函数
 public   | inner_product                    | double precision   | sparsevec, sparsevec                   | 函数
 public   | inner_product                    | double precision   | vector, vector                         | 函数
 public   | ivfflat_bit_support              | internal           | internal                               | 函数
 public   | ivfflat_halfvec_support          | internal           | internal                               | 函数
 public   | ivfflathandler                   | index_am_handler   | internal                               | 函数
 public   | jaccard_distance                 | double precision   | bit, bit                               | 函数
 public   | l1_distance                      | double precision   | halfvec, halfvec                       | 函数|
 public   | l1_distance                      | double precision   | sparsevec, sparsevec                   | 函数
 public   | l1_distance                      | double precision   | vector, vector                         | 函数
 public   | l2_distance                      | double precision   | halfvec, halfvec                       | 函数
 public   | l2_distance                      | double precision   | sparsevec, sparsevec                   | 函数
 public   | l2_distance                      | double precision   | vector, vector                         | 函数
 public   | l2_norm                          | double precision   | halfvec                                | 函数|
 public   | l2_norm                          | double precision   | sparsevec                              | 函数
 public   | l2_normalize                     | halfvec            | halfvec                                | 函数
 public   | l2_normalize                     | sparsevec          | sparsevec                              | 函数
 public   | l2_normalize                     | vector             | vector                                 | 函数
 public   | sparsevec                        | sparsevec          | sparsevec, integer, boolean            | 函数|
 public   | sparsevec_cmp                    | integer            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_eq                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_ge                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_gt                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_in                     | sparsevec          | cstring, oid, integer                  | 函数
 public   | sparsevec_l2_squared_distance    | double precision   | sparsevec, sparsevec                   | 函数
 public   | sparsevec_le                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_lt                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_ne                     | boolean            | sparsevec, sparsevec                   | 函数
 public   | sparsevec_negative_inner_product | double precision   | sparsevec, sparsevec                   | 函数
 public   | sparsevec_out                    | cstring            | sparsevec                              | 函数
 public   | sparsevec_recv                   | sparsevec          | internal, oid, integer                 | 函数
 public   | sparsevec_send                   | bytea              | sparsevec                              | 函数
 public   | sparsevec_to_halfvec             | halfvec            | sparsevec, integer, boolean            | 函数
 public   | sparsevec_to_vector              | vector             | sparsevec, integer, boolean            | 函数
 public   | sparsevec_typmod_in              | integer            | cstring[]                              | 函数
 public   | subvector                        | halfvec            | halfvec, integer, integer              | 函数
 public   | subvector                        | vector             | vector, integer, integer               | 函数
 public   | sum                              | halfvec            | halfvec                                | agg
 public   | sum                              | vector             | vector                                 | agg
 public   | vector                           | vector             | vector, integer, boolean               | 函数
 public   | vector_accum                     | double precision[] | double precision[], vector             | 函数
 public   | vector_add                       | vector             | vector, vector                         | 函数
 public   | vector_avg                       | vector             | double precision[]                     | 函数
 public   | vector_cmp                       | integer            | vector, vector                         | 函数
 public   | vector_combine                   | double precision[] | double precision[], double precision[] | 函数
 public   | vector_concat                    | vector             | vector, vector                         | 函数
 public   | vector_dims                      | integer            | halfvec                                | 函数
 public   | vector_dims                      | integer            | vector                                 | 函数
 public   | vector_eq                        | boolean            | vector, vector                         | 函数
 public   | vector_ge                        | boolean            | vector, vector                         | 函数
 public   | vector_gt                        | boolean            | vector, vector                         | 函数
 public   | vector_in                        | vector             | cstring, oid, integer                  | 函数
 public   | vector_l2_squared_distance       | double precision   | vector, vector                         | 函数
 public   | vector_le                        | boolean            | vector, vector                         | 函数
 public   | vector_lt                        | boolean            | vector, vector                         | 函数
 public   | vector_mul                       | vector             | vector, vector                         | 函数
 public   | vector_ne                        | boolean            | vector, vector                         | 函数
 public   | vector_negative_inner_product    | double precision   | vector, vector                         | 函数
 public   | vector_norm                      | double precision   | vector                                 | 函数
 public   | vector_out                       | cstring            | vector                                 | 函数
 public   | vector_recv                      | vector             | internal, oid, integer                 | 函数|
 public   | vector_send                      | bytea              | vector                                 | 函数|
 public   | vector_spherical_distance        | double precision   | vector, vector                         | 函数|
 public   | vector_sub                       | vector             | vector, vector                         | 函数|
 public   | vector_to_float4                 | real[]             | vector, integer, boolean               | 函数|
 public   | vector_to_halfvec                | halfvec            | vector, integer, boolean               | 函数|
 public   | vector_to_sparsevec              | sparsevec          | vector, integer, boolean               | 函数|
 public   | vector_typmod_in                 | integer            | cstring[]                              | 函数|
