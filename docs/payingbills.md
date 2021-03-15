[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

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

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
