[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="sowhat"></a> So What

I was working with someone recently and was reminded that sometimes the biggest time savers are the simplest things. What follows is a simple one line bash script. It isn't fancy. No one who has done much in bash is going to be impressed. But if you spend a lot of time on the command line with psql this sort of wrapping of things is very useful and if you are new to it this sort of thing is a great to have in your toolbox. 

I call the script 'so' though you could name it something more meaningful. Why do I call it 'so'? Basically because each time I run it I'm thinking "so, let us see what happens."

```
#!/bin/bash
time psql 'host=dbhost user=dbuser options=--search_path=my_project,evergreen,public' -f $1 2>&1 | tee $1.out
```

The script does the following things: 
1. it uses the 'time' command so that in the output we know when the command was run 
2. it invokes psql with a stored connection string 
3. it passes to psql a search path so that you can specify the schemas you will use a lot in your project
4. it split the output so that you can both see the results on the screen and store it to a file 

Sytax would be something like this: 

```
./so myfile.sql
```

This would put the output of the script on the screen and store output to myfile.sql.out which can be very h andy for review.


[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
