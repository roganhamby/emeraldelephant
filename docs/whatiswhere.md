[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)

### <a name="whatiswhere"></a> What Is Where in the Database?

It has been a while since I have blogged mostly due to life being very busy. However, I am shaking off the stupor a bit because the other day someone asked “how do I know where to find the various stuff in the Evergren database?” This is something I meant to write up a long time ago and kept putting off but it’s a beautiful Saturday morning to sit outside and write so let us do so.

Jumping in …. Content in Evergree is oranized into schemas, which you can think of folders to hold tables, views, functions, indexes and whatever else should be grouped together.  We will run down these alphabetically and I will generalize the intent of the schema as well as pointing out critical tables and noteworthy exceptions. I will save a discussion of the most useful views for another time.

**acq**

The acquisitions schema. If you don’t use Evergreen acquisitions you can ignore learning this but if you do no one table stands out as more important, you will want to become familiar with the schema as a whole.

**action**

As might be implied the action schema is things that are done for actors or by actors, primarily holds and circulations though curbside pickups and transits live here too. The two most important tables in this schema are action.circulation and action.hold_request. 


**action_trigger**

Primarily known to many as notices, action triggers actually do a lot more. Many tables in Evergreen are heavily interpreted by the UI but the action triggers and events as you see them in the staff client are pretty much a 1 for 1 representation of action_trigger.event_defintion and action_trigger.environment. When events actually happen that is tracked from action_trigger.event and the output, usually template or error, goes to action_trigger.event_output. Also note the process listed in action_trigger.event - that value is very useful for searching opensrf error logs for debugging an action trigger. 


**actor**

These are the entities of the database and three tables make their home here you will use over and over again - actor.usr (the staff and patrons), actor.card (their cards which can be many cards to one user), and actor.usr_address which you can probably guess what it stores. Oddly, organizational units are stored here too and you will routinely need to query actor.org_unit as it s linked more than probably any other table to other tables. The actor schema has one oddity in the existence of actor.ou_type - a sort of table that defines the kinds of org units (say you want to make sub libraries) and is usually in the config schema.

**asset**

The stuff.  Assets are copies, items, whatever you want to call them and they live in asset.copy.  

**auditor**

By default this stores copies of some tables every time a change is made via a trigger on the table. More can be added via a function but they can bloat the database quickly and the existing ones already take up a lot of space. Another of my posts talks about using the auditor in more detail. If you do use this a lot I recommend doing the queries on a read only database rather than a production read/write one.

**authority**

If you know authorities this schema will make sense. If you don’t, it won’t make things any clearer. The management of authorities as a resource is mostly handed via authority.record_entry.

**biblio**

Libraries need to define the materials they have and when talking about the attributes of an item versus a specific instance that is done in bibliographic records and those are stored in biblio.record_entry. If you do frequent updates to the marc, which stores the data in MARC-XML, then put pauses between or you could over whelm replication as the system does a lot of processing to bibs when changes are made. 

**booking**

Booking is used for rooms, computers or anything else you may want to reserve. If you have a large computer lab a specialized solution will work better but many libraries with just a few computers save money by using booking. It has also become much better in the last few versions thanks to contributions from academic libraries. 

**config** 

This is exactly what it sounds like.  You see an arbitrary value somewhere like circ_mod is ‘BOOK’ and you want to know what attributes that gives it?  Or a standing penalty is 24 what the heck does that mean? Those are ids of rows in tables in this schema. 

**container**

Buckets. Item buckets. Bib buckets. If its a bucket of some kind, even one that holds carousel entries, it is here.

**evergreen**

The evergreen schema holds evergreen specific things. Make sure it is in your path or every now and then you’ll run into something that doesn’t work. Sometimes people are tempted to place resources in this schema and I would just make a project specific schema if you need to instead. 

**extend_reporter**

In my mind I think of this as a place to drop data sources to extend the reporter. For example, I’ve put tables here with census data and build custom IDL entries so it could be accessed from the reporter. However, there is a stock place here to put historical circulation information for items, especially useful in migrations though what it will store is very limited.

**information_schema** 

This is a postgres schema not Evergreen’s but I am going to mention it because you can go a very long time without using it and then when you do it’s sooo useful. Want to write a script to know everywhere actor.usr’s id column is referenced? You can use this. Want to be able to get a list of columns on a table that are text so that they can be properly quoted and escaped in a CSV export function? You can do that with this.

**metabib**

Someone else probably has a more elegant explanation of this table but I think of it as where “when Evergreen tears apart and calculates stuff from a marc record it ends up here in some way.” metabib.real_full_rec stores the actual normalized tags from the marc record while other tables store things like their calculated search formats based off their MARC values. 

**migration_tools**

Thsis another not stock Evergreen schema but comes from the Equinox Open Library Iniatitve’s migration_tools repo on Github. I may be biased as a contributor to said repo but I find it has a lot of tools that are useful outside migrations.

**money**

Money stores information about billing. I’ve talked about this a fair bit on posts talking bout paying off bills.  

**oai**

Most users can safely ignore this schema. It provides useful views for an external service.

**offline**

The core of this is in the name - this tracks information related to offline transactions as they are uploaded and processed. If you need to use this table something has gone wrong somewhere.

**permission**

These tables map the permission groups, or profiles, of users as well as extended per person mapping and the org units they can use permissions at. The most important table here is permission.grp_tree though if you give a lot of individual permissions or use secondary profiles the other tables here will be close friends too.

**public**

Again, a postgres not Evergreen schema. Generally you should avoid this but if you only have read access to a system and you need somewhere to throw a temp table or define a function you can. If possible you are better off asking for access to write to a specific schema though it could be one made just for your stuff and doesn’t need to be an Evergreen table.

**query**

If you’re still learning Evergreen’s layout you can safely ignore this schema and come back to it when you have specific cause to.

**rating**

If you use popularity badges this schema stores that information but other than looking up some specifics to tweak badges (and that is an unlikely event) you’re not going to have need to come here. 

**reporter**

If something goes wrong with reports running you may need to fiddle with reporter.schedule or look at visibilties in the folders if something isn’t findable in the staff client. But the most useful feature day to day is reporter.super_simple_record which saves you the leg work of say coalescing the 100, 110 and 111 when doing your own reports.

**search**

Look up at what I said about the query schema. Same here.

**serial** 

Serials are increasingly less common in public libraries but still a mainstay of academics and their data that is unique to them is stored here.  There is a serial.record_entry but they shouldn’t be confused with the MARC record which is stored in biblio.record_entry which can be linked to these.  It should also be noted that serial.unit is a child table of asset.copy. 

**staging**

Used to store potential patron information before it does into the actual patron data by some secondary processes. It isn’t used a lot.

**unapi**

Used mostly to store functions used by the unapi services for external resources to get information from Evergreen.  It is unlikely you will ever have to do anything in this schema.

**url_verify**

This stores the stuff needed for Evergreen to use it’s URL verification ability to tell when records have bad links out, especially important for electronic resource records.  It is very possible you will never have to deal with this schema.

**vandelay**

One of the more cryptically named database elements it is the importer / exporter tables of Evergreen and again a schema you probably won’t deal with often.

And that’s it folks. 


[![Return to Index](https://raw.githubusercontent.com/roganhamby/emeraldelephant/master/Azzuri_tiny.png)](index.html)
