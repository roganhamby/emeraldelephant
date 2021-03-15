![Azzuri](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_medium.png)

Welcome to Emerald Elephants.  The goal of this blog is to demystify some of the workings of the Evergreen ILS database which runs on PostgreSQL.  Where does the name come from?  Evergreen = Emerald, Postgres' mascot is an Elephant.  No, I don't think I'm clever but I'm a sucker for allieration.  After a few years of teaching SQL for librarians classes and periodically getting questions about some of the corners of the Evergreen database I thought, why answer questions individually when I can compile them for posterity and force Github's corporate owners to pay one hundreth  of a penny hosting some more of my text.  Any samples mentioned here will be marked in the appropriate place in the accompanying Github repo @ https://github.com/roganhamby/emeraldelephant.git whose docs/ folder also contains this blog in markdown.  But, that's enough recursion for now.

For those looking for other Evergreen resources:

* [Evergreen WIKI](https://wiki.evergreen-ils.org/doku.php)
* [Jane Sandberg's posts on Evergreen](https://sandbergja.github.io/tags.html#Evergreen-ref)

I take questions via the Github issues and via Twitter @roganhamby or email if you can find it (it's not hard).

Quick links for posts:  
* [Grokking the Relationship Between Transactions and Bills](transactions_and_bills.md) 2020-06-15   
* [Paying Off Bills part 1: The Where of It](payingbills.md#payingbills1) 2020-06-30  
* [Paying Off Bills part 2: The How of It](payingbills.md#payingbills2) 2020-07-12
* [Ancestors and Descendants](ancestors.md) 2020-08-01
* [Views and Tables in A Materialized World](materialized.md) 2020-10-06
* [Do the Bibs Match the Copies](dotheymatch.md) 2020-10-09
* [Circulation Basics, Action.Circulation](circbasics.md#circbasics1) 2020-10-19
* [Circulation Basics, Tables and Views](circbasics.md#circbasics2) 2020-10-25
* [Circulation Basics, Reports](circbasics.md#circbasics3) 2020-11-30
* [Asking Discrete Questions 1 of 2](discrete.md#discrete1) 2020-12-28
* [Asking Discrete Questions 2 of 2](discrete.md#discrete2) 2020-12-31
* [XMLStarlet - Get It Over With](#xmlstarlet) 2021-03-14

### <a name="xmlstarlet"></a> XMLStarlet - Get It Over With

My job was once described as heavily involving punching MARC records in the face.  There is definitely some truth to that.  And if you do database work with Evergreen and are looking to handle ILS specific tasks you probably are working with MARC records.  Once you load marc records into a Postgres system, probably as MARCXML, you have a lot of options of how to deal with them.  But sometimes you don't want to bother.  So let us look at a non-database way of approaching a task of parsing data out of MARC.  This clearly is not Evergreen specific but it is my blog so I can break my own rules.  

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


rhamby@migrator2:~/data$ xmlstarlet el -v march11.xml | grep '852' | sort | uniq -c
      1 collection/record/datafield[@tag='852' and @ind1='0' and @ind2=' ']
   2018 collection/record/datafield[@tag='852' and @ind1='0' and @ind2='0']
   1114 collection/record/datafield[@tag='852' and @ind1='1' and @ind2=' ']
   3812 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2=' ']
  18672 collection/record/datafield[@tag='852' and @ind1=' ' and @ind2='0']



xmlstarlet el -v march11.xml | grep '852'

rhamby@migrator2:~/data$ xmlstarlet el -a march11.xml | sort | uniq -c
      1 collection
  25216 collection/record
  78949 collection/record/controlfield
  78949 collection/record/controlfield/@tag
 627560 collection/record/datafield
 627560 collection/record/datafield/@ind1
 627560 collection/record/datafield/@ind2
1163369 collection/record/datafield/subfield
1163369 collection/record/datafield/subfield/@code
 627560 collection/record/datafield/@tag
  25216 collection/record/leader
      1 collection/@xmlns

xmlstarlet [action] [one or more action parameters] [xpath statement] [file]

xmlstarlet sel -t -v 'count(//*[@tag="852"]/*[@code="p"])' march11.xml
25532

xmlstarlet sel -t -v '//*[@tag="852"]/*[@code="p"]/text()' march11.xml

xmlstarlet sel -t -v '//*[@tag="852"]/*[@code="p"]/text()' march11.xml > barcode_list_from_bib.txt

grep -o -f to_look_for.txt barcode_list_from_bib.txt

rhamby@migrator2:~/data$ grep -o -f to_look_for.txt barcode_list_from_bib.txt > found.txt
rhamby@migrator2:~/data$ wc -l found.txt
1064 found.txt
