[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="loadcsv"></a> Loading CSVs with a Bash Script

The last 18 months have flown by which is my only explanation for why I have been neglectful in updating the repo so I thought I would do penance by doing two posts today. In my last post I shared the kind of simple utility that makes up a persistent part of my workflows. In this post I will continue the trend, though instead of being something to run sql files it creates them.

This script does one very simple thing: it takes a CSV file and creates a SQL file for loading it into a table. The script is named create_from_csv. Like the last one, this script isn't going to win awards but is awfully useful.

```console
#!/bin/bash
# $1 = file name without the csv 
# $2 = schema name 
head -n1 $1.csv | tr '[:upper:]' '[:lower:]' | tr ',' '\n' > create_$1.sql
sed -i 's/^[ \t]*//;s/[ \t]*$//' create_$1.sql
sed -i 's/ /_/g' create_$1.sql
sed -i 's/"//g' create_$1.sql
sed -i 's/^/,l_/' create_$1.sql
sed -i 's/$/ TEXT/' create_$1.sql
sed -i '1 s/,//' create_$1.sql
sed -i "1 i\\CREATE TABLE $2.$1 (" create_$1.sql
echo ");" >> create_$1.sql
echo "\\copy $2.$1 FROM $1.csv CSV HEADER;" >> create_$1.sql
```

Say, you have a CSV file that looks like this:

```
first name,middle name,family name
Jamie,Patrick,Hyneman
Adam,,Savage
Kari,Elizabeth,Byron
```

The file is named this.csv.  Now I do this: 

```console
create_from_csv this my_project
```

The arguments are the name of the csv file without the csv extension and the schema name for your project. The result is a file called create_this.sql. The contents look like this:


```sql
CREATE TABLE my_project.this (
l_first_name TEXT
,l_middle_name TEXT
,l_family_name TEXT
);
\copy my_project.this FROM this.csv CSV HEADER;
```

The result is not only creating a table definition from a header and loading the file with minimal effort but it can be easily scripted into loops to work with many files. In case you're wondering, the 'l_' convention is one I use to indicate it is a legacy data field and I never alter those. I reserve that for fields I start with 'x_'. If you have files with a different delimiter just replace the commas with the approriate delimiter. Even better you could script it to take the delimiter as a parameter, which I will probably do at some point.

Now, what if you want to do a child table using inheritance as I've talked about elsewhere. Well, I'm glad you asked!

```console
#!/bin/bash
# $1 = file name without the csv 
# $2 = schema name
# $3 = production table
tblname=$3
tblname=${tblname//./_}
parenttbl=$tblname
l="_legacy"
tblname="$tblname$l"
#used for the list of fields to copy to 
copylist=$(head -n1 $1.csv)
copylist=${copylist//,/,l_}
copylist="l_$copylist"
head -n1 $1.csv | tr '[:upper:]' '[:lower:]' | tr ',' '\n' > create_$1.sql
sed -i 's/^[ \t]*//;s/[ \t]*$//' create_$1.sql
sed -i 's/ /_/g' create_$1.sql
sed -i 's/"//g' create_$1.sql
sed -i "s/^/    ,l_/" create_$1.sql
sed -i 's/$/ TEXT/' create_$1.sql
sed -i '1 s/,//' create_$1.sql
sed -i "1 i\\CREATE TABLE $2.$tblname (" create_$1.sql
echo ") INHERITS ($2.$parenttbl);" >> create_$1.sql
echo "\\copy $2.$tblname ($copylist) FROM $1.csv CSV HEADER;" >> create_$1.sql
```

This version is what I call create_child_from_csv and acts the same as the other but adds the fields to a child table. It uses the table to make a child of as the third parameter. To illustrate:

```console 
create_child_from_csv this my_project m_actor_usr
```

And the result is 

```console 
CREATE TABLE my_project.m_actor_usr_legacy (
    l_first_name TEXT
    ,l_middle_name TEXT
    ,l_family_name TEXT
) INHERITS (my_project.m_actor_usr);
\copy my_project.m_actor_usr_legacy (l_first name,l_middle name,l_family name) FROM this.csv CSV HEADER;
```

UPDATE!

I had cause to update the scripts a bit the other day so here they are:

* [Create Child Table From Header](create_child_table_from_header)
* [Create Parent Table From Header](create_standalone_table_from_header)

[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
