
[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.md)

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

