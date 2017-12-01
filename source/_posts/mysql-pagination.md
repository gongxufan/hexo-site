---
 layout: post
 title: "Efficient pagination of a table with 100M records"
 date: 2016-11-06 15:16
 tags: mysql
 category: 转载
 description: 海量数据分页查询优化的另一个视角。
 
---
mysql物理分页的另一个视角，直接看原文更精彩。

This Chapter is focused on efficient scanning a large table using pagination with offset on the primary key. This is also known as keyset pagination.
## Background
In the chapter, we use the following database structure for example. The canonical example about users should fit any domain.
```sql
CREATE TABLE `users` (
  `user_id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `external_id` varchar(32) NOT NULL,
  `name` varchar(100) COLLATE utf8_unicode_ci NOT NULL,
  `metadata` text COLLATE utf8_unicode_ci,
  `date_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `uf_uniq_external_id` (`external_id`),
  UNIQUE KEY `uf_uniq_name` (`name`),
  KEY `date_created` (`date_created`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```
A few comments about the structure:

* external_id column stores reference to the same user in other system in UUID format
* name represents Firstname Lastname
* metadata column contains JSON blob with all kinds of unstructured data

The table is relatively large and contains around 100 000 000 records. Let’s start our learning journey.
## Scanning a Large Table
Problem: You need to walk thru the table, extract each record, transform it inside your application’s code and insert to another place. We focus on the first stage in the post - scanning the table.

### Obvious and wrong solution
```sql
SELECT user_id, external_id, name, metadata, date_created
FROM users;
```
In my case with 100 000 000 records, the query is never finished. The DBMS just kills it. Why? Probably, because it led to the attempt to load the whole table into RAM. Before returning data to the client. Another assumption - it took too much time to pre-load the data before sending and the query was timed out. Anyway, our attempt to get all records in time failed. We need to find some other solution.
### Solution #2
We can try to get the data in pages. Since records are not guaranteed to be ordered in a table on physical or logical level - we need to sort them on the DBMS side with ORDER BY clause.
```sql
SELECT user_id, external_id, name, metadata, date_created
FROM users
ORDER BY user_id ASC
LIMIT 0, 10 000;

10 000 rows in set (0.03 sec)
```
Sweet. It worked. We asked the first page of 10 000 records, and it took only 0.03 sec to return it. However, how it would work for the 5000th page?
```sql
SELECT user_id, external_id, name, metadata, date_created
FROM users
ORDER BY user_id ASC
LIMIT 50 000 000, 10 000; --- 5 000th page * 10 000 page size

10 000 rows in set (40.81 sec)
```
Indeed, this is very slow. Let’s see how much time is needed to get the data for the latest page.
```sql
SELECT user_id, external_id, name, metadata, date_created
FROM users
ORDER BY user_id ASC
LIMIT 99 990 000, 10 000; --- 9999th page * 10 000 page size

10 000 rows in set (1 min 20.61 sec)
```
This is insane. However, can be OK for solutions that run in background. One more hidden problem with the approach can be revealed if you try to delete a record from the table in the middle of scanning it. 

Say, you finished the 10th page (100 000 records are already visited), going to scan the records between 100 001 and 110 000. But records 99 998 and 99 999 are deleted before the next SELECT execution. In that case, the following query returns the unexpected result:
```sql

```

 >N, id, ...
 1, 100 003, ...
 2, 100 004, ...
 
As you can see, the query skipped the records with ids 100 001 and 100 002. They will not be processed by application’s code with the approach because after the two delete operations they appear in the first 100 000 records. 

Therefore, the method is unreliable if the dataset is mutable.

## Solution #3 - the final one for today

The approach is very similar to the previous one because it still uses paging, but now instead of relying on the number of scanned records, we use the user_id of the latest visited record as the offset.
Simplified algorithm:

We get PAGE_SIZE number of records from the table. Starting offset value is 0.

Use the max returned value for user_id in the batch as the offset for the next page.

Get the next batch from the records which have user_id value higher than current offset.

The query in action for 5 000th page, each page contains data about 10 000 users:
```sql
SELECT user_id, external_id, name, metadata, date_created
FROM users
WHERE user_id > 51 234 123 --- value of user_id for 50 000 000th record
ORDER BY user_id ASC
LIMIT 10 000;

10 000 rows in set (0.03 sec)
```
Wow, it is significantly faster than the previous approach. More than 1000 times.

Note, that the values of user_id are not sequential and can have gaps like 25 348 is right after 25 345. The solution also works if any records from future pages are deleted - even in that case query does not skip records. Sweet, right?

## Explaining performance
For further learning, I recommend investigating results of EXPLAIN EXTENDEDfor each version of the query to get the next 10 000 records after 50 000 000.

>Solution	Time	Type	Keys	Rows	Filtered	Extra
 1 Obvious	Never	ALL	NULL	100M	100.00	NULL
 2 Paging using number of records as offset	40.81 sec	index	NULL / PRIMARY	50M	200.00	NULL
 3 Keyset pagination using user_id as offset	0.03 sec	range	PRIMARY / PRIMARY	50M	100.00	Using where

Let’s focus on the key difference between execution plans for 2nd and 3rd solutions since the 1st one is not practically useful for large tables.

Join type: index vs range. The first one means that whole index tree is scanned to find the records. range type tells us that index is used only to find matching rows within a specified range. So, range type is faster than index.

Possible keys: NULL vs PRIMARY. The column shows the keys that can be used by MySQL. BTW, looking into keys column, we can see that eventually PRIMARY key is used for the both queries.

Rows: 50 010 000 vs 50 000 000. The value displays a number of records analyzed before returning the result. For the 2nd query, the value depends on how deep is our scroll. For example, if we try to get the next 10 000 records after 9999th page then 99 990 000 records are examined. In opposite, the 3rd query has a constant value; it does not matter if we load data for the 1st page of the very last one. It is always half size of the table.

Filtered: 200.00 vs 100.00. The column indicates estimated the percentage of the table to be filtered before processing. Having the higher value is better. The value of 100.00 means that the query looks thru the whole table. For the 2nd query, the value is not constant and depends on the page number: if we ask 1st page the value of filtered column would be 1000000.00. For the very last page, it would be 100.00.

Extra: NULL vs Using where. Provides additional information about how MySQL resolves the query. Usage of WHERE on PRIMARY key make the query execution faster.

I suspect that join type is the parameter of the query that made the largest contribution to performance to make the 3rd query faster. Another important thing is that the 2nd query is extremely dependent on the number of the pages to scroll. More deep pagination is slower in that case.
More guidance about understaing output for EXPLAIN command can be found in the official documentation for your RDBMS.
## Summary
The main topic for the blog post was related to scanning a large table with 100 000 000 records using offset with a primary key (keyset pagination).
 
Overall, 3 different approaches were reviewed and tested on the corresponding dataset. I recommend only one of them if you need to scan a mutable large table.
Also, we revised usage of EXPLAIN EXTENDED command to analyze execution plan of MySQL queries. I am sure that other RDBMS have analogs for the functionality.

原文地址：http://allyouneedisbackend.com/blog/2016/09/24/the-sql-i-love-part-1-scanning-large-table/