<!--
.. title: PostgreSQL and faster trigram similarity search
.. slug: postgresql-and-faster-trigram-similarity-search
.. date: 2020-02-12 22:08:24 UTC+01:00
.. tags: postgresql,postgres,trigrams,search
.. category: 
.. link: 
.. description: 
.. type: text
-->

Awesome [PostgreSQL](https://www.postgresql.org/) claims to be the world's most advanced open source relational database. Considering what it can do, it probably really is. Today I want to write about a text search functionality it supports through the [**pg_trgm**](https://www.postgresql.org/docs/12/pgtrgm.html) module.

# pg_trgm

This is a module that already comes with the PostgreSQL database itself. As stated in the documentation:

> The pg_trgm module provides functions and operators for determining the similarity of alphanumeric text based on trigram matching, as well as index operator classes that support fast searching for similar strings.
>
> ([source](https://www.postgresql.org/docs/12/pgtrgm.html))

A trigram is a substring of three consecutive characters in a string. The *pg_trgm* module only takes into accout alphanumeric characters and padds each word with two empty spaces at the beginning and one empty space at the end. Therefore, trigrams of the `word` are computed as `__w`, `_wo`, `wor`, `ord`, `rd_`, where `_` represents one empty space.

# The problem

In one of the applications we are developing, we are given a text document and need to find similar documents in the database. In the year 2020 this calls for document to vector embedding methods and that is precisely what we plan to do. However, for the initial prototype, we thought that trigram similarity offered by the PostgreSQL would suffice. So one of our developers did a quick search on how this could be done. Store the document as text and setup the GIN index with the trigram ops. Something like:

```sql
CREATE TABLE documents (original_id int, searchable_text text);
CREATE INDEX trgm_idx ON documents USING GIN (searchable_text gin_trgm_ops);
```

And for the query, he came up with something like the following:

```sql
WITH current_document AS
(
    SELECT documents.original_id, searchable_text
    FROM documents
    WHERE documents.original_id = 94594
)
SELECT 
    documents.original_id AS original_id
FROM 
    documents, current_document
WHERE 
    documents.searchable_text <> '' 
    AND similarity(documents.searchable_text, current_document.searchable_text) IS NOT NULL
GROUP BY 
    documents.original_id, 
    documents.searchable_text,
    current_document.searchable_text
ORDER BY similarity(documents.searchable_text, current_document.searchable_text) DESC;
```

# Improving the search query

This, however, had a problem of too long execution time in the range of one minute. Since this call was meant to be used online, it was clearly taking too long. In our application each document text is quite large, in the range of few thousands of characters, which obviously impacts the execution time. So I went about optimizing the query using obvious improvements:

1. Adding `similarity` score to the `SELECT`, since this is what will be needed by the application in next steps.
2. Dropping the `GROUP BY`, since it makes no sense.
3. Dropping the `IS NOT NULL` condition in `WHERE`, since already computed similarity cannot be used in the `WHERE` clause. Only existing columns and not computed ones can be used in the `WHERE` clause.
3. Ordering is done by the computed similarity.

So I came up with the following statements (also using the `EXPLAIN ANALYZE` to see what is happening and where the time is spent):

```sql
EXPLAIN ANALYZE WITH current_document AS
(
    SELECT documents.original_id, searchable_text
    FROM documents
    WHERE documents.original_id = 94594
)
SELECT 
    documents.original_id AS original_id, similarity(documents.searchable_text, current_document.searchable_text) AS sml
FROM 
    documents, current_document
WHERE 
    documents.searchable_text <> '' 
ORDER BY sml DESC; 

Limit  (cost=2933.46..2933.47 rows=5 width=8) (actual time=31699.637..31699.648 rows=5 loops=1)
   ->  Sort  (cost=2933.46..2983.03 rows=19829 width=8) (actual time=31699.633..31699.636 rows=5 loops=1)
         Sort Key: (similarity(documents.searchable_text, documents_1.searchable_text)) DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Nested Loop  (cost=0.29..2604.11 rows=19829 width=8) (actual time=1.304..31676.653 rows=19784 loops=1)
               ->  Index Scan using documents_original_id_key on documents documents_1  (cost=0.29..8.31 rows=1 width=104) (actual time=0.134..0.138 rows=1 loops=1)
                     Index Cond: (original_id = 94594)
               ->  Seq Scan on documents  (cost=0.00..2347.94 rows=19829 width=108) (actual time=0.020..38.786 rows=19784 loops=1)
                     Filter: (searchable_text <> ''::text)
                     Rows Removed by Filter: 17037
 Planning Time: 0.272 ms
 Execution Time: 31699.717 ms
```

So the execution time was cut in half for this particular example (and others too). However, the GIN index is not used. After some research I found out why:

> PostgreSQL doesn't use indexed for functions, it only uses them for operators.
>
> _[source](https://stackoverflow.com/a/28546686)_

Since `similarity` is a function, there is no index use. Scanning through the documentation I see that `%` operator is used for similarity testing. Therefore, by limiting the search results only to the similar documents, as returned by the `%` operator, the query should perform better.

```sql
EXPLAIN ANALYZE WITH current_document AS
(
    SELECT documents.original_id, searchable_text
    FROM documents
    WHERE documents.original_id = 94594
)
SELECT 
    documents.original_id AS original_id, similarity(documents.searchable_text, current_document.searchable_text) AS sml
FROM 
    documents, current_document
WHERE 
    documents.searchable_text <> '' 
    AND current_document.searchable_text % documents.searchable_text
ORDER BY sml 
    DESC;

 Limit  (cost=1948.82..1948.83 rows=5 width=8) (actual time=62594.384..62594.396 rows=5 loops=1)
   ->  Sort  (cost=1948.82..1948.87 rows=20 width=8) (actual time=62594.381..62594.384 rows=5 loops=1)
         Sort Key: (similarity(documents.searchable_text, documents_1.searchable_text)) DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Nested Loop  (cost=1800.59..1948.49 rows=20 width=8) (actual time=822.798..62568.580 rows=19146 loops=1)
               ->  Index Scan using documents_original_id_key on documents documents_1  (cost=0.29..8.31 rows=1 width=104) (actual time=0.057..0.061 rows=1 loops=1)
                     Index Cond: (original_id = 94594)
               ->  Bitmap Heap Scan on documents  (cost=1800.30..1939.93 rows=20 width=108) (actual time=821.842..32317.506 rows=19146 loops=1)
                     Filter: ((searchable_text <> ''::text) AND (documents_1.searchable_text % searchable_text))
                     Rows Removed by Filter: 584
                     Heap Blocks: exact=1822
                     ->  Bitmap Index Scan on index_trgm_searchable_text  (cost=0.00..1800.29 rows=39 width=0) (actual time=820.627..820.627 rows=19734 loops=
1)
                           Index Cond: (searchable_text % documents_1.searchable_text)
 Planning Time: 0.346 ms
 Execution Time: 62596.338 ms
```

Now, index is used, however, the execution time has again spiked to a minute. Going through the output, it seems that the similarity operator does not filter out many rows. I assume that because our documents are so long, they are also quite similar with respect to the trigram similarity. After consulting the documentation, I see that `pg_trgm.similarity_threshold` GUC parameter can be increased to limit the similar results. After some trial I finally set on the following:

```sql
SET pg_trgm.similarity_threshold TO 0.8;
EXPLAIN ANALYZE WITH current_document AS
(
    SELECT documents.original_id, searchable_text
    FROM documents
    WHERE documents.original_id = 94594
)
SELECT 
    documents.original_id AS original_id, similarity(documents.searchable_text, current_document.searchable_text) AS sml
FROM 
    documents, current_document
WHERE 
    documents.searchable_text <> '' 
    AND current_document.searchable_text % documents.searchable_text
ORDER BY sml DESC;

 Limit  (cost=1948.82..1948.83 rows=5 width=8) (actual time=15327.644..15327.650 rows=4 loops=1)
   ->  Sort  (cost=1948.82..1948.87 rows=20 width=8) (actual time=15327.640..15327.642 rows=4 loops=1)
         Sort Key: (similarity(documents.searchable_text, documents_1.searchable_text)) DESC
         Sort Method: quicksort  Memory: 25kB
         ->  Nested Loop  (cost=1800.59..1948.49 rows=20 width=8) (actual time=1425.643..15327.608 rows=4 loops=1)
               ->  Index Scan using documents_original_id_key on documents documents_1  (cost=0.29..8.31 rows=1 width=104) (actual time=0.055..0.060 rows=1 loops=1)
                     Index Cond: (original_id = 94594)
               ->  Bitmap Heap Scan on documents  (cost=1800.30..1939.93 rows=20 width=108) (actual time=1424.625..15323.821 rows=4 loops=1)
                     Filter: ((searchable_text <> ''::text) AND (documents_1.searchable_text % searchable_text))
                     Rows Removed by Filter: 6372
                     Heap Blocks: exact=1739
                     ->  Bitmap Index Scan on index_trgm_searchable_text  (cost=0.00..1800.29 rows=39 width=0) (actual time=820.814..820.814 rows=6379 loops=1
)
                           Index Cond: (searchable_text % documents_1.searchable_text)
 Planning Time: 0.410 ms
 Execution Time: 15329.717 ms
```

So, 15 seconds execution time. Since this is not our final solution, I will leave it be. To speed things up, I will set-up caching on the application side. Since the documents do not change or update very frequently, caching with 10 minutes time-to-live, should be OK.

I have also tried using GIST index, however, the creation of index fails, because our text is too large:

```sql
CREATE INDEX index_trgm_searchable_text ON documents USING GIST (searchable_text gist_trgm_ops);
ERROR:  index row requires 12256 bytes, maximum size is 8191
```

