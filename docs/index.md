![Azzuri](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_medium.png)

Welcome to Emerald Elephants.  The goal of this blog is to demystify some of the workings of the Evergreen ILS database which runs on PostgreSQL.  Where does the name come from?  Evergreen = Emerald, Postgres' mascot is an Elephant.  No, I don't think I'm clever but I'm a sucker for allieration.  After a few years of teaching SQL for librarians classes and periodically getting questions about some of the corners of the Evergreen database I thought, why answer questions individually when I can compile them for posterity and force Github's corporate owners to pay one hundreth  of a penny hosting some more of my text.  Any samples mentioned here will be marked in the appropriate place in the accompanying Github repo @ https://github.com/roganhamby/emeraldelephant.git whose docs/ folder also contains this blog in markdown.  But, that's enough recursion for now.

For those looking for other Evergreen resources:

* [Evergreen WIKI](https://wiki.evergreen-ils.org/doku.php)
* [Jane Sandberg's posts on Evergreen](https://sandbergja.github.io/tags.html#Evergreen-ref)

I take questions via the Github issues and via Twitter @roganhamby or email if you can find it (it's not hard).

Quick links for posts:  
* [Grokking the Relationship Between Transactions and Bills](transactions_and_bills.md) 2020-06-15   
* [Paying Off Bills part 1: The Where of It](payingbills.md#payingbills1) 2020-06-30  
* [Paying Off Bills part 2: The How of It](payingbills.md#payingbills2) 2020-07-12
* [Ancestors and Descendants](#ancdesc) 2020-08-01
* [Views and Tables in A Materialized World](#materialized) 2020-10-06
* [Do the Bibs Match the Copies](#dotheymatch) 2020-10-09
* [Circulation Basics, Action.Circulation](#circbasics1) 2020-10-19
* [Circulation Basics, Tables and Views](#circbasics2) 2020-10-25
* [Circulation Basics, Reports](#circbasics2) 2020-11-30
* [Asking Discrete Questions 1 of 2](#discrete1) 2020-12-28
* [Asking Discrete Questions 2 of 2](#discrete2) 2020-12-31
* [XMLStarlet - Get It Over With](#xmlstarlet) 2021-03-14

### <a name="ancdesc"></a> Ancestors and Descendants

Our last few entries dealt with larger complicated issues in the Evergreen database but this time we’re going to cover something simple, quick, and very useful - ancestors and descendants.  Let us imagine we have a fairly typical setup in Evergreen like this:

```sql
    rhamby=# select id, parent_ou, ou_type, shortname, name from actor.org_unit;
      id  | parent_ou | ou_type | shortname |             name              
     -----+-----------+---------+-----------+-------------------------------
        1 |           |       1 | GOTHAM    | Gotham Public Library System     
      101 |         1 |       2 | BOWERY    | Bowery Neighborhood
      105 |       101 |       3 | MARTHA    | Martha Wayne Memorial Library
      104 |       101 |       3 | THOMAS    | Thomas Wayne Memorial Library
     (4 rows)
```

Here we have a consortium called GOTHAM, a neighborhood system called BOWERY and two local branches inside the BOWERY the Martha Wayne Memorial Library and the Thomas Wayne Memorial Library.  This is a very simple example but you can easily imagine an org tree that has dozens of branches and several other tiers of libraries.  

Let us say you need information about all the branches, it is typical to write a query including each branch.  For example, you could do:

```sql
SELECT COUNT(*) FROM action.circulation WHERE circ_lib IN (104,105);
```

But, what about if some circulations may be at the system level for some reason?  You could of course do:

```sql
SELECT COUNT(*) FROM action.circulation WHERE circ_lib IN (101,104,105);
```

And this works great for just a few branches.  But imagine the example where there are dozens or hundreds.  What if you mistype a value in the list?  You can look them up with a subquery like this:

```sql
SELECT COUNT(*) FROM action.circulation WHERE circ_lib IN (SELECT id FROM actor.org_unit WHERE parent_ou = 101);
```

But, you want the BOWERY also so ….

```sql
SELECT COUNT(*) FROM action.circulation WHERE circ_lib IN (SELECT id FROM actor.org_unit WHERE parent_ou = 101) OR circ_lib = 101;
```

Now, what if there is a fourth tier or subbranches, book mobiles or other complications?  What if you forget an ‘OR’ condition?  That is where descendant functions come in.  This the same way that Evergreen itself looks up lists of descendants to see where things like circulation policies apply.

For example,

```sql
rhamby=# SELECT 1, parent_ou, shortname FROM actor.org_unit_descendants(101);
  ?column? | parent_ou | shortname
 ----------+-----------+-----------
         1 |         1 | BOWERY
         1 |       101 | MARTHA
         1 |       101 | THOMAS
 (3 rows)
```

And to redo the earlier query:

```sql
SELECT COUNT(*) FROM action.circulation WHERE circ_lib IN (SELECT id FROM actor.org_unit_descendants(101));
```

You can do the same with permission groups, permission.grp_descendants.  For example:

```sql
 rhamby=# select id, name, parent from permission.grp_descendants(3);
  id |            name            | parent
 ----+----------------------------+--------
   3 | Staff                      |      1
   4 | Catalogers                 |      3
   5 | Circulators                |      3
   6 | Acquisitions               |      3
   7 | Acquisitions Administrator |      3
   8 | Cataloging Administrator   |      3
   9 | Circulation Administrator  |      3
  10 | Local Administrator        |      3
  11 | Serials                    |      3
  12 | System Administrator       |      3
  13 | Global Administrator       |      3
  14 | Data Review                |      3
  15 | Volunteers                 |      3
 (13 rows)
```

Although less commonly used there are also functions that look at the ancestors of an org unit or permission tree rather than descendants: permission.grp\_ancestors and actor.org\_unit\_ancestors.

Remember if you have ideas feel free to submit an issue or hit me up on Twitter.

### <a name="materialized"></a> Views and Tables in the Material World

_Where does the answer lie? / Living from day to day / If it's something we can't buy / There must be another way / We are spirits in the material world_

\- The Police, "Spirits in the Material World", 1981

The rest of the song doesn’t apply at all but this part could apply to data work.  Also, the Police are great for listening to while you work.  You may feel differently but you would be wrong.  However, while I don’t feel the value of The Police as a band is open to debate my answer to this question is.  As usual, this blog post came from a question, unusually an open ended one this time.  “What is a very Evergreen thing to know about what it does in the database?”

My mind went to pseudo-materialized tables.  Let’s back up.  Everyone reading this knows what a table is - a bunch of data that can be represented in rows and columns.  Most probably know what a view is - it is a query stored by the database and allows you to use it as if it was a table.  

If you need an illustration between the two go to a good sized data base and search a table and ask for everything, say ….

```sql
SELECT * FROM asset.copy;  
```

You might be waiting a good minute or so but you’ll probably get a response fairly quickly even with a huge table.  By the way, do this on a test system.  Seriously.

But if you try …

```sql
SELECT * FROM money.usr_summary;
```

On a small little test system you will get a quick response but on a large system you’re in for a world of wait.  Why?   Because what you’ve actually just done is executed this query:

```sql
SELECT materialized_billable_xact_summary.usr,
    sum(materialized_billable_xact_summary.total_paid) AS total_paid,
    sum(materialized_billable_xact_summary.total_owed) AS total_owed,
    sum(materialized_billable_xact_summary.balance_owed) AS balance_owed
    FROM money.materialized_billable_xact_summary
    GROUP BY materialized_billable_xact_summary.usr;
```

You can get that in psql by doing

    \d+ money.usr_summary

which will show the query behind any view.

However, specifying a value means you’re narrowing it down.

```sql
rhamby=# select * from money.usr_summary where usr = 12345;
   usr   | total_paid | total_owed | balance_owed
---------+------------+------------+--------------
  12345  |      19.25 |      21.50 |         2.25
(1 row)
```

And the response will come lickety-split.  So views are very handy at giving you a manipulated view of (see what I did there?) the data but they don’t solve the problem of getting access to a large collection of manipulated data quickly.

That’s, kind of, sort of, where materialized views come in.  Materialized views, like regular views, store the query as part of its definition but it executes it and stores the data as a table.  The problem with materialized table is that they have to be told to refresh and then if it’s a large table it takes a long time, just like executing a regular view would, so they’re really only useful for large data sets that change infrequently, say once a day.  In applications like Evergreen we sometimes need access to large manipulated views of data with pretty close to real time updates in data.  That is where our pseudo materialized views come in.

If you grab Evergreen from git and go to Open-ILS/src/sql/Pg you can do this:

    grep ‘materialized’ *.sql

This will show entries like ‘money.materialized\_billable\_xact\_summary’.

If you’re new to Evergreen’s git structure this is where the SQL files to create the database structure are stored.  Something like this will then help you find where a specific statement is though over time you’ll probably be able to guess as they’re (mostly) logically organized.

    grep 'money.materialized_billable_xact_summary' *.sql | grep 'CREATE'

    080.schema.money.sql:CREATE TABLE money.materialized_billable_xact_summary AS
    080.schema.money.sql:CREATE INDEX money_mat_summary_usr_idx ON money.materialized_billable_xact_summary (usr);
    080.schema.money.sql:CREATE INDEX money_mat_summary_xact_start_idx ON money.materialized_billable_xact_summary (xact_start);

If you then look in the file you will see:

```sql
CREATE TABLE money.materialized_billable_xact_summary AS
SELECT * FROM money.billable_xact_summary WHERE 1=0;
```

It looks a bit funky because it’s borrowing its base structure from another table but it is a standard, simple, every day, workhorse table.  Note that we don’t call lit a materialized view but we do call it materialized because that is what it is.  And if you want to see how we materialize it just scroll down a bit further in the file and you will see it referenced from triggers like this one:

```sql
CREATE OR REPLACE FUNCTION money.mat_summary_create () RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO money.materialized_billable_xact_summary (id, usr, xact_start, xact_finish, total_paid, total_owed, balance_owed, xact_type)
    VALUES ( NEW.id, NEW.usr, NEW.xact_start, NEW.xact_finish, 0.0, 0.0, 0.0, TG_ARGV[0]);
    RETURN NEW;
END;
$$ LANGUAGE PLPGSQL;
```

That may seem intimidating but functions are just tiny little SQL scripts and all it really means is here is a piece of code to insert into the table these values whenever the code is run.  Well, this kind of function RETURNS TRIGGER which means it will run when events defined for it happen in the database - usually rows of a table is inserted, updated, or deleted.  And if we search for the name of the function we find just a bit further on this:

```sql
CREATE TRIGGER mat_summary_create_tgr AFTER INSERT ON money.grocery FOR EACH ROW EXECUTE PROCEDURE money.mat_summary_create ('grocery');
```

So after a row is added to money.grocery this will run this code and also create a row in the materialized table.  Looking in the same area you will also find functions for updating the materialized table whenever a row is updated there.  Money is a prime candidate for this because we want billing interfaces to respond quickly in the staff client and not have to run the queries constantly.  In the reporter schema, you will also find super\_simple\_record sources to provide cleaned-up versions of title, author, and more that are also materialized.

So, that is my answer for a not unique but something idiosyncratic to be aware that Evergreen does on the database level.

### <a name="dotheymatch"></a> Do the Bibs Match the Copies

The question was simple.  

"We have these records whose fixed fields say that these bibs are cassette audiobooks.  How can we tell if the copies associated with the bibs are actually audiocassettes?""

Aside from pulling the materials off the shelf there isn't a way to tell for absolute sure but there are often clues in the circ modifier, call number label and shelving location of the items.  So doing this has three parts, 1) identifying the bib records by search / icon format, 2) pulling in all the copy level information and 3) analyzing the copy level information.  Number 3 really depends on the circs mods, shelving locations and call numbers of your library so I'm going to cover 1 & 2 here.

First let's look at a table called config.coded_value_map, there is a lot of important stuff in there but for our pupose we are going to focus on the search formats.  These are the things you select when you go to advanced search and say "I only want want mountain climbing videos".  Howe they are defined is elsewhere but for our purpose the fact that this is where the ids and codes are is enough.

```sql
rhamby=# select id, ctype, code, value from config.coded_value_map where ctype ~* 'search';
  id   |     ctype     |     code     |             value             |
-------+---------------+--------------+-------------------------------+
   610 | search_format | book         | All Books                     |
   613 | search_format | dvd          | DVD                           |
   614 | search_format | ebook        | E-book                        |
   616 | search_format | kit          | Kit                           |
   617 | search_format | map          | Map                           |
   621 | search_format | equip        | Equipment, games, toys        |
   622 | search_format | serial       | Serials and magazines         |
   623 | search_format | vhs          | VHS                           |
   625 | search_format | cdaudiobook  | CD Audiobook                  |
   626 | search_format | cdmusic      | CD Music recording            |
   627 | search_format | casaudiobook | Cassette audiobook            |
   628 | search_format | casmusic     | Audiocassette music recording |
   629 | search_format | phonospoken  | Phonograph spoken recording   |
   630 | search_format | phonomusic   | Phonograph music recording    |
   631 | search_format | lpbook       | Large Print Book              |
   633 | search_format | blu-ray      | Blu-ray                       |
...
```

Now let's look at metabib.record_attr_vector_list.  This happy place is where each bib has an array stored with a ton of values that represent things such as search format, language, and tons more things defined in the coded value map.  

```sql
rhamby=# select * from metabib.record_attr_vector_list limit 5;
 source  |                                                       vlist                                                        
---------+--------------------------------------------------------------------------------------------------------------------
 3383799 | {667,634,657,1067,-12648806,1350,-1536,-1140,-2125,-2151,997,181,106,471,221,10573,391,567,591,613}
 3383813 | {667,634,657,1050,-2656,1350,-1536,-1140,-2125,-2151,997,181,106,116,468,471,221,10573,36,567,591,613}
 3382301 | {667,634,657,1050,-1956,-1931,1347,-1536,-1140,-2125,-1415,997,181,106,471,221,10573,110,567,591,613}
 3382764 | {673,637,658,-302,1050,-103,1350,-1321,-1534,-1140,-2125,-2151,1461,1675,181,106,459,310,192,864,564,1659,588,610}
 3383592 | {667,634,657,1335,-1741,-733,1347,-1536,-1140,-2125,-2151,995,181,106,471,221,10573,391,567,591,613}
(5 rows)
```

This seems impenetrable but it's really not.  And for us since we are interested in cassette audiobooks we can see in the coded value list that it is id 627.  So to use it let us make a table of the sources (AKA bib ids) from metabib.record_attr_vector_list where one value of the array matches the id of cassette audiobook, which here is 627.

```sql
CREATE TABLE list AS SELECT source from metabib.record_attr_vector_list where 627 = ANY(vlist);
```

There are a number of places where bib values aren't removed when a bib is flagged deleted so it's not a bad idea to remove them from out list:

```sql
DELETE FROM list WHERE source IN (SELECT id FROM biblio.record_entry WHERE deleted);
```

Now Let us get the information!   

```sql
CREATE TABLE copies AS SELECT acp.id AS acp_ip, acp.barcode, acp.circ_modifier, acn.id AS acn_id, acn.label AS call_number, acl.name AS location, acn.record
FROM asset.copy acp
JOIN asset.call_number acn on acn.id = acp.call_number
JOIN asset.copy_location acl on acl.id = acp.location
WHERE NOT acp.deleted
AND acn.record IN (SELECT source FROM list);

rhamby=# select * from copies limit 5;
 acp_ip  |    barcode     | circ_modifier |  acn_id  |         call_number          |             location             | record  
---------+----------------+---------------+----------+------------------------------+----------------------------------+---------
  792947 | 51842039231743 | AudioBooks    |   840234 | BCA P Zoehfeld Kathleen      | Juvenile Book Audiocassette Kits |  237129
  760550 | 12398092384267 | AudioBooks    |  2048372 | BCA YA 796.334 Miller, Marla | Juvenile Book Audiocassette Kits |  988412
  770435 | 18494039227493 | CD Books      |   741133 | BCA Y Dickens, Charles       | Juvenile Book Audiocassette Kits |  983492
 2386754 | 23402350423941 | Books         | 92382834 | 658 Bla                      | Adult Nonfiction                 | 2908083
  779354 | 24923058137161 | AudioBooks    |   937363 | CAS Hugh, Dafydd ab          | Adult Audiocassette Books        |  984274
(5 rows)
```

There you.  With this you can skim things pretty quickly and construct queries to look at more details.

### <a name="circbasics1"></a> Basics of Circulation Reports part 1 Action.Circulation

Today someone asked on the general Evergreen list about circulation reports.  Their context was the reporter but it brought to mind all the questions I've had over time about circ reports and all the ways they can be miscounted.  So, I'm going to talk about the database here but all of these columns and tables have equivalents in the reporter.  This is a short and dirty common questions about circulations and how to count them.

I'm going to do this as multiple posts so the first one will be just looking at the core circulation table, action.circulation.

There are a lot of potentially useful bits of information in here.  Le'ts start with:

    parent_circ            | bigint                   |
    renewal_remaining      | integer                  | not null

 Many ILSes track renewals by a value on a circulation table and some poeple mistak the renewal_remaining as that.  It is not.  In Evergreen it simply tracks what it says.  Each renewal is its own entry in action.circualation and the previous circulation is linked as parent_circ.    So if your original circulation is id 9001 parent_\circ NULL renwals\_remaining 2, renewal one might be id 9005 parent\_circ 9001 renewals\_remaining 1.

    xact_start             | timestamp with time zone | not null default now()
    create_time            | timestamp with time zone | not null default now()

A lot of people consider these redundant and at first glance they are.  Indeed, the vast majority of the time they are but there are important cases where they are not.  Create time is when the record is created while xact_start is the beginning of the transaction.  In a normal circ they are the same but in migrated or offline transactions they will be different.

    xact_finish            | timestamp with time zone |
    stop_fines_time        | timestamp with time zone |
    checkin_time           | timestamp with time zone |
    checkin_scan_time      | timestamp with time zone |

Let's break these down.  xact\_finish has nothing to do with when the material is returned to the library, it marks the end of the circulation as a billable event, or when the bills (if any) are resolved.  The stop\_fines\_time is simply when fines stop potentially accumulating, either because it was checked in (perhaps without any bills), it hits max fines or whatever.  The checkin_time is when it's _considered_ to be checked in, perhaps due to a checkin modifier or something.  The checkin\_scan\_time is when it was actually scanned in though.

    copy_location          | integer                  | not null default 1

 This is unreliable if you have an older Evergreen system with old circulations.  This captures the shelving location at the time of circulation so if you check out DVDs from a NEW DVDS shelving location and they are later moved to NO ONE CARES ANYMORE DVDS you can see how they circulated then.  However, this when this was added there was no way to know what the shelving location was for items in existing circs so they were set to 1 or Stacks.

    circ_lib               | integer                  | not null

  This one creates a bit of confusion because there is also a circ\_lib on asset.copy.  The difference is pretty simple.  circ\_lib on asset.copy has nothing to do with where an item actually circulates, it has to do with being the field that the circulation matrix refers to in the copy_circ_lib field.  So, in theory it refers to circulating library but it's confusing because a circulation policy may not use it at all so the circulating library may be irrelevant to how it actually circulates.  Indeed, in the asset.copy it really refers to where the item is circulating from.  My opinion is that it should probably be called something like home_lib but that ship has sailed long ago.  The circ\_lib field on action.circulation though - that is where the item actually circulated.

### <a name="circbasics2"></a> Basics of Circulation Reports part 2, Tables and Views

If you go in the reporter and look at circulation data sources it is easy to feel overwhelmed.  There are a bunch of sources and it often isn't clear what is a table and what is a view.  So, today in part two of reporting on circulations we will quickly go through the actual tables that store circulation data and the views that just report it.  The first one we went over in more detail last week, action.circulation.  I don't need to go into it more except to say that when I refer to a circulation like data source I'm going to be talking about one that mostly share the same columns as action.circulaton.  

The table action.aged\_circulation may or may not have anything in it depending on your system preferences.  

Adjacent action.emergency\_closing\_circulation and action.usr\_circ\_history.  There are four global flags to set that set controls for how circulations get aged and a server side cron job that has to be enbled for it.  The purpose of aging circulations is fundamentally to protect patron privacy while keeping statistics.  As circulations are aged the row is removed from action.circulation and moved to action.aged\_circulation and the data transformed.  The value for the usr is entirely removed and new columns are added to provide an annoymous set of patron statistics:  

```sql
         Column         |           Type           |       Modifiers        
 usr_post_code          | text                     |
 usr_home_ou            | integer                  | not null
 usr_profile            | integer                  | not null
 usr_birth_year         | integer                  |
```

Statistically you can actually de-anonymize many transactions with birth year and postal code.  There is a bug here for that:  https://bugs.launchpad.net/evergreen/+bug/1861239 and a patch I supplied for allowing libraries to turn off the collection of one or the other.  It's currently in that discussion limbo stage that bugs sometimes fall into.  My opinions are on the bug if anyone is curious.

If there is one "wow, I didn't know that" I get a lot when I talk about circulation data sources it is that non cataloged circulations are stored separately from regular circulations.  Sometimes I've even talked to libraries and they've said "we don't have any of those so we aren't worried about it."  Me: "Really, how do you handle mass market paperbacks and magazines?"  The answer varies but sometimes it's "Oh, I didn't think about those."

```sql
rhamby=# \d action.non_cataloged_circulation
                                       Table "action.non_cataloged_circulation"
  Column   |           Type           |                                   Modifiers                                   
-----------+--------------------------+-------------------------------------------------------------------------------
 id        | integer                  | not null default nextval('action.non_cataloged_circulation_id_seq'::regclass)
 patron    | integer                  | not null
 staff     | integer                  | not null
 circ_lib  | integer                  | not null
 item_type | integer                  | not null
 circ_time | timestamp with time zone | not null default now()
```

Now come our two in house use tables.  In house use is not heavily used by many libraries but I would argue is a form of circulation.  The first one, the stock in\_house\_use table records in house use with cataloged copies while the second non\_cat\_in\_house\_use records use of non-cataloged items.  

```
rhamby=# \d action.in_house_use
                                      Table "action.in_house_use"
  Column  |           Type           |                            Modifiers                             
----------+--------------------------+------------------------------------------------------------------
 id       | integer                  | not null default nextval('action.in_house_use_id_seq'::regclass)
 item     | bigint                   | not null
 staff    | integer                  | not null
 org_unit | integer                  | not null
 use_time | timestamp with time zone | not null default now()

rhamby=# \d action.non_cat_in_house_use
                                       Table "action.non_cat_in_house_use"
  Column   |           Type           |                                Modifiers                                 
-----------+--------------------------+--------------------------------------------------------------------------
 id        | integer                  | not null default nextval('action.non_cat_in_house_use_id_seq'::regclass)
 item_type | bigint                   | not null
 staff     | integer                  | not null
 org_unit  | integer                  | not null
 use_time  | timestamp with time zone | not null default now()
```

From here we are going to talk about views instead of tables, basically database stored reports of tables rather than tables themeslves.  Let's start with action.all\_circulation.  It is often pointed out as the best table to get an accurate report of your circulations from.  And that isn't bad advice but with some caveats.  It is actually pretty far from complete.  This is cleaned up but the report looks like this:

```sql
SELECT aged_circulation.id,
...
FROM action.aged_circulation
UNION ALL
SELECT DISTINCT circ.id,
...
FROM action.circulation circ
...
```

So what all\_circulation really does is just combine action.circulation and action.aged\_circulation but excludes non-cataloged circulations and in house uses.  It also tries to keep all data even where it varies unlike action.all\_circulation_slim which is the same thing but trims it down slightly to common fields though the rows from the aged circulation table will have NULL for usr.

The view action.all\_circulation\_combined\_types is more comprehensive.  Because many of the tables it pull from are very different it doesn't give as much detail but if you are looking for a count all\_circulation\_combined\_types will give you numbers from all five circulation types and give you a field that lets you know which kind of circ each was.  So, if you're looking to do just a count, definitely use this view.

```sql
SELECT acirc.id,
...
'regular_circ'::text AS circ_type
FROM action.circulation acirc,
...
WHERE acirc.target_copy = ac_acirc.id
UNION ALL
SELECT ancc.id::bigint AS id,
...
'non-cat_circ'::text AS circ_type
FROM action.non_cataloged_circulation ancc,
...
UNION ALL
SELECT aihu.id::bigint AS id,
...
'in-house_use'::text AS circ_type
FROM action.in_house_use aihu,
...
UNION ALL
SELECT ancihu.id::bigint AS id,
...
'non-cat_circ'::text AS circ_type
FROM action.non_cat_in_house_use ancihu,
...
UNION ALL
SELECT aacirc.id,
...
'aged_circ'::text AS circ_type
FROM action.aged_circulation aacirc,
....
```

The view action.billable\_circulations is pretty straightforward, it is all circulations that may still be billed.  It is defined like this...

```sql
SELECT circulation.id,
...
FROM action.circulation
WHERE circulation.xact_finish IS NULL;
```

Note that this does not mean they have bills. Some probably do but the circulation hasn't closed yet so they may get bills still.  This will include by definition everything not checked in yet as well as anything with outstanding bills and other niche cases.  This does exclude the other tables because once a circulation is aged it is not billable anymore (and only closed circs are aged) and others either have no user or are not tracked item by item and therefore do not have the fine rules used for standard circulations applied.

The final view, action.open\_circulation is another minor variation on action.circulation and like billable_\circulations could be accomplished by just applying some filters to the circulations table/data source.  This is a smaller set than billable circulations as it is only those that have not been checked in yet whether they are due or not.

```sql
SELECT circulation.id,
...
FROM action.circulation
WHERE circulation.checkin_time IS NULL
ORDER BY circulation.due_date;
```

Next week we will do a few reports to show some useful circulation reports.

### <a name="circbasics3"></a> Circbasics: A Few Simple Reports

It has been a while since I last updated but we had Turkey day and I've been busy but I said I would do some circulation queries and here we are.  

Let us imagine that you want to count circulations that began during a given date range, say a fiscal year:

```sql
SELECT COUNT(*) FROM action.circulation WHERE DATE(xact_start) BETWEEN '2019-07-01' AND '2020-06-30';
```

 But this has a problem, it only checks circulations, not non-cataloged circulations, not in house use and not aged circulations so let's be a bit more comprehensive and use action.all_circulation_combined_types.  It is a view but we can use it the same way we do a table.

```sql
SELECT COUNT(*) FROM action.all_circulation_combined_types WHERE DATE(xact_start) BETWEEN '2019-07-01' AND '2020-06-30';
```

Now let's get a bit more detail.  Perhaps we want a break down by branches.  We can just add the circ\_lib like this:

```sql
SELECT COUNT(*), circ_lib FROM action.all_circulation_combined_types WHERE DATE(xact_start) BETWEEN '2019-07-01' AND '2020-06-30' GROUP BY 2;
```

but how about we actually link to the actor.org\_unit table so that we can get the library names instead and make it a little prettier.  The first thing is that since I'm adding a second source I'm going to make aliases to make referencing them easier.  The first version here is functionally the same as above but formatted a bit and with aliases.

```sql
SELECT COUNT(acct.*), acct.circ_lib
FROM action.all_circulation_combined_types acct
WHERE DATE(acct.xact_start) BETWEEN '2019-07-01' AND '2020-06-30' GROUP BY 2;
```

Now let's change that circ\_lib to the library name by joining to actor.org_unit and use it's name field.

```sql
SELECT COUNT(acct.*), aou.name
FROM action.all_circulation_combined_types acct
JOIN actor.org_unit aou ON aou.id = acct.circ_lib
WHERE DATE(acct.xact_start) BETWEEN '2019-07-01' AND '2020-06-30' GROUP BY 2;
```

There we go, yay!  Now we are going to both expand and restrict it.  Let us imagine we are in a consortium and we only want a subgroup of libaries.  We are going to restrict the libraries shown to all the branches that are children of a certain one using the Evergreen descendants function.

```sql
SELECT COUNT(acct.*), aou.name
FROM action.all_circulation_combined_types acct
JOIN actor.org_unit aou ON aou.id = acct.circ_lib
WHERE DATE(acct.xact_start) BETWEEN '2019-07-01' AND '2020-06-30'
AND acct.circ_lib IN (SELECT id FROM actor.org_unit_descendants((SELECT id FROM actor.org_unit WHERE shortname = 'INN')))
GROUP BY 2;
```

Note that we are using subselects to get values used in another query.  In one we have to use brackets () inside brackets because the innermost ones are wrapping around the value while the outer ones are for the function actor.org\_unit\_descendants.  We are nearly there.  Now we are going to find out what kind of circulations these are by adding circ\_type to the select list and chaging the group by to columns 2 and 3 instead of just 2.

```sql
SELECT COUNT(acct.*), acct.circ_type, aou.name
FROM action.all_circulation_combined_types acct
JOIN actor.org_unit aou ON aou.id = acct.circ_lib
WHERE DATE(acct.xact_start) BETWEEN '2019-07-01' AND '2020-06-30'
AND acct.circ_lib IN (SELECT id FROM actor.org_unit_descendants((SELECT id FROM actor.org_unit WHERE shortname = 'INN')))
GROUP BY 2, 3;
```
```
count  |  circ_type   |                 name                 
--------+--------------+--------------------------------------
   508 | in-house_use | Innsmouth - Bookmobile
  6390 | in-house_use | Innsmouth - Ogunquit Main Library
 11385 | in-house_use | Innsmouth - Lighthouse Point Branch
     1 | in-house_use | Innsmouth Township Library System
 12587 | regular_circ | Innsmouth - Bookmobile
 45813 | regular_circ | Innsmouth - Ogunquit Main Library
393197 | regular_circ | Innsmouth - Lighthouse Point Branch
    45 | regular_circ | Innsmouth Township Library System
```

That is a very basic and comprehensive circulation report.  What it lacks is detail.  From here the challenge is that this view combines different sources with different data definitions creating challenges for things like copy level information.  If the view had an optional copy level field it would be easier but it doesn't. There are ways to bring that information into a single report but at that point you have to ask yourself - do you really need one report to rule them all or would you be better off with several more specialized reports tailored to your purpose?

### <a name="discrete1"></a> Asking Discrete Questions part 1 of 2

One of the things I like about murder mysteries is the moment where the detective figures out the right question to ask and then everything falls into place.  SQL is like that.  I had two of those today where I looked at SQL written by someone else, was confused as heck and had to say, “all right, let’s back up and start at the beginning to see what is going on here.”  When you remove assumptions and start eliminating possibilities you can solve the mystery.  Like a murder mystery, writing SQL is all about identifying the questions you really want to ask.

One way this often bites newer SQL coders is that they try to get one complicated statement to do everything they want.  In murder mystery terms they ask “who killed Mr. Body” instead of who had a motive, opportunity and means.  Avoid asking one big question that results in convoluted queries.  Instead ask separate specific questions.  My philosophy is to avoid unnecessarily complicated queries like I avoid hitting my fingers with hammers.  Keeping your queries discrete, and by that I mean keeping queries for different things separate and then combining them, will help.  A lot.  I will illustrate by presenting a real world problem and different solutions to it.  In part 1 I am going to bring us through defining the problem and then in the next post I am going to present three possible solutions to it.

As our starting place we are going to look at a very simple circulation report that gives circulation IDs, patron username and titles checked out.

```sql
SELECT acirc.id, au.usrname, ssr.title
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;

id       | usrname     |                      title                       
---------+-------------+--------------------------------------------------
31086368 | 23000000012 | Now is the time to open your heart : a novel
31374411 | 23000000099 | Flywheel
31156260 | 23000000042 | Marcel the shell with shoes on : things about me
31374413 | 23000000031 | The mule
31155213 | 23000000089 | Bones don't lie
31436691 | 23000000077 | There are no bears in this bakery
31106286 | 23000000054 | The Princess of 8th Street
31155210 | 23000000037 | The jasmine moon murder
31348095 | 23000000087 | The Bletchley Circle : San Francisco
31236056 | 23000000067 | Trick or treat murder
(10 rows)
```

However, the users wanted to see stat cats with the circulations.  Now this means adding a line like this:

```sql
LEFT JOIN actor.stat_cat_entry_usr_map um ON um.target_usr = au.id
```
as well as 'um.stat\_cat\_entry' to the SELECT line.  Why?  You want a left join or you will lose patrons that don't have a stat cat entry.  They wanted to know when it was null or not Blue (both treated as not Blue) or Blue.  So condition 1 = no stat cat entry or any stat cat other than Blue and then condition 2 was the patron had as stat cat of Blue.  However, it does cause an issue as you will see ...

```sql
SELECT acirc.id, au.usrname, ssr.title, um.stat_cat_entry
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN actor.stat_cat_entry_usr_map um ON um.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;

id       |    usrname     |         title          |    stat_cat_entry    
---------+----------------+------------------------+----------------------
30852615 | ABCDEFGHI      | Baby Beluga            | Blue
30852615 | ABCDEFGHI      | Baby Beluga            | Blue
30852597 | ABCDEFGHI      | The dentist's office   | Blue
30852597 | ABCDEFGHI      | The dentist's office   | Blue
31397285 | 44440003077777 | Bringing down the Duke |
31397285 | 44440003077777 | Bringing down the Duke | Red
31397285 | 44440003077777 | Bringing down the Duke |
31397285 | 44440003077777 | Bringing down the Duke | Red
30852544 | ABCDEFGHI      | Hurty feelings         | Red
30852544 | ABCDEFGHI      | Hurty feelings         | Blue
(10 rows)
```

Understandably they didn’t like the duplicates so they tried to solve it by adding a filter for the stat cat of “AND um.stat\_cat\_entry = 'Blue'”.  This solve that problem but created another ...

```sql
SELECT acirc.id, au.usrname, ssr.title, um.stat_cat_entry
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN actor.stat_cat_entry_usr_map um ON um.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
AND um.stat_cat_entry = 'Blue'
LIMIT 10;

id        |    usrname     |                        title                        | stat_cat_entry
----------+----------------+-----------------------------------------------------+----------------
30819908  | 44440004077778 | Sing                                                | Blue
31397022  | 12340004077779 | How to do nothing : resisting the attention economy | Blue
31397048  | 12340004077779 | American dirt                                       | Blue
31397016  | 12340004077779 | Magnolia Kitchen : inspired baking with personality | Blue
31651141  | 12340004077779 | The favourite                                       | Blue
31496673  | 12340004077779 | The favourite                                       | Blue
31397063  | 12340004077779 | Last Christmas                                      | Blue
31496672  | 12340004077779 | The kitchen                                         | Blue
31397033  | 12340004077779 | Men in black. International                         | Blue
31397026  | 12340004077779 | The favourite                                       | Blue
(10 rows)
```

At this point they got frustrated and I started looking at it.  The darned if you do / darned if you don’t scenario was from counting two different things in one query, patron statistical category data and patron circulation data and combining multiple rows of one (stat cats) with single rows of another (circulations).  When things get like this it is a good practice to step back and re-evaluate assumptions, like can you break apart the logic into discrete elements?

I started by taking out the filter on the stat cat and prettying up the output by adding a case statement like this:  

```sql
SELECT acirc.id, au.usrname, ssr.title,
    CASE
        WHEN um.stat_cat_entry IS NULL THEN NULL
        WHEN um.stat_cat_entry != 'Blue' THEN NULL
        ELSE 'Blue'
    END AS stat_cat_text
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN actor.stat_cat_entry_usr_map um ON um.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;

id       |    usrname     |         title          | stat_cat_text
---------+----------------+------------------------+---------------
31397285 | 23450003077757 | Bringing down the Duke |
31397285 | 23450003077757 | Bringing down the Duke |
31397285 | 23450003077757 | Bringing down the Duke |
31397285 | 23450003077757 | Bringing down the Duke |
30852544 | AGABFLCMG      | Hurty feelings         | Blue
30852544 | AGABFLCMG      | Hurty feelings         |
31346829 | 20049000020969 | The valentine legacy   |
31346829 | 20049000020969 | The valentine legacy   |
31346829 | 20049000020969 | The valentine legacy   |
31346829 | 20049000020969 | The valentine legacy   |
(10 rows)
```

Problem - you're still getting multiple rows from stat cat entry where you really only want one.  So, when I present solutions in the next post I am going to break out the query logic for statistical categories into a discrete source from the circulation rows to remove the duplicate rows and show three different ways to do it.

### <a name="discrete2"></a> Asking Discrete Questions part 2 of 2

We are almost in 2021 but we have one thing to wrap up here first.  We left off in a minor conundrum so let's fix that.

The first thing to do is separate out the data we want on patron statistical categories into a separate table.  Here we create a TEMP table which means it only exists for the duration of our session.  I still drop it afterwards because I like to keep it tidy.  I do a left join to the table which means that each circulation will still show but we only get the entries for the 'Blue' ones.  This has the effect of both providing the data source and the filtering of the case while only giving one row per circulation.  

```sql
CREATE TEMP TABLE stat_cat_stuff AS
SELECT * FROM actor.stat_cat_entry_usr_map
WHERE stat_cat_entry = 'Blue';

SELECT acirc.id, au.usrname, ssr.title, scs.stat_cat_entry
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN stat_cat_stuff scs ON scs.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;

DROP TABLE stat_cat_stuff;
```

Step two: wash and repeat.  Here we are doing the same thing but instead of moving the logic into a TEMP table we make it into a subquery of the statement in this line:

```sql
LEFT JOIN (SELECT * FROM actor.stat_cat_entry_usr_map WHERE stat_cat_entry = 'Blue') scs ON scs.target_usr = au.id
```

and the query:

```sql
SELECT acirc.id, au.usrname, ssr.title, scs.stat_cat_entry
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN (SELECT * FROM actor.stat_cat_entry_usr_map WHERE stat_cat_entry = 'Blue') scs ON scs.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;
```

What is the functional difference?  On this scale, none that matters.  I do like keep it in one statement but it is a bit hard to read.  So, why not have it both ways?  We can, with a 'WITH' statement, also known as a CTE or Common Table Expression.  

```sql
WITH stat_cat_stuff AS (
  SELECT * FROM actor.stat_cat_entry_usr_map
  WHERE stat_cat_entry = 'Blue'
)
SELECT acirc.id, au.usrname, ssr.title, scs.stat_cat_entry
FROM actor.usr au
JOIN action.circulation acirc ON acirc.usr = au.id
JOIN asset.copy acp ON acp.id = acirc.target_copy
JOIN asset.call_number acn ON acn.id = acp.call_number
JOIN reporter.super_simple_record ssr ON ssr.id = acn.record
LEFT JOIN stat_cat_stuff scs ON scs.target_usr = au.id
WHERE acirc.xact_start BETWEEN '2020-01-01' and '2020-12-31'
LIMIT 10;
```

I like CTEs because they allow you to keep queries discrete and tidy, a win/win.

### <a name="xmlstarlet"></a> XMLStarlet - Get It Over With

My job was once described as heavily involving punching MARC records in the face.  There is definitely some truth to that.  And if you do database work with Evergreen and are looking to handle ILS specific tasks you probably are working with MARC records.  Once you load marc records into a Postgres system, probably as MARCXML, you have a lot of options of how to deal with them.  But sometimes you don't want to bother.  So let us look at a non-database way of approaching a task of parsing data out of MARC.  This clearly is not Evergreen specific but it is my blog so I can break my own rules.  

The task: I have a list of barcodes and I want to know if they are in the embedded holdings of a bunch of bibs.  I could load these into Evergreen and use oils_xpath to search with XPath.  Koha has a similar function.  But those require loading the bibs.  Instead I can just a command line tool.  Xmlstarlet exists for Windows, Mac, Linux and a few other operating systems.  

The general structure of an XMLStarlet invocation is as follows:

```
xmlstarlet [action] [parameters for action] file
```

A useful tool is to quickly just check to see what the structure of the XML file looks like.  With MARCXML is should be predictable.  

```
derleth@test:~/data$ xmlstarlet el -u bibs.xml
collection
collection/record
collection/record/controlfield
collection/record/datafield
collection/record/datafield/subfield
collection/record/leader
```


rhamby@migrator2:~/data$ xmlstarlet el -v march11.xml | grep '852' | sort | uniq -c
      1 collection/record/datafield[@tag='852' and @ind1='0' and @ind2=' ']
   2018 collection/record/datafield[@tag='852' and @ind1='0' and @ind2='0']
   1114 collection/record/datafield[@tag='852' and @ind1='1' and @ind2=' ']
   3812 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2=' ']
  18672 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2='0']



xmlstarlet el -v march11.xml | grep '852'

rhamby@migrator2:~/data$ xmlstarlet el -a march11.xml | sort | uniq -c
      1 collection
  25216 collection/record
  78949 collection/record/controlfield
  78949 collection/record/controlfield/@tag
 627560 collection/record/datafield
 627560 collection/record/datafield/@ind1
 627560 collection/record/datafield/@ind2
1163369 collection/record/datafield/subfield
1163369 collection/record/datafield/subfield/@code
 627560 collection/record/datafield/@tag
  25216 collection/record/leader
      1 collection/@xmlns

xmlstarlet [action] [one or more action parameters] [xpath statement] [file]

xmlstarlet sel -t -v 'count(//*[@tag="852"]/*[@code="p"])' march11.xml
25532

xmlstarlet sel -t -v '//*[@tag="852"]/*[@code="p"]/text()' march11.xml

xmlstarlet sel -t -v '//*[@tag="852"]/*[@code="p"]/text()' march11.xml > barcode_list_from_bib.txt

grep -o -f to_look_for.txt barcode_list_from_bib.txt

rhamby@migrator2:~/data$ grep -o -f to_look_for.txt barcode_list_from_bib.txt > found.txt
rhamby@migrator2:~/data$ wc -l found.txt
1064 found.txt
