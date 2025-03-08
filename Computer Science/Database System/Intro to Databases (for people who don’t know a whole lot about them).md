#database

I was a CS major from an Ivy League university who’s now a software developer at [an awesome company](https://www.brandverity.com/) — and I don’t know much about databases.

I’m guessing I’m not the only one. It didn’t seem to be a beginner-friendly subject. The specific class wasn’t required. Any time storage systems were mentioned, they were on a higher level, very theoretical. There seems to be a common issue around CS graduates knowing very little about real-world software development (source control? deployment? huh?), and it’s up to us to figure most of this stuff out after the fact.

But I’m embracing my lack of knowledge on the subject, and blogging is a cool thing. Here’s hoping for some clarity of thought via writing.

## Storage systems aren’t a scary, magical black box.

Besides the fact that it wasn’t super accessible or approachable in college, it’s probably also taken me so long to do any self-learning here because it just seemed so intimidating and mysterious (but this might just be me). I switched from design to CS two years into college with no prior experience, and I was grasping at things that were readily available and that piqued my interest. For whatever reason, databases seemed like a thing that more experienced or “smarter” people were into. “Backend work” was something I veered away from, and the word “database” conjured images of highly technical, complicated systems with jargon I just wouldn’t understand. It was much easier to pretend it was ~magic~ and leave it for others to figure out (for now — it was always my intention to pick it all up eventually).

![](https://miro.medium.com/v2/resize:fit:998/1*66lYXVFX3rJxRu-8Ze3meg.gif)

Basically how I viewed data storage and retrieval.

## Let’s start at square zero: What is a database?

Google defines _database_ as “a structured set of data held in a computer, especially one that is accessible in various ways.” At its most basic, a database is just a way of storing and organizing information. Ideally it is organized in such a way that it can be ==easily accessed, managed, and updated.==

I like metaphors, so this simple definition of a database for me is like a toolbox. You’ve got lots of screws, nails, bits, a couple different hammers… A toolbox is a storage system that allows you to easily organize and access all of these things. Whenever you need a tool, you go to the toolbox. Maybe you have labels on the drawers — those will help you find, say, a cordless power drill. But now you need the right battery for the drill. You look in your “battery” drawer, but how do you find the one that fits this particular drill? You can run through all of your batteries using trial and error, but that seems inefficient. You think, ‘Maybe I should store my batteries with their respective drills, link them in some way.’ That might be a viable solution. But if you need all of your batteries (because you’re setting up a nice new charging station maybe?), will you have to access each of your drills to get them? Maybe one battery fits multiple drills? Also, toolboxes are great for storing disjointed tools and pieces, but you wouldn’t want to have to take your car apart and store every piece separately whenever you park it in the garage. In that case, you would want to store your car as a single entry in the database (*ahem* garage), and access its pieces through it.

![](https://miro.medium.com/v2/resize:fit:1180/1*Xfxl8HoQqqg_KtpEsEcskw.jpeg)

At least I can easily find the alternator this way, right?

This example is contrived, but reveals some issues you’ll have to consider when choosing a database or how to store your data within it.

# Let’s learn the lingo.

If you start directionlessly googling “databases” (like I did), you’ll soon realize there are several different types and lots of terminology surrounding them. So let’s try and clear up any potential language barriers.

While I’m sure someone has written books on each of these (some of which I should probably read), I’ll try to keep my definitions relatively simple. These were all terms that I came across while doing this research that I thought could use some quick explanation.

- **Query**  
    - A query is a single action taken on a database, a request presented in a predefined format. This is typically one of SELECT, INSERT, UPDATE, or DELETE.  
    - We also use ‘query’ to describe a request from a user for information from a database. “Hey toolbox, could you get me the names of all the tools in the ‘wrenches’ drawer?” might look something like _SELECT ToolName FROM Wrenches._
- **Transaction**  
    A transaction is a sequence of operations (queries) that make up a single unit of work performed against a database. For example, Rob paying George $20 is a transaction that consists of two UPDATE operations; reducing Rob’s balance by $20 and increasing George’s.
- **ACID: Atomicity, Consistency, Isolation, Durability  
    **In most popular databases, a transaction is only qualified as a transaction if it exhibits the four “ACID” properties:  
    **-** _Atomicity_: Each transaction is a unique, atomic unit of work. If one operation fails, data remains unchanged. It’s all or nothing. Rob will never lose $20 without George being paid.  
    - _Consistency_: All data written to the database is subject to any rules defined. When completed, a transaction must leave all data in a consistent state.  
    - _Isolation_: Changes made in a transaction are not visible to other transactions until they are complete.  
    - _Durability_: Changes completed by a transaction are stored and available in the database, even in the event of a system failure.
- **Schema  
    -** A database schema is the skeleton or structure of a database; a logical blueprint of how the database is constructed and how things relate to each other (with tables/relations, indices, etc).  
    - Some schemas are **static** (defined before a program is written), and some are **dynamic** (defined by the program or data itself).
- **DBMS: database management system  
    **[Wikipedia](https://en.wikipedia.org/wiki/Database) has a great summary: “A database management system is a software application that interacts with the user, other applications, and the database itself to capture and analyze data. A general-purpose DBMS is designed to allow the definition, creation, querying, update, and administration of databases.” MySQL, PostgreSQL, Oracle — these are database management systems.
- **Middleware**  
    Database-oriented middleware is “all the software that connects some application to some database.” Some definitions include the DBMS under this category. Middleware might also facilitate access to a DBMS via a web server for example, without having to worry about database-specific characteristics.
- **Distributed vs Centralized Databases**  
    - As their names imply, a **centralized database** has only one database file, kept at a single location on a given network; a **distributed database** is composed of multiple database files stored in multiple physical locations, all controlled by a central DBMS.  
    - Distributed databases are more complex, and require additional work to keep the data stored up-to-date and to avoid redundancy. However, they provide **parallelization** (which balances the load between several servers), preventing bottlenecking when a large number of requests come through.  
    - Centralized databases make **data integrity** easier to maintain; once data is stored, outdated or inaccurate data (**stale data**) is no longer available in other places. However, it may be more difficult to retrieve lost or overwritten data in a centralized database, since it lacks easily accessible copies by nature.
- **Scalability  
    **Scalability is the capability of a database to handle a growing amount of data. There are two types of scalability:  
    - **Vertical scalability** is simply adding more capacity to a single machine. Virtually every database is vertically scalable.  
    - **Horizontal scalability** refers to adding capacity by adding more machines. The DBMS needs to be able to partition, manage, and maintain data across all machines.

![](https://miro.medium.com/v2/resize:fit:1400/1*pWp5uSIjn0TgU9pnJe9MzA.png)

Vertical vs Horizontal Scaling. Little buckets don’t need as much brawn to carry, but they do require better coordination.

## Relational (SQL) Databases

- **“relational”**  
    - I highly recommend [this article](https://medium.com/@pocztarski/what-if-i-told-you-there-are-no-tables-in-relational-databases-13d31a2f9677#.gtwav0tad), which explains, “The word ‘relational’ in a ‘relational database’ has nothing to do with _relationships_. It’s about _relations_ from _relational algebra._”  
    - In a relational database, each relation is a set of tuples. Each tuple is a list of attributes, which represents a single item in the database. Each tuple (“row”) in a relation (“table”) shares the same attributes (“columns”). Each attribute has a well-defined data type (int, string, etc), defined ahead of time — schema in a relational database is static.  
    - Examples include: Oracle, MySQL, SQLite, PostgreSQL

![](https://miro.medium.com/v2/resize:fit:700/1*ko1siDrIKwAdrEA1P5o--g.png)

Thanks, [Wikipedia](https://en.wikipedia.org/wiki/Relational_database#Terminology).

- **SQL: Structured Query Language**  
    SQL is a programming language based on relational algebra used to manipulate and retrieve data in a relational database. _note_: In the bullet above, I’m intentionally separating the relational database terminology (relation, tuple, attribute) from the SQL terminology (table, row, column) in order to provide some clarity and accuracy. Again, see [this post](https://medium.com/@pocztarski/what-if-i-told-you-there-are-no-tables-in-relational-databases-13d31a2f9677#.gtwav0tad) for more details on this.
- **Illustrative Example:**  
    We could store all the data behind a blog in a relational database. One relation will represent our blog posts, each of which will have ‘post_title’ and ‘post_content’ attributes as well as a unique ‘post_id’ (a **primary key**). Another relation might store all of the comments on a blog. Each item here will also have attributes like ‘comment_author’, ‘comment_content’, and ‘comment_id’ (again, a **primary key**), as well as its own ‘post_id.’ This attribute is a **foreign key**, and tells us which blog post each comment “relates” to. When we want to open a webpage for post #2 for example, we might say to the database: “select everything from the ‘posts’ table where the ID of the post is 2,” and then “select everything from the comments table where the ‘post_id’ is 2.”
- **JOIN operations  
    **- A JOIN operation combines rows from multiple tables in one query. There are a few different types of joins and reasons for using them, but t[his page](http://www.sql-join.com/) provides good explanations and examples.  
    **-** These operations are typically only used with relational databases and so are mentioned often when characterizing “relational” functionality.
- **Normalization and Denormalization  
    - Normalization** is the process of organizing the relations and attributes of a relational database in a way that reduces redundancy and improves data integrity (accurate, consistent, up-to-date data). Data might be arranged based on dependencies between attributes, for example — we might prevent repeating information by using JOIN operations.  
    - **Denormalization** then, is the process of adding redundant data in order to speed up complex queries. We might include the data from one table in another to eliminate the second table and reduce the number of JOIN operations.
- **ORM: Object-Relational Mapping  
    **ORM is a technique for translating the logical representation of objects (as in object-oriented programming) into a more atomized form that is capable of being stored in a relational database (and back again when they are retrieved). I won’t go into more detail here, but it’s good to know it exists.

## Non-Relational (NoSQL) Databases

- **“non-relational”  
    **At it’s simplest, a non-relational database is one that doesn’t use the relational model; no relations (tables) with tuples (rows) and attributes (columns). This title covers a pretty wide range of models, typically grouped into four categories: key-value stores, graph stores, column stores, and document stores.

![](https://miro.medium.com/v2/resize:fit:1280/1*Pq8DSf6o1z2N-Qc30yAoPA.jpeg)

No tables.

- **Illustrative Example:**  
    - When we set up our blog posts and comments in a relational database, it worked in the same way as the drawers of our toolbox. But, much like our drill and battery example, does it make sense to always store our blog posts in one place, and comments in another? They’re clearly related, and it’s probably rare that we’d want to look at a post’s comments without also wanting the post itself. If we used a non-relational database, opening a webpage for post #2 might look something like this: “select post #2 and everything related to it.” In this case, that would mean a ‘title’ and ‘content’, as well as a list of comments. And since we’re no longer constrained by rows always sharing the same columns, we can associate any arbitrary data with any blog posts as well — maybe some have tags, others images, or as your blog grows, you’d like some of your new posts to link to live Twitter streams. With the non-relational model, we don’t need to know ahead of time that all of our blog posts have the same attributes, and as we add attributes to newer items, we are not required to also add that “column” to all previous items as well.  
    - This model also works well for the car example from earlier in this post. If you have three cars in your garage, it doesn’t make sense to store all of their tires together, seats together, radiators together… Instead, you store an entire car and everything related to it in its own “document.”  
    - However, there may be a downside to this. If you wanted to know how many seats (or comments, or batteries) you have total, you may have to go through every car and count each seat individually. There are ways around this of course, but it’s less trivial than just opening up the “seats” drawer and checking your total, especially on much larger scales.
- **NoSQL  
    **“NoSQL” originally referred to “non-SQL” or “non-relational” when describing a database. Sometimes “NoSQL” is also meant to mean “Not only SQL”, to emphasize that they don’t prohibit SQL or SQL-like query languages; they just avoid functionality like relation/table schemas and JOIN operations.
- **Key-Value Store**  
    - Key-value stores don’t use the pre-defined structure of relational databases, but instead treat all of their data as a single collection of items. For example, a screwdriver in our toolbox might have attributes like “drive_type”, “length”, and “size”, but a hammer may only have one attribute: “size”. Instead of storing (often empty) “drive_type” and “length” fields for every item in your toolbox, a “hammer_01” key will return only the information relevant to it.  
    - Success with this model lies in its simplicity. Like a map or a dictionary, each **key-value pair** defines a link between some unique “key” (like a name, ID, or URL) and its “value” (an image, a file, a string, int, list, etc). There are no fields, so the entire value must be updated if changes are made. Key-value stores are generally fast, scalable, and flexible.  
    - Examples include: Dynamo, MemcacheDB, Redis
- **Graph Store**  
    - Graph stores are a little more complicated.Using graph structures, this type of database is made for dealing with interconnected data — think social media connections, a family tree, or a food chain. Items in the database are represented by “nodes”, and “edges” directly represent the relationships between them. Both nodes and edges can store additional “properties”: id, name, type, etc.

![](https://miro.medium.com/v2/resize:fit:1232/1*pK3tfUnVRfdCeUtS76ixAQ.png)

Something [like this](https://en.wikipedia.org/wiki/Graph_database#Description).

- The strength of a graph database is in traversing the connections between items, but their scalability is limited.  
- Examples include: Allegro, OrientDB, Virtuoso

- **Column Store**  
    - Row-oriented databases describe single items as rows, and store all the data in a particular table’s rows together: ‘hammer_01’, ‘medium’, ‘blue’; ‘hammer_02’, ‘large’, ‘yellow’. A column store, on the other hand, generally stores all the values of a particular column together: ‘hammer_01’, ‘hammer_02’; ‘medium’, ‘large’; ‘blue’, ‘yellow’.  
    - This can definitely get confusing, but the two map data very differently. In a row-oriented system, the primary key is the row ID, mapped to its data. In the column-oriented system, the primary key is the data, mapping back to row IDs. This allows for some very quick aggregations like totals and averages.  
    - Examples include: Accumulo, Cassandra, HBase
- **Document Store**  
    - Document stores treat all information for a given item in the database as a single instance in the database (each of which can have its own structure and attributes, like other non-relational databases). These “documents” can generally be thought of as sets of key-value pairs: {ToolName: “hammer_01”, Size: “medium”, Color: “blue”}  
    - Documents can be independent units, which makes performance and horizontal scalability better, and unstructured data can be stored easily.  
    - Examples include: Apache CouchDB, MongoDB, Azure DocumentDB.
- **Object or Object-Oriented Database  
    **Not as common as other non-relational databases, an object or object-oriented database is ones in which data is represented in the form of “objects” (with attributes and methods) as used in object-oriented programming. This type might be used in place of a relational database and ORM, and may make sense when the data is complex or there are complex many-to-many relationships involved. Beware its language dependence and difficulty with ad-hoc queries though.

# So what does all of this look like in the real world?

Now that we know some stuff about databases, how can we apply that knowledge? How do you compare/test/benchmark different databases? What does it look like when they’re actually implemented, or when you have many working together?

All of this and more coming soon in blog post dos.