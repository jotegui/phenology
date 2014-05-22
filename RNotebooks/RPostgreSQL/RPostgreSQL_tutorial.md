Brief tutorial on RPostgreSQL
========================================================

Documentation mainly taken from here: https://code.google.com/p/rpostgresql/

Setup
-----

RPostgreSQL is nothing but a wrapper of the DBI library, and a lot of the functions come from this library. Therefore, it is mandatory to have the latest version of the DBI library installed. Then, we can move on to installing RPostgreSQL


```r
# install.packages('DBI')

# install.packages('RPostgreSQL')
```


Now, we just load it as a regular library


```r
library("RPostgreSQL")
```

```
## Loading required package: DBI
```


To be able to query the database, we need to establish a connection with the same. We will do that with the `dbConnect` generic connection function. **ATTENTION:** The connection to the PostgreSQL server in auditrix.colorado.edu must be done through a SSH tunnel. Make sure you have the PuTTY session loaded before trying to connect to the database. If not, you will receive a RS-DBI driver error.





```r
# Parameters for the connection
host = "localhost"
port = 5555
user = ""  # Replace with your PostgreSQL database user name
passwd = ""  # Replace with your PostgreSQL database password
dbname = "phenology"

# First, load the driver
drv <- dbDriver("PostgreSQL")

# Actual command for connecting to the PostgreSQL database
conn <- dbConnect(drv, host = host, port = port, user = user, password = passwd, 
    dbname = dbname)
```


`dbConnect` is a DBI function, a generic method to establish a connection with any database. The first argument, `drv`, uses the previously selected driver in `dbDriver("PostgreSQL")`, which indicates `dbConnect` that the type of connection to be expected is a PostgreSQL database. The rest of the arguments are the values for the PostgreSQL connection.

Basic usage
-----------

There are two ways of querying the database, and their usage depends on the expected output.

### Small amount of records

If the expected resultset has a low number of records, like counting the number of rows in a table, the best methd is `dbGetQuery`. This function submits and executes the query, and extracts the output, all in one operation.


```r
dbGetQuery(conn, "select count(*) as num_records from synonymies")
```

```
##   num_records
## 1          21
```


The first argument is the connection object, the one we created with the `dbConnect` function. The second argument is the actual SQL query. In this case, count every row in table synonymies and assign the name "num_records" to it. The output can be thrown to the screen (as in the previous example), or can be stored in an object.


```r
rs <- dbGetQuery(conn, "select count(*) as num_records from synonymies")
rs
```

```
##   num_records
## 1          21
```

```r
typeof(rs)
```

```
## [1] "list"
```


`dbGetQuery` returns objects as named lists, with each column being an element consisting of an array of values. Each column can be referenced with their name or with their position in the list:


```r
rs <- dbGetQuery(conn, "select * from synonymies limit 5")
rs
```

```
##              current_name              old_name
## 1         Vermivora pinus  Vermivora cyanoptera
## 2      Oreothlypis celata      Vermivora celata
## 3 Oreothlypis ruficapilla Vermivora ruficapilla
## 4     Setophaga americana      Parula americana
## 5      Setophaga petechia    Dendroica petechia
```

```r
typeof(rs)
```

```
## [1] "list"
```

```r
rs$old_name == rs[[2]]
```

```
## [1] TRUE TRUE TRUE TRUE TRUE
```


There is a way of extracting a whole table (if the table is small enough to be handled), and that is by using the `dbReadTable` function. In this case, instead of launching a `select * from table_name` query, we can just say


```r
synonymies <- dbReadTable(conn, "synonymies")
```


### Large amount of records

If we are expecting a large amount of records, then it is better to use the `dbSendQuery` function. This time, the function sends the query to the database, but does not return the full dataset. Instead, it returns a `PostgreSQLResult` object, that works as a cursor to the result set. It is important to capture this cursor into an object:


```r
rs <- dbSendQuery(conn, "select * from synonymies")
rs
```

```
## <PostgreSQLResult:(14248,0,5)>
```


Once we have the result set, we can take a look at the column names and types, to see if the query was good. To do that, we use the `dbColumnInfo` function:


```r
dbColumnInfo(rs)
```

```
##           name    Sclass    type len precision scale nullOK
## 1 current_name character VARCHAR  -1        -1    -1   TRUE
## 2     old_name character VARCHAR  -1        -1    -1   TRUE
```


This function only works if the result set is of type `PostgreSQLResult`. Now, in order to extract the actual values, we can use the `fetch` function, which retrieves a given number of rows from a `PostgreSQLResult` object:


```r
fetch(rs, n = 5)
```

```
##              current_name              old_name
## 1         Vermivora pinus  Vermivora cyanoptera
## 2      Oreothlypis celata      Vermivora celata
## 3 Oreothlypis ruficapilla Vermivora ruficapilla
## 4     Setophaga americana      Parula americana
## 5      Setophaga petechia    Dendroica petechia
```


In this case, we have specified `fetch` to return 5 records from the `rs` object. As any cursor, `fetch` iterates over the result set, without repeating the values, so if we execute again the same command:


```r
fetch(rs, n = 5)
```

```
##              current_name               old_name
## 6  Setophaga pensylvanica Dendroica pensylvanica
## 7      Setophaga coronata     Dendroica coronata
## 8    Setophaga nigrescens   Dendroica nigrescens
## 9        Setophaga virens       Dendroica virens
## 10    Setophaga townsendi    Dendroica townsendi
```


we get records 6 to 10, and there is no way of getting records 1 to 5 back unless we re-execute the query. This is a great option for improving performance on the extraction of large datasets, and it is a good function to integrate in any `for` loop in R.

`fetch` has a special `n` value, -1, that means "return everything left". Running `fetch(rs, n=-1)` in the first place returns all the records in the result set. If we run it now:


```r
fetch(rs, n = -1)
```

```
##               current_name               old_name
## 11  Setophaga occidentalis Dendroica occidentalis
## 12      Setophaga discolor     Dendroica discolor
## 13      Geothlypis formosa      Oporonis formosus
## 14       Setophaga citrina       Wilsonia citrina
## 15      Cardellina pusilla       Wilsonia pusilla
## 16   Cardellina canadensis    Wilsonia canadensis
## 17    Artemisiospiza belli       Amphispiza belli
## 18 Parkesia noveboracensis Seiurus noveboracensis
## 19      Setophaga magnolia     Dendroica magnolia
## 20      Parkesia motacilla      Seiurus motacilla
## 21      Geothlypis tolmiei      Oporornis tolmiei
```


we extract the missing rows, those from line 11 to the last.

Iterating to the last row in a result set sets that object free for re-assignment. In other words, if a cursor has started fetching records, but has not yet finished, the object is "blocked", and it cannot be assigned to another result set.


```r
rs <- dbSendQuery(conn, "select * from synonymies")
fetch(rs, 5)
```

```
##              current_name              old_name
## 1         Vermivora pinus  Vermivora cyanoptera
## 2      Oreothlypis celata      Vermivora celata
## 3 Oreothlypis ruficapilla Vermivora ruficapilla
## 4     Setophaga americana      Parula americana
## 5      Setophaga petechia    Dendroica petechia
```

```r
rs <- dbSendQuery(conn, "select * from pheno_species")
```

```
## Error: RS-DBI driver: (connection with pending rows, close resultSet
## before continuing)
```


There is a function that tells whether or not the result set has finished processing (fetching) the results, and that is `dbHasCompleted`:


```r
dbHasCompleted(rs)
```

```
## [1] FALSE
```


And to check how many rows have been fetched so far, we can use `dbGetRowCount`


```r
dbGetRowCount(rs)
```

```
## [1] 5
```


Finally, to solve this issue, there is a function that closes the result set and releases the object. That function is `dbClearResult`


```r
dbClearResult(rs)
```

```
## [1] TRUE
```

```r
rs <- dbSendQuery(conn, "select * from pheno_species limit 10")
fetch(rs, n = -1)
```

```
##                   latin
## 1  Empidonax difficilis
## 2         Piranga rubra
## 3   Coccyzus americanus
## 4    Passerina caerulea
## 5    Spizella passerina
## 6  Setophaga nigrescens
## 7   Protonotaria citrea
## 8   Contopus sordidulus
## 9      Pipilo chlorurus
## 10  Empidonax virescens
```



Some PostgreSQL management functions
------------------------------------

This set of functions might be handy for the management of the database:

`dbListTables` returns a list of the available tables in the database:


```r
dbListTables(conn)[1:10]  # To avoid cluttering, I have limited the output to 10 elements
```

```
##  [1] "nagrid"                         "selected_absences"             
##  [3] "selected_presences"             "presences_empidonax_difficilis"
##  [5] "absences_empidonax_difficilis"  "cells_empidonax_difficilis"    
##  [7] "cells_passerina_caerulea"       "allpoints"                     
##  [9] "presences_piranga_ludoviciana"  "presences"
```


We might also want to check if a table exists before querying it, with `dbExistsTable`:


```r
dbExistsTable(conn, "ebird")  # Should exist
```

```
## [1] TRUE
```

```r
dbExistsTable(conn, "foo_table")  # Should not exist
```

```
## [1] FALSE
```


We also might want to know the structure of a table. To know what columns are in a table, we can use the `dbListFields` function


```r
dbListFields(conn, "jetz_maps")
```

```
##  [1] "specid"               "latin"                "occcode"             
##  [4] "date"                 "editsinfo"            "citation"            
##  [7] "notes"                "origin"               "raster"              
## [10] "the_geom"             "the_geom_webmercator" "cartodb_id"          
## [13] "created_at"           "updated_at"           "iucn_seasonality"
```



Finishing a connection
----------------------

When the processing has finished, make sure you close the connection


```r
dbDisconnect(conn)
```

```
## [1] TRUE
```


And release the resources used by the driver


```r
dbUnloadDriver(drv)
```

```
## [1] TRUE
```


