sqlτyped - a macro which infers Scala types from database
=========================================================


> _Towards a perfect impedance match..._
>
> * The types and column names are already defined in the database schema and SQL query. Why not use those and infer types and accessor functions?
>
> * SQL is a fine DSL for many queries. It is the native DSL of relational databases and wrapping it with another DSL is often unncessary (SQL sucks when one has to compose queries, or if you have to be database agnostic).
> 
> 
> **sqlτyped converts SQL string literals into typed functions at compile time.**
> 
> ```"select age, name from person where age > ?"```
>        
>  ==>
>
> ```Int => List[{ age: Int, name: String }]```


Examples
--------

The following examples use schema and data from [test.sql](https://github.com/jonifreeman/sqltyped/blob/master/core/src/test/resources/test.sql)

First some boring initialization... 

Start console: ```sbt```, then ```project sqltyped``` and ```test:console```.

```scala
    import java.sql._
    import sqltyped._
    Class.forName("com.mysql.jdbc.Driver")
    object Tables { trait person; trait job_history }
    object Columns { object name; object age; object salary; }
    implicit val c = Configuration(Tables, Columns)
    implicit def conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/sqltyped", 
                                                    "root", "")
    import Tables._
    import Columns._
```

Now we are ready to query the data.

```scala
    scala> val q = sql("select name, age from person")
    scala> q() map (_ get age)
    res0: List[Int] = List(36, 14)
```

Notice how the type of 'age' was infered to be Int.

```scala
   scala> q() map (_ get salary)
   <console>:24: error: No such key Columns.salary.type
                  q() map (_ get salary)
```

Oops, a compilation failure. Can't access 'salary', it was not selected in the query.

Query results are returned as List of type safe records (think ```List[{name:String, age:Int}]```).
As the above examples showed a field of a record can be accessed with get function: ```row.get(name)```.
Functions ```values``` and ```tuples``` can be used to drop record names and get just the query values.

```scala
   scala> q().values
   res1: List[shapeless.::[String,shapeless.::[Int,shapeless.HNil]]] = 
     List(joe :: 36 :: HNil, moe :: 14 :: HNil)

   scala> q().tuples
   res2: List[(String, Int)] = List((joe,36), (moe,14))
```

Input parameters are parsed and typed.

```scala
    scala> val q = sql("select name, age from person where age > ?")

    scala> q("30") map (_ get name)
    <console>:24: error: type mismatch;
     found   : String("30")
     required: Int
                  q("30") map (_ get name)

    scala> q(30) map (_ get name)
    res4: List[String] = List(joe)
```

Nullable columns are inferred to be Scala Options.

```scala
    scala> val q = sql("""select p.name, j.name as employer, j.started, j.resigned 
                          from person p join job_history j on p.id=j.person order by employer""")
    scala> q().tuples
    res5: List[(String, String, java.sql.Timestamp, Option[java.sql.Timestamp])] = 
      List((joe,Enron,2002-08-02 12:00:00.0,Some(2004-06-22 18:00:00.0)), 
           (joe,IBM,2004-07-13 11:00:00.0,None))
```

Functions are supported too. Note how function 'max' is polymorphic on its argument. For String
column it is typed as String => String etc.

```scala
    scala> val q = sql("select max(name) as name, max(age) as age from person where age > ?")
    scala> q(10).tuples.head
    res6: (Option[String], Option[Int]) = (Some(moe),Some(36))
```

### Analysis ###

So far all the examples have returned results as Lists of records. But with a little bit of query
analysis we can do better. Like, it is quite unnecessary to box the values as records if just one 
column is selected.

```scala
    scala> sql("select name from person").apply
    res7: List[String] = List(joe, moe)

    scala> sql("select age from person").apply
    res8: List[Int] = List(36, 14)
```

Then, some queries are known to return just 0 or 1 values, a perfect match for Option type. 
The following queries return possible result as an Option instead of List. The first query uses 
a uniquely constraint column in its where clause. The second one explicitly wants at most one row.

```scala
    scala> sql("select name from person where id=?").apply(1)
    res9: Some[String] = Some(joe)

    scala> sql("select age from person order by age desc limit 1").apply
    res10: Some[Int] = Some(36)
```

### Inserting data ###

```scala
    scala> sql("insert into person(name, age, salary) values (?, ?, ?)").apply("bill", 45, 30000)
    res1: Int = 1
```

Return value was 1, which means that one row was added. However, often a more useful return value 
is the generated primary key. Table 'person' has an autogenerated primary key column named 'id'. To get
the generated value use a function ```sqlk``` (will be changed to ```sql(..., keys = true)``` once 
[Scala macros](https://issues.scala-lang.org/browse/SI-5920) support default and named arguments).

```scala
    scala> sqlk("insert into person(name, age, salary) values (?, ?, ?)").apply("jill", 45, 30000)
    res2: shapeless.TypeOperators.@@[Long,Tables.person] = 3
```

The return value is a key, hence it's type was tagged to be ```Long @@ person```. See [Tagging](https://github.com/jonifreeman/sqltyped/wiki/User-guide#wiki-tagging).

Inserting multiple values is supported too.

```scala
    scala> sqlk("insert into person(name, age, salary) select name, age, salary from person").apply
    res3: List[shapeless.TypeOperators.@@[Long,Tables.person]] = List(4, 5, 6)
```

Updates work as expected.

```scala
    scala> sql("update person set name=? where age >= ?").apply("joe2", 30)
    res4: Int = 1
```


Documentation
-------------

See [wiki](https://github.com/jonifreeman/sqltyped/wiki).

How to try it?
--------------

### Install ###

Requires Scala 2.10.

sqlτyped is published to Sonatype repositories.

```scala
    "fi.reaktor" %% "sqltyped" % "0.1.0"
```

### Build ###

    git clone https://github.com/jonifreeman/sqltyped.git
    cd sqltyped

Then either:

    mysql -u root -e 'create database sqltyped'
    mysql -u root sqltyped < core/src/test/resources/test.sql
    sbt test // Requires >= 0.12.0 

or:

    sudo -u postgres createuser -P sqltypedtest  // Note, change the password from project/build.scala
    sudo -u postgres createdb -O sqltypedtest sqltyped
    sudo -u postgres psql sqltyped < core/src/test/resources/test-postgresql.sql

