---
layout: post
title: Spark Datasets and type-safety
tags:
 - Spark 2
 - Datasets
 - type-safety
---

Spark 2.0 has introduced the Datasets API (in a stable version). Datasets promise is to to add type-safety to dataframes, that were more a SQL oriented API. I used to rely on the more "low level" RDD (distributed Spark collections) on some parts of my code when I wanted more type-safety but it lacks some of the dataframe optimisations (for example on groupBy operations). So the recommended way is now to use datasets everywhere, except if you're doing something very specific and if you need to use the low level RDD functions. Let's see how it looks.

This is the classical Word Count using RDD :

    ```scala
    val textFile = sc.textFile("hdfs://...")
    val counts = textFile.flatMap(line => line.split(" "))
                 .map(word => (word.toLowerCase, 1)) // put 1 with each word istance
                 .reduceByKey((accumulator, current) => accumulator+current) // add all words, grouped by value (by key)
    ```

And the word count with dataframes :

```scala
val df = sparkSession.read.text("hdfs://...")
val wordsDF = df.select(split(df("value")," ").alias("words"))
val wordDF = wordsDF.select(explode(wordsDF("words")).alias("word"))
val wordCountDF = wordDF.groupBy("word").count
```

The drawback is that you loose the type information and fields name, so you need to use columns names as strings, which can be error prone.
Also, my personal opinion for this kind of example is that map/flatMap operations are easier to read.

With datasets, you have a mix of RDD and Dataframes, with dataframes like high level API (groupBy, count...) while retaining some type information :

```scala
sparkSession.read.text("hdfs://...").as[String]
  .flatMap(_.split(" "))
  .groupByKey(_.toLowerCase())
  .count()
```

This version is the easiest to read and to understand.
So, it seems perfect! But... if you try to do some more complex operations on rows, you will quick fall back on the dataset API.

For example, if you want to order the words by count :

```scala
sparkSession.read.text("hdfs://...").as[String]
  .flatMap(_.split(" "))
  .filter(_ != "")
  .groupByKey(_.toLowerCase())
  .count()
  .orderBy($"count(1)".desc) // WTF
```

The count column name is pretty weird, but it gets a little better if you use groupBy (which non type-safe) instead of groupByKey :

```scala
sparkSession.read.text("hdfs://...").as[String]
  .flatMap(_.split(" "))
  .filter(_ != "")
  .groupBy($"value") // value is the default column name
  .count()
  .orderBy($"count".desc)
```

You will also have this problem if you  want to join 2 datasets (which is pretty common) :

```scala

//creates dataset from a Seq
val departments = Seq(
  Department(1, "rh"),
  Department(2, "it"),
  Department(3, "marketing")
).toDS

val people = Seq(
  Person("jane", 28, "female", 2000, 2),
  Person("bob", 31, "male", 2000, 1),
  //...
).toDS

people.joinWith(departments, people("deptId") === departments("id"))
```

Here again you have to pass the column names as strings.
To be honest, it's still easier than the RDD equivalent, where you would have to create (key, value) Pair RDDs to be able to join data :

```scala
val departmentsById = departments.rdd.map{ department =>
  (department.id, department)
}

val peopleByDepartmentId = people.rdd.map{ person =>
  (person.deptId, person)
}

peopleByDepartmentId.join(departmentsById)
```

It get worst if you do some more complex values :

```scala
people.filter(_.age > 30)
    .join(departments, people("deptId") === departments("id"))
    .group(departments("name"), $"gender")
    .agg(avg(people("salary")), max(people("age")))
```

That can be desapointing... but, there is a library named [Frameless](link here), based on the awesome Shapeless, that can add more type-safety te datasets!



```scala
import frameless.TypedDataset

val departments = TypedDataset.create(Seq(
Department(1, "rh"),
Department(2, "it"),
Department(3, "marketing")
))

val people = TypedDataset.create(Seq(
Person("jane", 28, "female", 2000, 2),
Person("bob", 31, "male", 2000, 1),
//...
))

val joined = people.join(departments, people('deptId), departments('id))

// val joined = people.join(departments, people('detppid), departments('id)) <-- Won't compile as 'detppid symbol is wrong
```

As you can see one the second line, if you provide a bad column name, you will have a compilation error. Pretty great isn't it?


Another problem with regular datasets, is that they can produce null values :

```scala
    departments.joinWith(people, departments("id") === people("deptId"), "left_outer").show
```

The output is :

```
//    +-------------+--------------------+
   //    |           _1|                  _2|
   //    +-------------+--------------------+
   //    |       [1,rh]|[linda,37,female,...|
   //    |       [1,rh]|[john,45,male,300...|
   //    |       [1,rh]|[bob,35,male,2200,1]|
   //    |       [1,rh]|[bob,31,male,2000,1]|
   //    |       [2,it]|[joe,40,male,3000,2]|
   //    |       [2,it]|[jane,28,female,2...|
   //    |[3,marketing]|                null| <--- WARNING : null
   //      +-------------+--------------------+
   //
```

With Frameless, you won't have this problem :

```scala
    val leftJoined: TypedDataset[(Department, Option[Person])] = departments.joinLeft(people, departments('id), people('deptId))
```

Frameless is still described as a "proof of concept" and does not cover all dataset operations, but I think it's an intersting library to follow!
