[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)


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

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
