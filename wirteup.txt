Part 1

1. indexes on column id, and on title

imdb=> SELECT * FROM pg_indexes WHERE tablename = 'movies';
 public     | movies    | movies_pkey  |            | CREATE UNIQUE INDEX movies_pkey ON movies USING btree (id)
 public     | movies    | movies_title |            | CREATE INDEX movies_title ON movies USING btree (title)

2. It uses the same query plan as before. But it can also use index scan given that title is indexed.

imdb=> explain select title from movies;
Seq Scan on movies  (cost=0.00..8418.65 rows=464065 width=18)

3. It uses Index Only Scan over the index movies_title instead of sequential scan on movies

imdb=> explain select title from movies order by title;
 Index Only Scan using movies_title on movies  (cost=0.42..15677.40 rows=464065 width=18)

4. It no longer uses Index Only Scan because year is not indexed, and Index Only Scan only supports columns stored in the indexes.

imdb=> explain select title, year from movies order by title;
 Index Scan using movies_title on movies  (cost=0.42..30786.82 rows=464065 width=22)

5. Seq scan on people_wide is a bad idea because the appended column with type character(2000) takes up a huge space in the disk for each record, the query only asks for the name column from people_wide yet seq scan has to look at the character(2000) column for each record, versus Index only scan retrieve the data from the heap files based on the indexes stored, which involves a lot of random access but is still more efficient than seq scan, and this is also why Index only scan is not as good as seq scan for people_reduced.

6.

imdb=> explain analyze select name from people_reduced;
 Seq Scan on people_reduced  (cost=0.00..18.00 rows=1000 width=14) (actual time=0.010..0.209 rows=1000 loops=1)
 Planning time: 0.185 ms
 Execution time: 0.277 ms
imdb=> set enable_seqscan=off;
SET
imdb=> explain analyze select name from people_reduced;
 Index Only Scan using people_reduced_name on people_reduced  (cost=0.28..43.27 rows=1000 width=14) (actual time=0.047..0.205 rows=1000 loops=1)
   Heap Fetches: 0
 Planning time: 0.056 ms
 Execution time: 0.270 ms

imdb=> explain analyze select name from people_wide;
 Index Only Scan using people_wide_name on people_wide  (cost=0.28..43.27 rows=1000 width=14) (actual time=0.021..0.137 rows=1000 loops=1)
   Heap Fetches: 0
 Planning time: 0.077 ms
 Execution time: 0.237 ms
imdb=> set enable_indexonlyscan=off;
SET
imdb=> explain analyze select name from people_wide;
 Seq Scan on people_wide  (cost=0.00..50.00 rows=1000 width=14) (actual time=0.008..0.510 rows=1000 loops=1)
 Planning time: 0.048 ms
 Execution time: 0.587 ms





