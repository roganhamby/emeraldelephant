![Azzuri](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_medium.png)

Welcome to Emerald Elephants.  The goal of this blog is to demystify some of the workings of the Evergreen ILS database which runs on PostgreSQL.  Where does the name come from?  Evergreen = Emerald, Postgres' mascot is an Elephant.  No, I don't think I'm clever but I'm a sucker for allieration.  After a few years of teaching SQL for librarians classes and periodically getting questions about some of the corners of the Evergreen database I thought, why answer questions individually when I can compile them for posterity and force Github's corporate owners to pay one hundreth  of a penny hosting some more of my text.  Any samples mentioned here will be marked in the appropriate place in the accompanying Github repo @ https://github.com/roganhamby/emeraldelephant.git whose docs/ folder also contains this blog in markdown.  But, that's enough recursion for now.

For those looking for other Evergreen resources:

* [Evergreen WIKI](https://wiki.evergreen-ils.org/doku.php)
* [Jane Sandberg's posts on Evergreen](https://sandbergja.github.io/tags.html#Evergreen-ref) 

I take questions via the Github issues and via Twitter @roganhamby or email if you can find it (it's not hard).

Quick links for posts:  
* [Grokking the Relationship Between Transactions and Bills](#grokkingbills) 2020-06-15   
* [Paying Off Bills part 1: The Where of It](#payingbills1) 2020-06-30  
* [Paying Off Bills part 2: The How of It](#payingbills2) 2020-07-12
* [Ancestors and Descendants](#ancdesc) 2020-08-01
* [Views and Tables in A Materialized World](#materialized) 2020-10-06
* [Do the Bibs Match the Copies](doesthebibmatchthecopies) 2020-10-09

### <a name="grokkingbills"></a> Grokking the Relationship Between Transactions and Bills

"A nickel ain't worth a dime anymore." -- Yogi Berra

A question wasn't directed at me but I was around at the time.  The question was, "so..... are billing IDs the same as circ IDs, and I have just never noticed before, or is that just test server data?"  It is a valid question.  The very nature of how test server is built can lead to co-oincidences in id sequences that are less likely in production data.

My reply: "they are, the circ table is actually a child table of bills."

That's a terse reply and the relationship between circulations and bills is something that is often confusing to database delvers in Evergreen.   Here is a longer answer.  

#### Table Inheritance 

Critical to how the ids are shared is the concept of table inheritance.  When I have taught this concept in classes I have found people sometimes struggle with understanding it so bear with me.  If you feel comfortable with it skip ahead to the heading 'money.billable_xact'.  I think that many who have struggled with the concept of inheritance think 'it has to be more complicated than that.'  But it's not.  How it works is simple, one table is a subset of another.  It works like this, let's say you create a table to represent creatures.  Oh, quick head's up, I'm going to use some SQL to illustrate the point but it's introductory level SQL.  If you're not familiar with it I think just following the meaning of the key terms in English will still get to the point.

```sql
    CREATE TABLE animals (	
        name TEXT,
        age INTEGER
    );

    CREATE TABLE snakes (
        poisonous BOOLEAN NOT NULL,
        length INTEGER 
    ) INHERITS (animals);
```

What does this do?  Imagine you have a few million rows to put into the animals table.  Sure, you could put the columns in for the reptiles but then you're wasting a whole bunch of space for all those non-snakes in there.  Plus it would seem a bit odd to have to mark a panda bear as non-poisonous since that column doesn't allow NULL.  At least I'm assuming pandas are non-poisonous.  Let's go with that.

The important thing to understand is that by having table 'snakes' inherit from table 'animals' those columns from 'animals' - name and age are present in snakes as well.  The columns aren't copied but an actual subset so when I do this :

```sql
    INSERT INTO snakes (name,poisonous) VALUES ('brown burrower',FALSE);
```

This is what appears if you select from each table respectively:

```sql
    rhamby=# select * from animals;  
    name | brown burrower
    age  |

    rhamby=# select * from snakes; 
    name      | brown burrower
    age       |
    poisonous | f  
    length    |
```

And if I update the age on snakes:

```sql
    rhamby=# update snakes set age = 2 where name = 'brown burrower';
    UPDATE 1

    rhamby=# select * from animals;
    name | brown burrower
    age  | 2

    rhamby=# select * from snakes;
    name      | brown burrower
    age       | 2
    poisonous | f
    length    |
```

So, to reiterate one table is a subset of the other with the inherited columns shared between the tables.  All good?  Now, let's head into the trees.

#### money.billable_xact

Let's do a \d+ in Postgres on the action.circulation table.  There is a whole lot of output but the part we want is at the very, very end: 

`Inherits: money.billable_xac`  

If you're thinking 'hey, you were just talking about that' you are clearly a person of discernment and far sight.  The action.circulation table inherits money.billable_xact.  Why?  money.billable_xact has only five columns: id, usr, xact_start, xact_finish, and unrecovered.  All of these are properites of circulations as well, hence why it makes sense for them to be shared columns since any circulation could also have bills.  You could tie them together in other ways but this manner is effecient, simple and elegant.  Since every action.circulation entry needs each of the values in money.billable_xact no resoures are wasted even if a circulation doesn't have a bill.

Now let's look more about how money.billable\_xact relates to bills.  Let us turn our probing to details on money.billable_xact. It has three child tables:

action.circulation  
booking.reservation  
money.grocery  

We already knew that action.circulation was a child table.  Bookings are a sort of equivalent to a circulation so that makes sense.  It may not seem as obvious but groceries are an equivalent as well.  Money.grocery is a bit confusing to some folks but the term means miscellaneous here more than looking to fill the pantry.  Since a grocery bill is a miscellaneious bill not attached to a circulation or booking reservation the entry in this table acts as a transaction as well as bill.  So, the common element of all the child tables of money.billable\_xact is that they track transactions and while billable_xact is a jumping off point for any bills associated with transactions.  Money.grocery has that bill built into transactions because it is the most effecient way to do it for those as they are by definition both transaction and bill.  Circulations are divided from their bills since they may not have any.

Finally, let's look at bills.  Groceries are their own bills but everything else is tracked in money.billing.  The money.billing table has a very important column: xact.  This links to money.billable\_xact's id column.   Everytime you have a bill on a circulation it becomes an entry in money.billing and they are all tied together to show they are all bills on that one circulation by connecting back to money.billable_xact.  How does that connect to the circulation?  Because of action.circulation inheriting money.billable\_xact, action.circulation ids are money.billable\_xact ids so when you see the xact value in a money.billing entry - it is also a circulation id or a money.grocery id or a booking.reservation id.  Convenient.

![money.billable_xact and money.billing relationship](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/images/moneybillablemoneyxact.png)

To illustrate the point let us imagine a book that has been checked out and generated some bills.  I check out a copy of Jonathan Gash's "The Judas Pair" - the very first checkout in this copy of Evergreen.  The causes an entry with id of 1 in money.billable\_xact and an entry into action.circulation which is also id 1 because it inherited money.billable\_xact's.  Once the book goes overdue every day a new entry goes into money.billing and every one has a value of xact that is 1, all linking back to that one entry in money.billable_xact id 1, circulation id 1.  In database terminology the billing xact column is a foreign key linking to billable\_xact.  So, money.billable\_xact's relationship to money.billing is a one to many relationship.  Any future lost fees, lost charges and so on do the same.  Thus, we can have any number of bills, always linking to one table of transactions  with child tables representing their unique needs without any waste of space in the database.

So in summation to understand how transactions connect to circulations those two ideas are critical: how circulations / grocery bills / booking reservations are child tables of money.billable\_xact and the one to many relationship between money.billable\_xact and money.billing.

Now, paying for those bills, that is a future entry in it's own right.

### <a name="payingbills1"></a> Paying Off Bills part 1: The Where of It

The last time I posted I wrote about how bills worked.  I hadn’t planned to write about how payments worked next but, this was posted on the public open-ils list:

“One of our library systems is wanting to do an amnesty project, and clear all bills over 5 yrs old.  Anyone have a script we can start with? Or any gotchas we should know about?”

They aren’t alone.  I’ve done more clearing of bills in the last month than I think I had in the previous five years combined.  There is something in the air with libraries re-opening amidst the COVID-19 pandemic that people want to lower barriers to using the library.  While I certainly don’t celebrate the pandemic, I’m a huge fan of no-fines policies and forgiveness projects so I’ve cheered this on.  And as libraries look at these tasks they are running reports and making decisions.  Voiding a few dozen fines?  The staff client has you covered.  Removing tens or hundreds of thousands?  Uh… let’s talk about scripts for that.  But first, we have to understand how things are laid out. 

So, we have two components to this: 1) how payments are structured in the database and 2) how to use them to our benefit.  Due to the amount of content, I’m going to break the blog post up into two parts with this first part being how the payments are structured in the database.  Then in my next post, I’ll get into how you can approach doing something with them in the database.

Here are the topics we will cover in this post:  
How bills are structured (quick refresher, look at the 2020-06-16 article for more)  
The kinds of payments  
How payments are applied to billings (hint: they're not, mostly)

In part two we will cover:  
Ways to actually pay the bills off  
Related concerns with closing transactions 

Let’s get stuck in.

#### How Bills Are Structured 

Since I covered this in detail in ‘2020-06-16 Groking the Relationship Between Transactions and Bills’ this will be the expedited version.  The money.billable\_xact table has an id column that is shared with its child tables of money.grocery, action.circulation and booking.reservation.  So, each kind of transaction (grocery, circ, booking) has an id in money.billable\_xact that is used as the connection point for billings.  All of those combined billings is the bill for that circ or whatever.  Think of many days worth of fines in money.billing linking back to one circulation in action.circulation which shares its id with its parent table money.billable_xact.  

#### The Kinds of Payments

Evergreen’s payment structure is … special.  There have been long-standing discussions about changing it.  However, although it’s inflexible it does work and any awkwardness it has is largely invisible to patrons and staff.  So, I don’t think it’s likely to change soon unless some dev really becomes develops a compulsion and plenty of free time.  

If you do a table listing in the money schema you’ll see ten payment tables: 

bnm\_desk\_payment  
bnm\_payment  
cash\_payment   
check\_payment  
credit\_card\_payment  
credit\_payment  
forgive\_payment    
goods\_payment  
payment  
work\_payment

Let’s look at how these all tie together and throw one more into the mix, account_adjustment because although it’s not technically for payments per se we really can’t talk about adjusting bills comprehensively without talking about it too.  

![Payment Tables](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/images/payments.png)

Each of these lines from the table name indicates a child / parent table relationship where the child table is inheriting rows from the parent table.  So, our base table that everything else comes from is money.payment.  Every other payment type table has those rows in common.

Then in the rose-colored header after that is money.bnm\_payment.  BNM here means ‘brick n mortar.’  Presumably, the original intent was that there would be different child tables from money.payment, some to represent payments made at physical locations, and some to represent other avenues of transaction.  However, that hasn’t happened and all other payment tables descend from money.bnm\_payment so every other payment table also has it’s two additional rows of the amount collected and an accepting user.  

A total of nine tables descend from money.bnm\_payment but let’s start with the four that have no additional rows: credit\_payment, forgive\_payment, goods\_payment and work\_payment.  Since these have no additional rows they simply act as ways of saying “hey, if the row is in here it’s this kind of payment.”  You could accomplish the same thing with a single table and a column with values of ‘credit’, ‘forgive’, ‘goods’, and ‘work.’  In practice the forgive payment table is the most commonly used, used whenever a payment is made to forgive a part of a fine.  We will talk about the difference between voiding and forgiving later.  Goods is sometimes used, especially for things like canned food drives to remove fines.  

Underneath those four on the chart is account\_adjustment.  We will talk about it more when we talk about paying bills because it is unique in that it links to specific billings.  More on that aspect later as well.

Finally, the last table child of money.bnm\_payment is money.bnm\_desk\_payment.  This provides an optional value for linking to a cash drawer, also known as the registered workstation.  This is not as useful in practice as it might sound.  So, the practical differentiation between money.bnm\_desk\_payment and it’s parent is not large in practice and since the child table is inheriting everything from its parent you can just use bnm\_desk\_payment and ignore that bnm\_payment even exists.  

Then bnm\_desk\_payment has three children of its own: cash_payment, credit_card_payment and check_payment.  These, along with forgiven, account for the most common types of payments made.  

#### How Payments Are Applied to Billings

So, how are payments applied to billings?  I already hinted at the answer when I said, tongue firmly in cheek, above that they’re not.  And they aren’t.  Remember that the parent table of all billable transactions is money.billable_xact and it’s ‘id’ column is the ‘xact’ referenced by money.payment.

So, if that wasn’t clear: 

money.payment.xact = money.billable_xact.id 

Now, billings exist in the money.billing table.  So, with one exception to mention in a minute, no payments ever reference a specific billing, they only reference the transactions.  So, when you make a credit card payment, a check payment, forgive a fine, and so on you aren’t actually paying off a billing, you are simply applying a payment to offset an amount.  That is why you can have a total bill like this:

The Novel  
billing one: overdue 10 cents  
billing two: overdue 10 cents  
billing three: overdue 10 cents  
billing four: overdue 10 cents  
billing five: overdue 10 cents  
Lost replacement cost: $10.00  
Total: $10.50

I know what you’re going to say, going to lost after five days is rough but it’s a rough world folks.  Anyway, I now owe $10.50 on this book.  I go up to the desk and have $0.37 in my pocket so I give it to the library to pay off this small part of the bill.  They take it as a cash payment.  Now we have:

The Novel  
billing one: overdue 10 cents  
billing two: overdue 10 cents  
billing three: overdue 10 cents  
billing four: overdue 10 cents  
billing five: overdue 10 cents  
Lost replacement cost: $10.00  
cash payment: 0.37  
Total: $10.13  

Note that no billings are removed or closed out in any way and no billings match that $0.37.  Eventually, when the payments equal the billings it is closed out.  However, if you go a penny over you end up with a negative bill where the library now owes the patron (at least in bookkeeping theory - the reality may be quite different).  

Account adjustments are different because they do connect to specific billings and track per billing how much is paid.  One billing can have multiple account adjustments.  These were introduced when the Adjust to Zero function in the staff client was added.  One of the advantages of this is that if you have a negative bill, where more is paid than owed, you can logically zero it out with an account adjustment instead of applying a negative payment.  Really, negative payments don’t make sense, do they? 

But I’m not going to digress on the historical things that have been done to pay bills or bizarre things that could be done still.  Instead, next time we will talk about some best practices script wise as well as provide a peek as to what happens when you do these things in the staff client.  

The next post will be Paying Off Bills part 2: The How of It.

### <a name="payingbills2"></a> Paying Off Bills part 2: The How of It

Welcome to part 2 of how to pay off bills in the Evergreen database.  Last time we went over the background information of the tables and now we are going to actually wipe some bills out.  Again, this is mostly useful for bulk removal of bills.  If you are looking at dozens of fines, just do it in the staff client.

#### Voiding vs Paying Bills

First let’s talk about the ways of getting rid of bills and what they mean.

Voiding a single billing is simple.

```sql
    UPDATE money.billing SET voided = TRUE, voider = 1 WHERE id = 1;
```

Of course, this makes some big assumptions.  One of those assumptions is that you want to resolve billings and not whole transactions.  Another assumption is that you don’t care about what kind of billing the billing is.  But if you do care …  For example, maybe you want to remove all late fees on a specific circulation but not lost fees and things like that.  You can do that like this:

```sql
    UPDATE money.billing 
    SET voided = TRUE, voider = 1 
    WHERE xact = 123 AND btype = (SELECT id FROM config.billing_type WHERE name = ‘Overdue Materials’);
```

However, all of this assumes no payments have been made on the transaction.  You really want to check that first, like this:  

```sql
    select * from money.billable_xact_summary where id = 4;;
    -[ RECORD 1 ]-----+------------------------------
    id                | 4
    usr               | 10676
    xact_start        | 2011-06-07 17:00:00-04
    xact_finish       | 2011-12-20 15:42:08.144589-05
    total_paid        | 4.00
    last_payment_ts   | 2011-12-20 15:42:08.144589-05
    last_payment_note | 
    last_payment_type | credit_card_payment
    total_owed        | 4.20
    last_billing_ts   | 2011-07-17 00:59:59-04
    last_billing_note | System Generated Overdue Fine
    last_billing_type | Overdue materials
    balance_owed      | 0.20
    xact_type         | circulation
```

Here we have a record where all the bills were overdue fines that added up to $4.20.  All but $0.20 were paid on it.  If we did this:

```sql
    UPDATE money.billing SET voided = TRUE, voider = 1 
    WHERE xact = 4 
    AND btype = (SELECT id FROM config.billing_type WHERE name = ‘Overdue Materials’);
```

We would end up with a negative balance of $4.00 since the bills would be voided but $4.00 in payments were already on the account.  This is no bueno.  So, voiding is only useful per billing when the total left won’t result in a negative billing.  I use money.billable\_xact\_summary and check to see if balance\_owed is equal to total_owed.  If it’s not then some payments have already been made and I skip over it.  I only use voiding on bills where they haven’t been touched yet.  

So why use voiding at all if you’re going to have to do at least one level of work past that?  It’s clean.  The actual payment methods involve adding additional rows to other tables.  The tablespace probably won’t matter that much in the long run but it’s more elegant and easier to read from a forensic standpoint if you ever want to go back and look.  

However, I’m not an unreserved fan of it.  On some level, I feel like bills should only be voided when they never should have been issued in the first place.  A good example is a book’s late fines when the circulation staff doesn’t catch that it didn’t scan on check in and it’s on the shelf while accumulating fines.  However, that is a distinction I am in the far minority in caring about. It’s not implicit in the software itself and I’ve not seen a library that has cared about in practice yet, including my own when I was a circulation manager.  Sometimes you have to admit you’re Canute and swinging a sword at the waves just isn’t going to be useful.

So, if you want to use voiding to get rid of fines, I think that is fine.  It is also fine to avoid it and simply use payments and adjustments.  However, you can’t only use voiding as voiding can not address the issue of when a payment has been made.  For that you will have to use payments and, maybe adjustments.  
Paying the Price

Let’s go back and look at some bill summaries again.

```sql
    select id, balance_owed from money.billable_xact_summary where id < 10;
     id | balance_owed 
    ----+--------------
      4 |         0.20
      5 |         0.00
      6 |        -0.10
      7 |         1.20
```

With these four bills we have four very different scenarios.  #4 has been partially paid down to twenty cents, #5 has been paid off, #6 has been overpaid and is now a negative balance and #7 hasn’t had any payments at all.

One simple way to handle all of these is :  

```sql
INSERT INTO money.forgive_payment (xact, amount, note, amount, amount_collected, accepting_usr) 
SELECT id, balance_owed * -1, ‘forgiven due to fresh start project’, balance_owed * -1, 1 
FROM money.billable_xact_summary 
WHERE id < 10;
```

Technically this works for all of them but I have two quibbles.  One, it unnecessarily creates a new entry of 0.00 for #5 which can create a lot of junk rows in the database.  So, I would recommend a ‘WHERE balance_woed != 0’ filter for that.  My other issue has to do with a negative balance.  Technically you have made a negative payment to balance a negative bill which from a forensic standpoint is bad accounting.  It works because … it’s math but it’s not pretty.  I would recommend this:

```sql
INSERT INTO money.forgive_payment (xact, amount, note, amount, amount_collected, accepting_usr) 
SELECT id, balance_owed * -1, ‘forgiven due to fresh start project’, balance_owed * -1, 1 
FROM money.billable_xact_summary WHERE id < 10 AND balance_owed > 0;
```

This will avoid both zero amounts and ugly negative payments.  Now, here I used the forgive payments because the context is this project but you could do any kind of payments like this though honestly, it’s hard to imagine situations where you would want to do bulk payments of other kinds in the database.  

#### Putting it Into Practice 

Let’s take what we’ve talked about so far and put it into some context.  Let us start by creating a schema to hold our work in.  

```sql
CREATE SCHEMA fine_cleanup;
```

Let us imagine we are working with a library that want to remove all past circulation fines related to any items that are not currently listed as having the item status of LOST.  They also want to clean up all past grocery and booking fines.  Now, if they had been concerned about only circulations we could have just used that table but instead, we are going to use a combination of the money.billable_xact and the action.circulation tables.  

```sql
DROP TABLE IF EXISTS fine_cleanup.loans_to_cleanup;
CREATE TABLE fine_cleanup.nuke_bills AS
SELECT mb.id AS xact_id, mbxs.balance_owed, mbxs.total_paid 
FROM money.billable_xact mb 
JOIN money.billable_xact_summary mbxs ON mbxs.id = mb.id 
WHERE mb.xact_finish IS NULL AND mbxs.balance_owed != 0;
```

Depending on the size of your data this stage could take a while since you’re accessing large tables and views but that is why it’s important to filter out the ones where the transactions have been closed already by looking at the xact_finish column.  On a smaller system you could grab all of them and filter later but on a large one … yeah, filter upfront.

Now, let’s get rid of the ones that are attached to items that are now LOST.  Of course, the library should be warned that the fines on a given account may have come about from a previous circulation to when the item was lost but in this case let’s imagine they say that’s fine, they will get a report at the end and handle those by hand.  You might also be able to do some filtering by item status changed time versus transaction start but for this purpose we will keep it simple. 

Why not build our table off circs to begin with?  Because we wanted to get groceries and booking bills as well which using money.billable_xact gives us.  

```sql
DELETE FROM fine_cleanup.nuke_bills 
WHERE xact_id IN (SELECT id FROM action.circulation WHERE target_copy IN (SELECT id FROM asset.copy WHERE status = (SELECT id FROM config.copy_status WHERE name = ‘Lost’)));
```

First, let’s void the bills that have no payments: 

```sql
UPDATE money.billing 
SET voided = TRUE, voider = 1 
WHERE xact IN (SELECT xact_id FROM fine_cleanup.nuke_bills 
WHERE balance_owed > 0 AND total_paid = 0);
```

This is like the previous voiding statement but we are now grabbing everything from the table we made listing bills and saying only give them to be if the balance is postiive and no payments have been made.  We will use a forgive payment to take care of those partial bills like this:

```sql
    INSERT INTO money.forgive_payment (xact, amount, note, amount, amount_collected, accepting_usr) 
    SELECT id, balance_owed * -1, ‘forgiven due to fresh start project’, balance_owed * -1, 1 
    FROM fine_cleanup.nuke_bills 
    WHERE balance_owed > 0 AND total_paid > 0;
```

At this point you could be done if you’d changed that last line to :

    WHERE balance_owed != 0 AND total_paid > 0;

Then the negative balances would be caught too but you would have those ugly negative payments.  That brings us to adjusting bills …

#### Adjusting Bills 

Adjusting Bills works great for negative balances since it’s not a payment but an adjustment and targets specific billings.  So, why not use them for everything?  You can but I find they are a lot more complicated for both front line staff and back end people to follow if they have to look at an account in the future and figure out what happened.  Thus, I reserve them for when I need them.  Now there are three ways you can adjust them.

1) You can write a SRFSH script and send them through the adjust to zero function.  But, this isn’t a SRFSH blog.  
2) You could open the staff client and manually adjust to zero.  Honestly, if it was me as the admin at a local library and there were only a few dozen negative bills to adjust to zero this is what I’d do.  
3) But this is an Evergreen SQL blog so let us talk about the third option - scripting it in SQL.

I’m probably going to want to sue some logic that is bit beyond what a single statement easily gives me so I’m probably going to use a DO loop.  This will allow me to setup logic to loop through the individual billings.  There is a lot going on here so lines that start with two dashes will be comments to help explain the code.

```sql
    DO $$
        DECLARE
        transaction_id   BIGINT;
        billing_id BIGINT;
        descending_payment         NUMERIC;
        bill_row                   money.billing%ROWTYPE; 
        bxs                        money.billable_xact_summary%ROWTYPE;
        transaction_amount_owed    NUMERIC DEFAULT 0.0;
    BEGIN
        -- grab transactions one at a time that match our criteria 
        FOR transaction_id IN SELECT xact_id FROM  fine_cleanup.nuke_bills WHERE balance_owed < 0 LOOP
        -- we grab the grow of data and set a descending_payment variable which
        -- we will use to track to see how much is left to remove at any point 
            SELECT * FROM money.billable_xact_summary WHERE id = transaction_id INTO bxs;
            descending_payment := bxs.total_paid;
            transaction_amount_owed := bxs.balance_owed;
            -- now loop through each billing in the transaction
           FOR billing_id IN SELECT id FROM money.billing WHERE xact = transaction_id AND voided = FALSE ORDER BY amount DESC LOOP
                -- grab the info about the billing 
                SELECT * FROM money.billing WHERE id = billing_id INTO bill_row;
                -- beyond this point we are tracking how much is owed total and and making billing by billing adjustments 
                IF bill_row.amount = descending_payment THEN
                    descending_payment := 0;
                ELSIF bill_row.amount > descending_payment THEN
                    INSERT INTO money.account_adjustment (xact,amount,note,amount_collected,accepting_usr,billing)
                    VALUES (transaction_id,(bill_row.amount - descending_payment),payment_note,(bill_row.amount - descending_payment),staff,bill_row.id);
                    descending_payment := 0;
                ELSIF bill_row.amount < descending_payment THEN
                    descending_payment := descending_payment - bill_row.amount;
                END IF;
             END LOOP;
            IF descending_payment > 0 THEN
                INSERT INTO money.billing (xact,amount,billing_type,btype,note) VALUES
                (transaction_id,descending_payment,'Misc',101,'bill added to zero out account');
            END IF;
        END LOOP;
    END $$;
```

Note that the above code is adapted from something else I’ve done and untested.  I’m providing it as an illustration of the logic involved.  Indeed, to return to an above point, if you’re talking about dozens of these, giving a report to circ staff to adjust to zero the negative balances is perfectly legitimate.  

#### Finishing the Transactions 

After the actual billings are paid out there are other things you will want to think about.  Setting xact\_finish should definitely be part of your scripts.  Normally when bills are paid off the staff client takes care of that but since you’re making payments and voiding things manually you will want to close the transactions out or they will appear as open in the staff client and confuse staff.  You can do that by adjusting the xact\_finish of money.billable\_xact and setting it to NOW().

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

### <a name="doesthebibmatchthecopies"></a> Do the Bibs Match the Copies

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

