Part 1

1. indexes on column id, and on title
```
imdb=> SELECT * FROM pg_indexes WHERE tablename = 'movies';
 public     | movies    | movies_pkey  |            | CREATE UNIQUE INDEX movies_pkey ON movies USING btree (id)
 public     | movies    | movies_title |            | CREATE INDEX movies_title ON movies USING btree (title)
```
2. It uses the same query plan as before. But it can also use index scan given that title is indexed.
```
imdb=> explain select title from movies;
 Seq Scan on movies  (cost=0.00..8418.65 rows=464065 width=18)
```
3. It uses Index Only Scan over the index movies_title instead of sequential scan on movies
```
imdb=> explain select title from movies order by title;
 Index Only Scan using movies_title on movies  (cost=0.42..15677.40 rows=464065 width=18)
```
4. It no longer uses Index Only Scan because year is not indexed, and Index Only Scan only supports columns stored in the indexes.
```
imdb=> explain select title, year from movies order by title;
 Index Scan using movies_title on movies  (cost=0.42..30786.82 rows=464065 width=22)
```
5. Seq scan on people_wide is a bad idea because the appended column with type character(2000) takes up a huge space in the disk for each record, the query only asks for the name column from people_wide yet seq scan has to look at the character(2000) column for each record, versus Index only scan retrieve the data from the heap files based on the indexes stored, which involves a lot of random access but is still more efficient than seq scan, and this is also why Index only scan is not as good as seq scan for people_reduced.

6. Running with seq scan on people_reduced takes 0.2777 ms, and index only scan takes 0.270 ms. A slight improvement, so Postgres isn't optimizing it. Postgres is clearly estimating the cost wrong for index only scan and seq scan.
```
imdb=> explain analyze select name from people_reduced;
 Seq Scan on people_reduced  (cost=0.00..18.00 rows=1000 width=14)
  (actual time=0.010..0.209 rows=1000 loops=1)
 Planning time: 0.185 ms
 Execution time: 0.277 ms
imdb=> set enable_seqscan=off;
SET
imdb=> explain analyze select name from people_reduced;
 Index Only Scan using people_reduced_name on people_reduced  (cost=0.28..43.27 rows=1000 width=14)
   (actual time=0.047..0.205 rows=1000 loops=1)
   Heap Fetches: 0
 Planning time: 0.056 ms
 Execution time: 0.270 ms
```
Running seq scan takes 0.587 ms which is slower than its planned index only scan that only takes 0.237 ms. The estimated cost for both scans are off from the actual cost, and the estimated ratio between the two scans is also off, but because it estimates seq scan slower it made the right choice on index only scan.
```
imdb=> explain analyze select name from people_wide;
 Index Only Scan using people_wide_name on people_wide  (cost=0.28..43.27 rows=1000 width=14)
   (actual time=0.021..0.137 rows=1000 loops=1)
   Heap Fetches: 0
 Planning time: 0.077 ms
 Execution time: 0.237 ms
imdb=> set enable_indexonlyscan=off;
SET
imdb=> explain analyze select name from people_wide;
 Seq Scan on people_wide  (cost=0.00..50.00 rows=1000 width=14)
 (actual time=0.008..0.510 rows=1000 loops=1)
 Planning time: 0.048 ms
 Execution time: 0.587 ms
```

7. Since multicolumn index works the best when the constraints(WHERE clause) applies on the left most column, which is movie_id, then the second query that applies constraint on person_id still needs to lookup all movie_id indexed in the b-tree so that unmatched records are skipped. Since movie_id and person_id have a one to many relationship, it further confirms that (movie_id, person_id) index is best for lookup movie_id, and for lookup person_id it is better to use (person_id, movie_id).

8.
```
imdb=> explain select avg(birth_year) from people
imdb-> join cast_members on people.id = cast_members.person_id
imdb-> join movies on cast_members.movie_id = movies.id
imdb-> where people.birth_year is not null and movies.year > 2005;
 Aggregate  (cost=136659.17..136659.18 rows=1 width=4)
   ->  Hash Join  (cost=32629.19..136177.90 rows=192507 width=4)
         Hash Cond: (cast_members.movie_id = movies.id)
         ->  Hash Join  (cost=21015.61..114407.12 rows=548809 width=14)
               Hash Cond: (cast_members.person_id = people.id)
               ->  Seq Scan on cast_members  (cost=0.00..54568.21 rows=3333521 width=20)
               ->  Hash  (cost=18744.75..18744.75 rows=181669 width=14)
                     ->  Seq Scan on people  (cost=0.00..18744.75 rows=181669 width=14)
                           Filter: (birth_year IS NOT NULL)
         ->  Hash  (cost=9578.81..9578.81 rows=162781 width=10)
               ->  Seq Scan on movies  (cost=0.00..9578.81 rows=162781 width=10)
                     Filter: (year > 2005)
```
```
imdb=> explain select avg(birth_year) from people
imdb-> join cast_members on people.id = cast_members.person_id
imdb-> join movies on cast_members.movie_id = movies.id
imdb-> where people.birth_year is not null and movies.year > 2050;
 Aggregate  (cost=10177.09..10177.10 rows=1 width=4)
   ->  Nested Loop  (cost=0.86..10177.02 rows=27 width=4)
         ->  Nested Loop  (cost=0.43..10099.16 rows=165 width=10)
               ->  Seq Scan on movies  (cost=0.00..9578.81 rows=23 width=10)
                     Filter: (year > 2050)
               ->  Index Scan using cast_members_pkey on cast_members  (cost=0.43..22.53 rows=9 width=20)
                     Index Cond: (movie_id = movies.id)
         ->  Index Scan using people_pkey on people  (cost=0.43..0.46 rows=1 width=14)
               Index Cond: (id = cast_members.person_id)
               Filter: (birth_year IS NOT NULL)
```

9. Seq scan and index scan

10. Hash join and nested loop

11. Q2005's seq scans on movies, people, cat_members are much faster than the estimate, also the hash join of movies and cast_members are very fast compared to estimate(estimated about 103548 units of cost, and actual is about 2835 units) despite it has a lot more rows to join than estimated.
```
                                                                QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=136659.17..136659.18 rows=1 width=4) (actual time=3741.895..3741.895 rows=1 loops=1)
   ->  Hash Join  (cost=32629.19..136177.90 rows=192507 width=4) (actual time=878.841..3713.601 rows=255589 loops=1)
         Hash Cond: (cast_members.movie_id = movies.id)
         ->  Hash Join  (cost=21015.61..114407.12 rows=548809 width=14) (actual time=207.128..2864.562 rows=1391484 loops=1)
               Hash Cond: (cast_members.person_id = people.id)
               ->  Seq Scan on cast_members  (cost=0.00..54568.21 rows=3333521 width=20) (actual time=0.004..353.758 rows=3333521 loops=1)
               ->  Hash  (cost=18744.75..18744.75 rows=181669 width=14) (actual time=206.898..206.898 rows=179651 loops=1)
                     Buckets: 32768  Batches: 1  Memory Usage: 8071kB
                     ->  Seq Scan on people  (cost=0.00..18744.75 rows=181669 width=14) (actual time=0.031..157.656 rows=179651 loops=1)
                           Filter: (birth_year IS NOT NULL)
                           Rows Removed by Filter: 923824
         ->  Hash  (cost=9578.81..9578.81 rows=162781 width=10) (actual time=154.134..154.134 rows=162395 loops=1)
               Buckets: 16384  Batches: 1  Memory Usage: 6661kB
               ->  Seq Scan on movies  (cost=0.00..9578.81 rows=162781 width=10) (actual time=9.551..112.389 rows=162395 loops=1)
                     Filter: (year > 2005)
                     Rows Removed by Filter: 301670
 Planning time: 2.059 ms
 Execution time: 3744.193 ms
(18 rows)
```

Q2050 every estimate except aggregate in the end is much higher than the actual cost.
```

                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=9764.84..9764.85 rows=1 width=4) (actual time=81.895..81.895 rows=1 loops=1)
   ->  Nested Loop  (cost=0.86..9764.77 rows=27 width=4) (actual time=70.866..81.858 rows=3 loops=1)
         ->  Nested Loop  (cost=0.43..9686.91 rows=165 width=10) (actual time=70.803..81.634 rows=6 loops=1)
               ->  Seq Scan on movies  (cost=0.00..9578.81 rows=23 width=10) (actual time=70.701..81.526 rows=1 loops=1)
                     Filter: (year > 2050)
                     Rows Removed by Filter: 464064
               ->  Index Only Scan using cast_members_pkey on cast_members  (cost=0.43..4.61 rows=9 width=20) (actual time=0.079..0.083 rows=6 loops=1)
                     Index Cond: (movie_id = movies.id)
                     Heap Fetches: 0
         ->  Index Scan using people_pkey on people  (cost=0.43..0.46 rows=1 width=14) (actual time=0.035..0.035 rows=0 loops=6)
               Index Cond: (id = cast_members.person_id)
               Filter: (birth_year IS NOT NULL)
               Rows Removed by Filter: 0
 Planning time: 2.052 ms
 Execution time: 82.030 ms
(15 rows)
```

12. When year is > 2016, it switches the plan. It is reasonable since there are much less movies relatively consider it is only 2017.

13. https://drive.google.com/file/d/0B6MVTDt9ZRosbTFIVWJDQm5YQ0k/view?usp=sharing 

14. About 600ms.


