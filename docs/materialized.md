[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="materialized"></a> Materialized Tables

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

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
