[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

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

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
