# "Out with the Old, In With the New" vs. Retract the Old, Add the New

For many of us, the idea of a "database" instantly brings to mind a data store that utilizes a relational model. We think of tables which have relationships to one another using primary keys, and data that can be queried using SQL. I believe though, that in order to fully understand our data, it's important to realize that there are different ways to store, query and relate to data. There was a world before the relational model, and based on some recent developments in the past decade or so, there is a future in it as well. This blog post should not be read as a slight to the relational database model, because for a lot of data that model really works. However, after my recent experience with learning and using Datomic on a fairly large client project, I've seen first hand that there are different ways to look at our data, and it's important to explore them. 

I'm going to begin with the assumption that many of us have a basic understanding of a relation database model. But to make sure that we're all on the same page, and have a common context, let's say that I have a database of 8th Light's contacts, where I can store first and last names, as well as email addresses. I may have a `contacts` table that looks like this:

![](/images/datomic-blog-table-1.png)

*I'd like to take this time to quickly write a disclaimer, that though I am a New York Giants fan, that this does not necessarily reflect the opinions of the rest of my colleagues at 8th Light. Okay, back to business...*

It's possible that some of our contacts have several email addresses that they'd like to share with us. So, let's create another table called `alternate_email_addresses` to store this information, if we have it. I'm then able to link the `contact` to their `alternate_email_address` by including the `contact_id` in this table:

![](/images/datomic-blog-table-2.png)

We can then query for Eli Manning's alternate email address(s) using the follow SQL statement: 
```psql
SELECT * 
FROM alternate_email_addresses
WHERE contact_id is 2;
```
And now, let's pretend, that Eli has decided to quit football, leave the New York Giants, and start a software apprenticeship at 8th Light. We will need to change his primary email address in our database to "eli@8thlight.com" from "eli@nygiants.com" since his New York Giants email address will presumably be deactivated:
```psql
UPDATE contacts 
SET primary_email_address='eli@8thlight.com' 
WHERE first_name='eli';"
```
And now our data looks like this:

![](/images/datomic-blog-table-3.png)

As you'll notice, after changing Eli's `primary_email_address` we no longer have record that his email address was ever "eli@nygiants.com". We could have moved it to the `alternate_email_addresses` table. However, this email address has been deactivated, so perhaps we would want to add a flag to that table to identify deactivated email addresses:
 
![](/images/datomic-blog-table-4.png)

You may notice, that this change would probably require some other data clean up: we would possibly need to indicate that all of the other records in `alternate_email_addresses` are still active. This could get tricky depending on how many `contacts` we have. And all of this effort is to save the *fact* that Eli used to have a different `email_address`. This information may never be needed again... but what if it is?

----
Let's take a little break from our New York Giants (as I'm sure many fans want to do after the past two weeks), to talk about what Datomic is, how it's architected, and how it aims to handle data differently.

##### Datomic has three main components:
- **Transactor:**
The Transactor's sole responsibilities are to write information to the data store, and to then alert it's Peers about the new data updates. It must ensure that the data remains consistent.

- **Data Store:**
Data storage is a unique component of a Datomic system, because it is external. Datomic, allows users to choose any data server to persist their data in. Datomic doesn't care whether it's a SQL system such as Postgres, or a NoSQL service such as Amazon DynamoDB. The rationale behind this, is that storing data is a problem that computing already has solved, and the Datomic team was eager to focus on other challenges like handling data reads and writes.

- **Peers:**
A Peer is any application that includes the Datomic Peer Library. As Rich Hickey mentions in his [Intro to Datomic video](https://youtu.be/RKcqYZZ9RDY) (which I highly recommend), Datomic "put[s] the brains back into the system, inside the application". What he means by "brains" includes the abilities to query our data within the app, a way to communicate with the other elements of the system (Transactor and Data Store), as well as local memory is used for caching. Since our application is able to query its local memory, this means that each application instance has the ability to interact with the dataset independently.

The application is not able to write to the database directly. Instead, it is able to send updates to the Transactor, which as we mentioned before, negotiates the transactions in our data storage. Since the Transactor then lets all of its associated Peer applications know when it has successfully written to the database, the Peers will be up-to-date with the latest data. Before we dig into Dataomic's data model, let's quickly go over a couple of architectural benefits that the three components above give us:
- Each Peer has its own brain, so we're able to safely distribute our application instances. Every Peer will be able to query its own data internally, and receive updates from the Transactor, so that it stays current. If one instance is performing a resource intensive query, it won't affect the rest of the instances.
- Datomic's query engine was designed to do its work locally for each Peer, and also has the capability to query other local sources. This means that we can use the same engine (and language) to query in-memory non-database collections, along with collections that are a combination of database and non-database sources. 
# [need to work on the bullet point below]
- Again, since we are interacting with our data via a local copy of the database, we are able to ensure that the data we're querying at any given moment is stable. When the query engine asks for the database at a given moment, it is give a "database value". This value is a snapshot of the data at a given point in time. It is also an immutable value, so we can safely query and manipulate this value without worrying about inadvertently affecting the validity of the data in our storage, or contaminating the data of our Peers.

----
##### Immutability
Speaking of immutability, I can't wait any longer to mention my *favorite*, though perhaps most mind-bending, part about Datomic: all data in Datomic is immutable. This means that each element of data can never be changed, and is never deleted! I know, it sounds crazy, but bear with me. First, before we get into specifics, let's point out that since we'll never be changing any piece of data, we can cache it... a lot. We can rely on that specific piece of data to always being the same, *** [fix/research]which is what allows our Peers to build up a local cache that can potentially need to communicate over the network less and less as time goes on. 

I understand that you're probably dying to see how this immutability is helpful, and how it can even work in a large scale production application, or really in any application, that ever needs to edit data. So, let's dig in! The main tenant of Datomic is that **it never forgets**. The database can be thought more of as a ledger of *facts*, rather than a series of boxes (or relational tables), where we organize data. Again in Rich Hickey's [Intro to Datomic video](https://youtu.be/RKcqYZZ9RDY), he likens Datomic to the method of record keeping that has been used for hundreds of years, before computers or databases: *facts* were written down on paper, and each new piece of information that needed to be recorded would be *added*. Existing data would never be overwritten.

In relational databases, we're often overwriting our existing records with new information. We can even see this in our example above, with Eli Manning's email address. In order to represent that Eli's `primary_email_address` is now "eli@8thlight.com", we had to lose the information that it once was "eli@nygiants.com" by overwriting it with the new data. This idea of "new information replaces old" originated when disk space was expensive, and computers didn't have any to spare. Data is saved to a specific place, and then recovered via a pointer. We remove old information, with the assumption (or the hope) that we'll never need it again. Datomic turns this thought process on its head, by allowing us to pose the question, *what if we do need old data again?*. 

Datomic's model is more dependent on the **time** at which information is written into the database, rather than the **place** that it is stored. Each transaction written by the Transactor is put in a *new* place in memory. And every element of data that's saved, is marked as being part of a specific transaction, and is given a timestamp (through the transaction). The implication of knowing which transaction added which piece of data, means that we are able to easily access all of our data `as-of` any point in time, or even within a window of time.

As we'll see soon, we don't need to worry about losing Eli's email address from his days in the NFL, since we can always query our database `as-of` before his 8th Light apprenticeship began.

# [do i put enough emphasis on the fact that we're looking at data as a snapshot in time?]
# [in a transaction, talk about being able to add or retract]

##### The Datom
It seems like it's about time to introduce Datomic's *one and only* data structure, the Datom. That's right, every piece of data in Datomic fits in the definition of a Datom. As per the official Datomic glossary, a Datom is 
>An atomic fact in a database, composed of entity/attribute/value/transaction/added.

For those of us whose brains are accustomed to thinking of data in rows and columns, we can correlate a Datom to a row, and a Datom's attribute to a column. Let's take look at how Eli Manning's information may be stored in a Datomic database comprised of several Datoms.

![](/images/datomic-blog-table-5.png)

We have one Datom for each attribute that makes up Eli's contact entity. This entity is itself is represented by a unique `entity-id`, 2, which is generated by the Transactor. Let's go back to our example of replacing Eli's New York Giants email address, with a brand new 8th Light one. It is possible, in Datomic, to define attributes in such a way that they can be associated with just one value, or with multiple values. We'll touch on this a little more, soon, but for now, let's take a look at the new *fact* that we would write in our database to represent replacing *both* of Eli's email addresses:
 # [insert table 7]
 
- We have the same entity number, because we're still working with Eli's contact entity.
- We include the attribute we're editing, `:contact/email-addresses`.
- The value of that attribute is what we're changing, so notice how we still include the "eli@themannings.com" address, but we change the other one. This allows us to say that these are both of Eli's current `email-addresses` as of transaction # 1001.
- Like we mentioned before, if we want to see what Eli's `email-addresses` were `as-of` transaction 1000, or 999, or 1, we could do that at *any time*.

As you'll notice in the column furthest to the right, we have our "added" component of the Datom which is `true` for all of these Datoms. We're also able to **retract** a fact if it is no longer true for an entity. To illustrate this, let's suppose that Eli decides to "go off the grid" entirely, and deactivates both of his current `emails-addresses`. To reflect this in our database, we would want to add a new Datom that says that these `email-addresses` are now false. 

![](/images/datomic-blog-table-8.png)

##### To Schema, or not to Schema
*is the schema part that interesting/important?

Datomic does in fact utilize a schema, though it differs from what we may think of a "tradition" relational schema. A Datomic schema describes specific characteristics of attributes, but doesn't necessarily need to limit those attributes to a specific type of entity - this can be done in the application code. When defining an attribute definition three pieces are required: the unique identifier (or name) of the attribute (:db/ident), the type (:db/type), and the cardinality (:db/cardinality). You can see that even Datomic's built-in attributes follow this structure: the identifier of each (:db/ident, :db/type, :db/cardinality) is defined under the :db namespace.

For 8th Light's contact database, we may create a schema defining three attributes: 

![](/images/datomic-blog-table-5.png)

Though it is not essential to include a namespace for each attribute, that matches up to their entity, it does help to avoid name collisions, and also helps to communicate the intent of each attribute. In this schema, we are expressing that we're setting our database up for a contact entity, with three attributes.

#### Benefits & Challenges
The benefits of implementing a looser structure in our data allows us to be adaptable to almost anything that our ever-changing requirements may throw at us. With Datomic we are not bound by a rigid structure of existing tables, relationships and keys. 

The beauty of the Datom being fairly generic, is that it's also ubiquitous. We can use it for any entity use case that comes our way. What if Eli's big brother, Peyton, also decides that he wants to join 8th Light, but instead of supplying his email addresses, he gives us his several phone numbers? We could easily accommodate this with our loosely-structured Datomic database. We would simply add a Peyton entity, and associate the attributes that are pertinent to him, without having to worry about the mismatch in data between the two `contact` entities (`email-addresses` vs. `phone-numbers`). We would also need to add a `:contact/phone-numbers` attribute to our schema, and we would be good to go!

![](/images/datomic-blog-table-9.png)

The possibilities of extensibility are endless! Going further (or maybe, a little bit too far), we could even start recording details about each of the Super Bowl Championships that every new 8th Light apprentice has won. Again, we wouldn't have to worry about the fact that *most* incoming 8th Lights do not, in fact, have a Super Bowl ring. Yet we could easily still add this attribute to both of the Manning entities.

Perhaps this is a lesson that we can bring into our development of systems that use a relational model. Why should we shy away from adding more data that is useful to our application, just because our current structure of tables isn't set up for it? After exploring Datomic's ability to allow us to be flexible, adaptable and agile, I am encouraged to continually seek out more creative ways to represent data.

It is worth mentioning, that Datomic still seems like a relatively new technology, and this brings about its own set of challenges. It is still evolving and as such, there aren't always a plethora of resources online. When trying to solve a particular problem, it's usually difficult to find more than a few Stack Overflow entries, blog posts and tutorials to assist, besides Datomic's own documentation. However, in lots of Google searching, I have notice that Rich Hickey himself seems to be very active in answer questions on both Stack Overflow, and even in a Google Group dedicated to Datomic, which is pretty cool, and encouraging.

Sometimes it *can* feel like you're the first one to ever try to query data in a particular way - which can be daunting. But, it is also really exciting. We, as early users of a new technology, have the opportunity to shape the way it's used, and possiblye where it goes in the future. Which is pretty cool. But honestly, the opportunity to think about data in a new, and different way is even more exciting to me. Understand data in a linear/time-sensitive way, instead of relationally is eye-opening. It's helped me to understand my data more thoroughly, as well as realize more creative ways that data can be linked and understood. We're not always restricted by the database structure that we currently have - and this allows us to add new relationships between our data.