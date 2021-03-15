[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

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


[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
