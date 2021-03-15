[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="dotheymatch"></a> Do They Match

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

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
