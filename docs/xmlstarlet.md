
[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="xmlstarlet"></a> XMLStarlet - Get It Over With

My job was once described as heavily involving punching MARC records in the face.  There is definitely some truth to that.  And if you do database work with Evergreen and are looking to handle ILS specific tasks you probably are working with MARC records.  Once you load MARC records into a Postgres system, probably as MARCXML, you have a lot of options of how to deal with them.  But sometimes you don't want to bother.  So let us look at a non-database way of approaching a task of parsing data out of MARC.  This clearly is not Evergreen specific but it is my blog so I can break my own rules.  

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

You can also look to see exactly what variations of nodes with values exist but it will be such a long list, for every kind of MARC tag that it isn't as useful.  But you can verify that a tag, say holdings in the 852 do exist and with what indicators pretty quickly.

```
derleth@test:~/data$ xmlstarlet el -v bibs.xml | grep '852' | sort | uniq -c
      1 collection/record/datafield[@tag='852' and @ind1='0' and @ind2=' ']
   2018 collection/record/datafield[@tag='852' and @ind1='0' and @ind2='0']
   1114 collection/record/datafield[@tag='852' and @ind1='1' and @ind2=' ']
   3812 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2=' ']
  18672 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2='0']
```

Now let's get into the practical.  The 'sel' command allows us to search with xpath in the MARCXML file.  That means we can use a count to quickly find out how many 852 tags there are with a subfield p, which in this case means holdings with barcodes.

```
xmlstarlet sel -t -v 'count(//*[@tag="852"]/*[@code="p"])' bibs.xml
25532
```

We can then get the actual values and send them to a file:

```
xmlstarlet sel -t -v '//*[@tag="852"]/*[@code="p"]/text()' bibs.xml > barcode_list_from_bib.txt
```

Since we have an existing list then we can just use grep's built in file comparision:

```
grep -o -f to_look_for.txt barcode_list_from_bib.txt
```

If you want it in a file send that there instead of the screen (probably more useful that way).

```
grep -o -f to_look_for.txt barcode_list_from_bib.txt > found.txt
rhamby@migrator2:~/data$ wc -l found.txt
1064 found.txt
```

And that is it.  Until next time, be kind to elephants, they are awesome creatures.

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
