---
title: "To Stored Procedure or Not"
date: 2021-12-07T20:32:49+01:00
draft: false
---
### Or: My personal Object-Relational Mapping [Vietnam war story](https://blogs.tedneward.com/post/the-vietnam-of-computer-science/)

![tools](/img/rene-deanda-fa1U7S6glrc-unsplash-1.jpg)

__tl;dr__
Choose the right tool for the job. What that is, well, it depends.

The standard reaction of a lot of java developers (add enterprisey language X) to the question: 'Stored procedures?' is often a plain 'No'.
The reasoning goes somewhat like this:
_'A database is a dumb datastore that might be swapped out, so we don't want to add intelligence there.'_
At least that is what I heard multiple times in the past, and as you can guess by now: the database is never, ever swapped out. God I wished I was Larry Ellison!

A more valid argument goes like this:
_'We need to add business logic and we want to centralize that in the application'._

And indeed you would not want to put logic in the database as well as the application. The debugging burden doubles and weird interaction effects may follow.

Often people make a distinction between high-level and low-level logic. It is a good thing not to put address retrieval (low level) and 'validate address for customer message X' (high level) in the same component. Downside is that there are always gray areas and looking at it from the outside, you may not be sure to take the right direction when fixing some bug. _Do you take the database or the application?_

Service layers are a typical place to suffer from this sometimes blurry distinction. Either logic from above (the REST controller) or from below (the datastore, messaging) leaks in. Service layers ideally are abstracted away from technical implementation details. But while this is all nice and good, it is often difficult to uphold. Service layers get messy quickly for other reasons too. A service needs to call another service, calling another... Microservices mitigate this, but I am digressing. In the end a lot of responsibility lays in the hand of the senior developer, given unittests and enough time for refactoring.

Let me introduce a situation where the service layer is actually a no-op, only passing on data from and to the restcontroller and the database. The data objects themselves are also mere structs, devoid of logic. This ['anemic'](https://martinfowler.com/bliki/AnemicDomainModel.html) service layer , whether materialized in actual code that does nothing, or just absent, should indicate that there is, indeed no business logic. A mere CRUD type application, or even just reading database values for displaying them in some user interface. This type of application is quite common. It can be a part of a larger system, where it fulfills a specific need: show some data. Could be ticket retrieval in a messaging system, or a blog application, a CMS etcetera. It could also be a part of an eventsourcing architecture in which domain events are aggregated in a read-only datastore that is optimized for a small number of queries.

Working in enterprisey governmental organization X, we built such a system with 'boring' tech: a java enterprise (JEE) application, running in IBM Webpshere, on linux. This tech stack is fortunately getting less and less common. Now that IBM has acquired Redhat, big bureaucratic entities have an shiny new yet still amazingly enterprisey alternative: openshift and openliberty, the latter being a 'stripped down' websphere, somewhat more suited to the past decade (ie. containerization). This way JEE can still pay for mortgage.

The application connected to SQLServer running on Windows. While initially things were all well, at one point the development team ran into a problem. The SQLServer guys demanded what they consider normal: login through the windows active directory couplings that windows provides and other OS's lack obviously. So, good old username and password were not considered safe enough. 'Only when we disable 'mixed-mode' (as it's called), and use Active Directory for database user management, we can make sure all database activity is audited and that is what we want', is what they said. 
Websphere on top of linux, on the other hand, in use in basically the rest of the organization, has no support for this, nor for Kerberos authentication in datasources. Kerberos should be a common standard for this type of authentication challenges, and is well supported both on linux as in standard java. Webpshere too, supports it, just not for datasources in our case (only when you need user credential delegation, and still hardly documented, if at all).

So what do we do? 

The project goal is to build a customer messaging system that is structured like a typical workflow application, enriching, routing and transforming messages from mere data to a pdf and other formats. Whenever some step in the process goes wrong, the message gets stuck in an 'error' state, that requires manual intervention. This is all implemented in an off the shelf application, let's call it Cardhouse (not the real name), with bespoke (javascript) components for specfic steps in the process. 
So now there is a need for a user interface for analyzing messages with errors and it was decided that a web application and corresponding REST backend be built, serving the database records in JSON format. Originally the architects had envisioned this backend as part of the Cardhouse solution itself, but the development team had advised against it ('not possible/feasible'), and so java, being the standard for custom applications, was decided upon.

Let's take a look at how this java application is structured and what kind of code it contains:

First: a fairly standard and decently implemented layered architecture:
* MessageDao (Data access object), uses JPA @Entity's like Message, MessageErrorRecord, MessageDetails
* MessageService, uses the JPA entities, simply passing them on to:
* MessageRestController @Stateless, uses MessageDTO and MessageDetailDTO, these objects can be automatically converted to JSON.

But there is more:
The Controller's input argument is a bean with @BeanParams that has @QueryParams, all optional url query parameters, to filter certain message properties like date, status, and type. And because we don't want to generate the database query (with all the optional select columns) ourselves, we use JPA's Criteria API. 
The following code has been anonymized to protect the innocent:
```
...
final Join<Message, MessageErrorRecord> messageErrorRecordListJoin = from.join(Message_.messageErrorRecordList, JoinType.LEFT); //*
final Subquery<Timestamp> subQuery = query.subquery(Timestamp.class);
final Root<MessageErrorErrord> subRoot = subQuery.from(MessageErrorRecord.class);
final Path<Timestamp> errorEventDateTime = subRoot.get(MessageErrorRecord_.errorEventDateTime);
subQuery.select(builder.greatest(errorEventDateTime));
subQuery.where(builder.equal(from, subRoot.get(MessageErrorRecord_.message)));
final List<Predicate> predicates = new ArrayList<>();
predicates.add(builder.or(builder.equal(subQuery, messageErrorRecordListJoin.get(MessageErrorRecord_.errorEventDateTime)),
    builder.isNull(messageErrorRecordListJoin.get(MessageErrorRecord_.errorEventDateTime))));
...
errorCode.ifPresent(ec -> predicates.add(builder.like(messageErrorRecordListJoin.get(MessageErrorRecord_.errorCode), like(ec))));
messageType.ifPresent(mt -> predicates.add(builder.equal(from.get(Message_.status), mt)));
...
```
* class names ending with underscore (Message_) are a result of JPA @StaticMetaModel, in case you're wondering. They are generated by a Maven plugin.

So what's going on??

First: here is the (simplified) datamodel
```
message -< (1..*) message_error
```
Meaning a message can have zero or more message_error records. There are more tables, having the same 1-to-* cardinality, that all contain extra information about the message while undergoing processing. It's worth mentioning that this model was devised for the Cardhouse system itself, not the java application or its function. A large part of it though is storing data for tracking and auditing purposes. There is a genuine  need for accountability, because the messages sent can indeed ruin lives!

The user (an administrator) wants to see messages with at least one message_error record, and when there are more (when processing has failed multiple times), just the latest (for an overview page, he or she then clicks for details). 

In pure SQL (simplified for readability): 
```
SELECT * FROM message m
JOIN message_error me on me.message_id = m.id
WHERE status = 'error'
AND [OPTIONAL SELECT's]
AND me.datetime = (SELECT MAX(datetime) FROM message_error WHERE message_id = m.id)
...
```

So the query is a little bit complex, as there is a join and a subselect. There are other ways to do this, bit it works fine, performance-wise as well, but believe me, it hasn't been properly performance tested yet and isn't live at the time of this writing. It's about nine months old now.

_ insert meme here: It's like that, and it's the way it is _

The JPA code above generates more or less the same query in the end. It adds even more unnecessary complexity by the way, but the bigger point I am trying to make is: this JPA code, while using all the industry standards, is absolutely horrendous!

The astute reader (good job!), might notice that the java code is built on the assumption that there might be records in the message table that have status 'error' but do not have an entry in message_error. This is not a valid situation and I left it out of the SQL query. This is actually a fascinating side story, because the java team and the Cardhouse team hardly spoke for a year, because the Cardhouse team, the producer of messages and message_errors had one priority: go live with their system. This wrong assumption is minor really, compared to others...

Anyway

I started this story describing the impediment with the websphere datasource. There was a regular datasource in DEV, because that sqlserver instance still had so called mixed-mode, enabling good old username & password. So no one noticed until the dev team asked for a connection string to the TST database.
And that's were things went sour pretty quickly and the term 'total rewrite' was heard aloud in meeting #1. 'There must be other options!'

Here starts just another aspect of this story.

The initial java code was written by a medior level developer. His style of writing code is basically: copy stuff from the previous project and adjust until it works. And it worked, after a couple of weeks. It did not yet contain all the fancy predicate code, just plain SQL within JPA and those fine nullchecks:
```
String query = ...
if (timestamp_from !=null){
    query += " AND me.timestamp > ?";
}
```
you get the gist...mind the spaces.

Then, another medior level developer, a younger more ambitious guy, yearning to build beautiful things, took over. Should this have happened, scrum-wise? I wonder, but I wasn't really involved at the time, so I don't know. I do recall that there wasn't really any pressure coming from the backlog. We had to beg for work at the time, so is that a blessing or a foe? This younger developer introduced the Criteria API and some other modern inventions and created a rather intricate web of code. I entered the picture when a review was needed. I spent a lot of time getting my head around it, partly because I did't know JPA that well, but also because of unclear naming. So I commented on the code, got into an argument, but in the end prevailed, and convinced the guy that my simpler code was better. 

This was the code I actually called horrendous...

And here this part of the story ends. We built the UI as well using a very well known enterprisey web framework. All was fine and good until we wanted to test the lot in the TST environment. How long did it take to write the jave code? It's hard to reconstruct, so let's give some statistics:

number of classes: 32
lines of code: 1516
explanation: just a quick manual count, including whitespace and comments

I included all javax.validation code, to make sure a date entered in the UI is actually a date. All jackson customizations, for correctly serializing dates (them again...), Boilerplate like the class that contains @ApplicationPath etcetera. But I did not include all the xml that JEE still requires. Not the huge openliberty server config xml ... etcetera.

Still, 1516 lines of code to inspect 5 tables (3 more that I didn't mention for brevity)

- that - is - a - lot - insert fitting meme here -

Ok, JEE is bad! But trust me, it wouldn't have been (much) better in springboot, micronaut, quarkus or any other java framework du jour. Sure springboot can autogenerate queries, but not this one...

Did I mention we use Lombok? There are not getters or setters in the count.. - insert smile emoji - We all love Lombok, don't we? 

So what's the real problem? 

I think the real elephant in the room, the thing that is omnipresent, but overlooked all the time is -context-.
Let me explain:
This is a simple application, with simple requirements, the most important being: it's read only.
This rules out a swath of stuff. There is no state, no user session, no business logic, some input validation. No concurrency. No huge performance requirements (total number of users: 5, a wild guess). 
And JEE (or J-$X), this one-solution-fits-all kind of a beast that it still is (despite micro-profile and all) is just overengineering from the beginning. - In - this - Context - (add emphasis)

[note to self: add the word 'bloated' somewhere]

But what about language X, what about PHP, ruby, python/django, .Net etc. ??
Ok, There are frameworks in other languages that are probably better at some (a lot of) things than java. On the other hand: being an IT manager (not me) it seems java is a good solution in this case because:
-java is battletested
-java has been used before in projects like this
-we only want to hire java developers
-there are zillions of java developers in this country (The Netherlands) and in places like Rumania and India..
-they all sort of get the job done, which is good!

And while some languages like python do shine in one direction, they tend to have disadvantages as well. It's a tradeoff, like everything else.

So maybe the language you chose, or was non-negotiable, is not the issue. What is?

So let's go back to the original problem:

'Select and show some data in sqlserver in a web user interface'

This is what I ended up with:
```
CREATE OR ALTER PROCEDURE MessageOverview
  @errorcode VARCHAR(4) = NULL,
  @date_from DATETIME2(3) = NULL,
  @date_to DATETIME2(3) = NULL,
  ... other filter criteria,
  @offset INT = 0,
  @fetch_size INT = 25,
  @sort VARCHAR(25) = 'id.ASC'
AS
IF @fetch_size >200
BEGIN
   SET @fetch_size = 200
END
SELECT(
    SELECT m.Id ...,
        me.error_code, ...
    FROM message m 
    JOIN message_error me on me.message_id = m.id
    WHERE m.status = 'error'
    AND (@errorcode IS NULL OR (m.error_code = @errorcode))
    AND me.datetime = (SELECT MAX(datetime) FROM message_error WHERE message_id = m.id)
    ORDER BY
        CASE WHEN @sort = 'id.ASC' THEN m.Id END ASC,
        CASE WHEN @sort = 'id.DESC' THEN m.Id END DESC,
        ...
    OFFSET @offset ROWS
    FETCH NEXT @fetch_size ROWS ONLY
    FOR JSON PATH) AS results
,
...
FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
GO;
```

This is some remarkably effective code. It recreates the exact same JSON structure as before, which is an object containing a page of X results and a count for the total number of records in the database for the given filter criteria. I had not written a stored procedure in my life, and it cost me like half a day. This is not about me bragging. It is about the documentation readily available online, from microsoft and stackoverflow. And the clarity and strength in SQL combined with the recent (2016) JSON capabilities in SQLServer.

In a total of 71 lines it does what cost 1516 lines of java code did. Well, not really. There's hardly any input validation here and it isn't a http server. But we didn't count the lines in openliberty now, did we?

And don't get me wrong. This seems to me a good fit given the _Context_ I described earlier.
Whenever you need to make decision in code, based on the data ('business logic'), you probably want a single place as mentioned. So would you want that in a stored procedure? I guess not. One option is to have the procedure as a lowlevel component, providding 'objects' to the domain layer where the 'real' decisions are made.
This all depends and there is no silver bullet, only tradeoffs, many of which not directly technical.

We could do a couple of things now. Replacing the java backend with a small java service that calls this procedure is not one of them because we haven't fixed the database login yet.
If only we could run on windows... then logging in would be easy, right. .Net? Yeah, but there's no .Net yet in this project. That would mean more new infrastructure, and we can't completely remove java yet. Nah, the project owner/manager doesn't like that. What about some custom javascript code in 'Cardhouse' that calls the database and returns the JSON to the client? This product does run on windows here and importantly, already talks to this database, so why not add a service? As it turns out, we can actually do this. In fact, it's what the architect had originally wanted, remember? So everyone is happy.


PS. now, don't laugh when I mention this: Cardhouse, what is that written in? Not javascript, no it's: java... Yes you heard right, plain old f##ing java and the Rhino javascript engine. Irony, doubled.