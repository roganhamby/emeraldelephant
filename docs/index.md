![Azzuri](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_medium.png)

Welcome to Emerald Elephants.  The goal of this blog is to demystify some of the workings of the Evergreen ILS database which runs on PostgreSQL.  Where does the name come from?  Evergreen = Emerald, Postgres' mascot is an Elephant.  No, I don't think I'm clever but I'm a sucker for allieration.  After a few years of teaching SQL for librarians classes and periodically getting questions about some of the corners of the Evergreen database I thought, why answer questions individually when I can compile them for posterity and force Github's corporate owners to pay one hundreth  of a penny hosting some more of my text.  Any samples mentioned here will be marked in the appropriate place in the accompanying Github repo @ https://github.com/roganhamby/emeraldelephant.git whose docs/ folder also contains this blog in markdown.  But, that's enough recursion for now.

I take questions via the Github issues and via Twitter @roganhamby or email if you can find it (it's not hard).

Quick links for posts:  
 [Grokking the Relationship Between Transactions and Bills 2020-06-15](#grokkingbills)   
 [Paying Off Bills part 1: The Where of It](#payingbills1)

### <a name="grokkingbills"></a> Grokking the Relationship Between Transactions and Bills

"A nickel ain't worth a dime anymore." -- Yogi Berra

A question wasn't directed at me but I was around at the time.  The question was, "so..... are billing IDs the same as circ IDs, and I have just never noticed before, or is that just test server data?"  It is a valid question.  The very nature of how test server is built can lead to co-oincidences in id sequences that are less likely in production data.

My reply: "they are, the circ table is actually a child table of bills."

That's a terse reply and the relationship between circulations and bills is something that is often confusing to database delvers in Evergreen.   Here is a longer answer.  

#### Table Inheritance 

Critical to how the ids are shared is the concept of table inheritance.  When I have taught this concept in classes I have found people sometimes struggle with understanding it so bear with me.  If you feel comfortable with it skip ahead to the heading 'money.billable_xact'.  I think that many who have struggled with the concept of inheritance think 'it has to be more complicated than that.'  But it's not.  How it works is simple, one table is a subset of another.  It works like this, let's say you create a table to represent creatures.  Oh, quick head's up, I'm going to use some SQL to illustrate the point but it's introductory level SQL.  If you're not familiar with it I think just following the meaning of the key terms in English will still get to the point.

    CREATE TABLE animals (	
        name TEXT,
        age INTEGER
    );

    CREATE TABLE snakes (
        poisonous BOOLEAN NOT NULL,
        length INTEGER 
    ) INHERITS (animals);

What does this do?  Imagine you have a few million rows to put into the animals table.  Sure, you could put the columns in for the reptiles but then you're wasting a whole bunch of space for all those non-snakes in there.  Plus it would seem a bit odd to have to mark a panda bear as non-poisonous since that column doesn't allow NULL.  At least I'm assuming pandas are non-poisonous.  Let's go with that.

The important thing to understand is that by having table 'snakes' inherit from table 'animals' those columns from 'animals' - name and age are present in snakes as well.  The columns aren't copied but an actual subset so when I do this :

    INSERT INTO snakes (name,poisonous) VALUES ('brown burrower',FALSE);

This is what appears if you select from each table respectively:

    rhamby=# select * from animals;  
    name | brown burrower
    age  |

    rhamby=# select * from snakes; 
    name      | brown burrower
    age       |
    poisonous | f  
    length    |

And if I update the age on snakes:

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
