Disclaimer
==========

This code may change in the future. I intend to publicly store data in
this repo, but the file names and content of the data is subject to
periodic change. Open a bug if you would like to have access to a stable
or private data-set. We have been using out current setup since June 21,
2015, and we are producing data at a prodigious rate. Data collected
prior to June 21 is more variable and inconsistent due to the
limitations with the preceding rig and our problems figuring out how to
use it. I do not recommend using that data for analytical purposes.

Project Code
============

The code discussed here is available on GitHub here:
[<https://github.com/Choens/blood-sugars/blob/master/get-cgm-data.Rmd>](https://github.com/Choens/blood-sugars/blob/master/get-cgm-data.Rmd)

Goals
=====

[Nighscout](https://nightscout.github.io/) stores all of Karen's CGM
data on a hosted [Mongodb](https://www.mongodb.org) server, controlled
by our family. To use Karen's Nightscout data to better understand her
diabetes, I have to learn how to use Mongo. Mongodb is the formal name
of the product, but referring to it simply as Mongo appears to be an
acceptable shorthand and is used here.

I have been using the [Robomongo](http://robomongo.org/) application to
view Mongo JSON data directly in the db server. Like R, it is FOSS and
is available for Linux, Mac and Windows.

*GOALS:*

1.  *DONE:* Query Karen's Nightscout database.
2.  *DONE:* Import Nightscout data (JSON) and turn it into a R data
    frame.
3.  *DONE:* QA this data frame (number of entries, data completeness,
    etc.).
4.  *DONE:* Export the data frame as a CSV for further analysis.
5.  *IN PROGRESS:* Develop a function to simplify the query process
    against Nightscout databases.

Goals \#1 and \#2 were hard because I am not an experienced NOSQL /
Mongo programmer. Goals \#3 and \#4 were easy, these "goals" are basic R
programming. Goal \#5, a function to simplify querying the Nightscout
data, will be completed at a later date. I will use what I learned here,
to inform that process. I'm not ready to publicly discuss my ideas for
that goal yet. My next post will use the data we collect today to create
some simple analytic views of the CGM data.

There are two R packages able to query Mongo,
[RMongo](http://cran.r-project.org/web/packages/RMongo/index.html) and
[rmongodb](http://cran.r-project.org/web/packages/rmongodb/index.html).
After skimming the documentation to both packages, I first tried to use
RMongo. The documentation made it look easier to use. Unfortunately, I
was never able to connect to our Mongodb instance running on
[mongolab](https://mongolab.com/) using RMongo. RMongo lacks the ability
to connect to Mongo using user-names, passwords or custom ports. The
syntax is more R-centric and it is probably great if you want to connect
to a local Mongo server, but it is not useful for connecting to a remote
server[1]. Fortunately, I had better luck with rmongodb, even though
using it appears to be more complicated. Unlike RMongo's syntax, which
appears to hide Mongo's inner working from the user, rmongodb requires
the user to use a form of programming more commonly seen in web
development than analytic code. It feels unnecessarily complicated but
it does work (and well). Much of the complexity I don't like is because
of structural differences in JSON objects and data structures used in R.

Goals \#1 & \#2: Import & Convert To Data Frame
===============================================

Nearly all of my database experience is with Relational Database
Management Systems (RDBMS) such as Postgres or SQL Server. To succeed
with rmongodb, I had to adopt a different work flow than I am used to.
It isn't really harder, but it quite different compared to what I would
do to query data via RODBC or other RDBMS-oriented package.

Init
----

I always start with a chunk called init, to define variables, load
packages, etc. This script has two dependencies, rmongodb and dplyr.

For obvious security reasons, passwords.R is not part of the public
repo. Because of this decision, you cannot run this code. I prefer to
post public code that can be run by anyone, but that simply isn't
possible in this case. I may be willing to share the password with
co-workers, but I do have to restrict direct access to the server. The
data queried from the server today is available in the data/ directory
of this repository. Analytically oriented posts will use this data,
rather than data obtained directly from the server, to ensure the
reproducibility of the analysis.

    ## passwords.R -----------------------------------------------------------------
    ## Defines the variables I don't want to post to GitHub. (Sorry)
    ## ns is short for Nightscout.
    ## ns_host = URL for the host server (xyz.mongolab.com:123456)
    ## ns_db = Name of the Nightscout database (lade_k_nightscout)
    ## ns_user = Admin User Name (Not admin)
    ## ns_pw = Admin Password (Not Admin)
    source("passwords.R")

    ## Required Packages -----------------------------------------------------------
    library(rmongodb)  ## For importing the data.
    library(dplyr)     ## For QA / data manipulation.
    library(pander)    ## For nice looking tables, etc.

As of July, 2015, rmongodb returns a dramatic warning when running
`library(rmongodb)`:

> WARNING! There are some quite big changes in this version of rmongodb.
> mongo.bson.to.list, mongo.bson.from.list (which are workhorses of many
> other rmongofb high-level functions) are rewritten. Please, TEST IT
> BEFORE PRODUCTION USAGE. Also there are some other important changes,
> please see NEWS file and release notes at
> <https://github.com/mongosoup/rmongodb/releases/>

I did not experienced any problems using rmongodb, other than a few I
think I created for myself. Opening a connection to Mongo is similar to
opening a connection to a RDBMS[2]. Mongo stores data in an object
called a collection, which is sorta-kinda like a table in a traditional
RDBMS. However, unlike a table, the structure of a collection is not
defined prior to use. There are several other important differences,
which you can learn about by reading the [introduction
tutorial](https://www.mongodb.org/about/introduction/) written by the
developers.

The following code chunk returns a list of all the collections which
exist in our Nighscout database. The "entries" collection is the only
collection we are interested in today. The database name,
lade\_k\_nightscout is prepended to each collection name.

    ## Open a connection to mongo --------------------------------------------------
    con <- mongo.create(host = ns_host,
                        username = ns_user,
                        password = ns_pw,
                        db = ns_db
                       )

    ## Make sure we have a connection ----------------------------------------------
    if(mongo.is.connected(con) == FALSE) stop("Mongo connection has been terminated.")


    ns_collections <- mongo.get.database.collections(con, ns_db)
    pandoc.list(ns_collections)

-   lade\_k\_nightscout.entries
-   lade\_k\_nightscout.devicestatus
-   lade\_k\_nightscout.treatments

<!-- end of list -->
The next code chunk will produce some variables needed to temporarily
hold the Nightscout data. When importing data from a RDBMS, it is normal
practice to place the imported data into a R data frame. When importing
data from a non-relational database such as Mongo, we need to import the
data as a series of vectors. This is further complicated by the fact
that records have a different number of fields and we have to handle the
NULL values in R, rather than via the database.

The query imports thousands of records. Rather than build each vector
incrementally, it is faster and more memory efficient to create vectors
large enough to hold all of the data present in the server. The
variable, ns\_count, is used to hold the number of records in the
"entries" collection.

    ## Make sure we still have a connection ----------------------------------------
    if(mongo.is.connected(con) == FALSE) stop("Mongo connection has been terminated.")

    ## Collections Variables -------------------------------------------------------
    ## Yeah, I just hard-coded these. Sue me.
    ns_entries <- "lade_k_nightscout.entries"

    ## Mongo Variables -------------------------------------------------------------
    ## ns_count: Total number of records in entries.
    ## ns_cursor: A cursor variable capable of returning the valie of all fields in
    ##            a single row of the entries collection.
    ##
    ns_count   <- mongo.count(con, ns_entries)
    ns_cursor <- mongo.find(con, ns_entries)

    ## R Vectors to hold Nightscout  data ------------------------------------------
    ## If you don't define the variable type, you tend to get characters.
    device     <- vector("character",ns_count)
    date       <- vector("numeric",ns_count)
    dateString <- vector("character",ns_count)
    sgv        <- vector("integer",ns_count)
    direction  <- vector("character",ns_count)
    type       <- vector("character",ns_count)
    filtered   <- vector("integer",ns_count)
    unfiltered <- vector("integer",ns_count)
    rssi       <- vector("integer",ns_count)
    noise      <- vector("integer",ns_count)
    mbg        <- vector("numeric",ns_count)
    slope      <- vector("numeric",ns_count)
    intercept  <- vector("numeric",ns_count)
    scale      <- vector("numeric",ns_count)

As of 2015-07-17 the "entries" collection contains 11,960 records. That
is a lot of data, about a single person. The next code chunk imports all
of the records in the collection and places the data into the vectors
produced above.

The use of the "ns\_cursor" variable should be familiar to anyone with
web-development experience. A mongo cursor works in much the same way a
RDBMS cursor works. It returns a single record at a time. This appears
to be the preferred way of getting results from Mongo. This is very
different than importing data from a RDBMS, which would normally be
imported directly as a data frame.

    ## Get the CGM Data, with a LOOP -----------------------------------------------
    ## The examples I found on the Internet always show this loop as a while loop.
    ## Depending on my future import needs, I may need to change this to a for loop.
    ## A new record is added every five minutes, which means it is not impossible for
    ## a new data entry to be produced while this script is running, even though it
    ## takes less that a couple of seconds to run.

    i = 1

    while(mongo.cursor.next(ns_cursor)) {
        
        # Get the values of the current record
        cval = mongo.cursor.value(ns_cursor)

        ## Place the values of the record into the appropriate location in the vectors.
        device[i] <- if( is.null(mongo.bson.value(cval, "device")) ) NA else mongo.bson.value(cval, "device")
        date[i] <- if( is.null(mongo.bson.value(cval, "date")) ) NA else mongo.bson.value(cval, "date")
        dateString[i] <- if(is.null(mongo.bson.value(cval, "dateString")) ) NA else mongo.bson.value(cval, "dateString")
        sgv[i] <- if( is.null( mongo.bson.value(cval, "sgv") ) ) NA else mongo.bson.value(cval, "sgv")
        direction[i] <- if( is.null( mongo.bson.value(cval, "direction") ) ) NA else mongo.bson.value(cval, "direction")
        type[i] <- if( is.null(mongo.bson.value(cval, "type") ) ) NA else mongo.bson.value(cval, "type")
        filtered[i] <- if( is.null( mongo.bson.value(cval, "filtered") ) ) NA else mongo.bson.value(cval, "filtered")
        unfiltered[i] <- if( is.null( mongo.bson.value(cval, "unfiltered") ) ) NA else mongo.bson.value(cval, "unfiltered")
        rssi[i] <- if( is.null( mongo.bson.value(cval, "rssi") ) ) NA else mongo.bson.value(cval, "rssi")
        noise[i] <- if( is.null( mongo.bson.value(cval, "noise") ) ) NA else mongo.bson.value(cval, "noise")
        mbg[i] <- if( is.null(mongo.bson.value(cval, "mbg"))) NA else mongo.bson.value(cval, "mbg")
        slope[i] <- if( is.null( mongo.bson.value(cval, "slope") ) ) NA else mongo.bson.value(cval, "slope")
        intercept[i] <- if( is.null( mongo.bson.value(cval, "intercept") ) ) NA else mongo.bson.value(cval, "intercept")
        scale[i] <- if( is.null( mongo.bson.value(cval, "scale") ) ) NA else mongo.bson.value(cval, "scale")

        ## Increment the cursor to the next record.
        i = i + 1
    }


    ## Data Clean Up ---------------------------------------------------------------

    ## Fixes the date data.
    ## I'm not sure why I have to divide by 1000. If I don't, this won't work.
    date <- as.POSIXct(date/1000, origin = "1970-01-01 00:00:01")


    ## Builds the data.frame -------------------------------------------------------
    entries <- as.data.frame(list( device = device,
                                   date = date,
                                   dateString = dateString,
                                   sgv = sgv,
                                   direction = direction,
                                   type = type,
                                   filtered = filtered,
                                   unfiltered = unfiltered,
                                   rssi = rssi,
                                   noise = noise,
                                   mbg = mbg,
                                   slope = slope,
                                   intercept = intercept,
                                   scale = scale
                                  )
                             )

There is one really annoying aspect about this code. Mongo allows each
record to contain a different number of data elements. Thus, not all
records have a "mbg" element. Thus, rather than just asking the "cval"
variable for the value of the data element, it must first check to see
if it exists. If it doesn't, the client must create the NA (NULL) value.
Mongo can store NULL values. It can also not have a data element and the
two are different and must be handled on the client-side. This took me a
while to figure out.

The next code chunk does some very minimal QA on the "entries" data
frame. If the data frame has 0 rows, it will force the script to stop.
Otherwise, it returns a table with some basic meta-data about the
imported data set.

    if(dim(entries)[1] == 0) stop("Entries variable contains no rows.")

    entries %>%
        summarize(
            "N Rows" = n(),
            "N Days" = length(unique( format.POSIXct(.$date, format="%F" ))),
            "First Day" = min( format.POSIXct(.$date, format="%F" )),
            "Last Day" = max( format.POSIXct(.$date, format="%F" ))
            ) %>%
        pander()

<table>
<colgroup>
<col width="12%" />
<col width="12%" />
<col width="16%" />
<col width="16%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">N Rows</th>
<th align="center">N Days</th>
<th align="center">First Day</th>
<th align="center">Last Day</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">11960</td>
<td align="center">62</td>
<td align="center">43</td>
<td align="center">2015-07-17</td>
</tr>
</tbody>
</table>

The following code chunk returns the number of rows per day, to make
sure the "rig" is properly working and uploading data. This is
restricted to a specific time frame, because we don't want to look at
all of the data in this data frame.

    entries %>%
        filter(date >= "2015-06-20" & date <= "2015-07-07") %>%
        group_by( "Date" = format.POSIXct(.$date, format="%F") ) %>%
        summarize("N Entries" = n() ) %>%
        pander()

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">Date</th>
<th align="center">N Entries</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">2015-06-21</td>
<td align="center">15</td>
</tr>
<tr class="even">
<td align="center">2015-06-22</td>
<td align="center">279</td>
</tr>
<tr class="odd">
<td align="center">2015-06-23</td>
<td align="center">261</td>
</tr>
<tr class="even">
<td align="center">2015-06-24</td>
<td align="center">275</td>
</tr>
<tr class="odd">
<td align="center">2015-06-25</td>
<td align="center">286</td>
</tr>
<tr class="even">
<td align="center">2015-06-26</td>
<td align="center">237</td>
</tr>
<tr class="odd">
<td align="center">2015-06-27</td>
<td align="center">85</td>
</tr>
<tr class="even">
<td align="center">2015-06-28</td>
<td align="center">130</td>
</tr>
<tr class="odd">
<td align="center">2015-06-29</td>
<td align="center">282</td>
</tr>
<tr class="even">
<td align="center">2015-06-30</td>
<td align="center">285</td>
</tr>
<tr class="odd">
<td align="center">2015-07-01</td>
<td align="center">276</td>
</tr>
<tr class="even">
<td align="center">2015-07-02</td>
<td align="center">283</td>
</tr>
<tr class="odd">
<td align="center">2015-07-03</td>
<td align="center">256</td>
</tr>
<tr class="even">
<td align="center">2015-07-04</td>
<td align="center">283</td>
</tr>
<tr class="odd">
<td align="center">2015-07-05</td>
<td align="center">269</td>
</tr>
<tr class="even">
<td align="center">2015-07-06</td>
<td align="center">290</td>
</tr>
</tbody>
</table>

June 21 was the first day we used the new CGM Platinum w/ Share CGM. I
set it up late at night. As a result, it only recorded 15 records on
that day. Karen believes she changed her sensor out on June 27, which is
why there are only 85 records there. We aren't sure what happened on
June 28th. For some reason the rig either was not reading or it wasn't
communicating with the server. We aren't sure. The other days are fairly
consistent, but shows that the number of records recorded does vary by
day.

The final code chunk creates a date-stamped data set. I'll try to add a
new data set periodically to the public data if anyone wants to use it.
Old data sets will remain frozen, for reproducibility purposes.

    ## Saves the data as a CSV file ------------------------------------------------
    ## You are welcome to use the data stored publicly in the data folder.
    file_name <- paste("data/entries-", Sys.Date(), ".csv", sep="")
    write.csv(entries, file_name, row.names = FALSE)

    ## Clean up the session and good-bye.
    rm(list=ls())

[1] If anyone leaves a comment proving me wrong, I will amend this
statement.

[2] I make these comparisons, because it is what I know.