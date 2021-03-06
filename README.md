# Problem with sparks classpath after including geomesa dependencies into fat jar

Geomesa seems to pull in some hadoop /xml ... transitive dependencies clashing with spark.
```
compileOnly "org.apache.spark:spark-core_2.11:2.2.0.2.6.4.9-3"
compileOnly "org.apache.spark:spark-sql_2.11:2.2.0.2.6.4.9-3"
compileOnly "org.scala-lang:scala-library:2.11.12"

compile "org.locationtech.geomesa:geomesa-spark-sql_2.11:2.0.1"
compile "org.locationtech.geomesa:geomesa-fs-datastore_2.11:2.0.1"
compile "org.locationtech.geomesa:geomesa-fs-storage-orc_2.11:2.0.1"
```
as is crashing with:
```
javax.xml.parsers.ParserConfigurationException: Feature 'http://apache.org/xml/features/xinclude' is not recognized.
```


## to reproduce
```
./gradlew shadowJar
spark-shell --master 'local[2]' \
		--jars build/libs/geomesaDependenciesProblem-0.0.1-SNAPSHOTo-all.jar
```

How can this be fixed? What are suitable/sensible exclusion rules?

I believe it is fine that geomesa is contained in the fat jar, but existing dependenies could maybe be removed? However, from the documentation I know that geomesa is sometimes requiring more up to date libraries (in particular kryo) so I am unsure what settings to choose here.

In particular, I am concerned about:
- kryo
- org.apache.hadoop:*
- org.apache.orc
- various XML and JSON parsers (which are not shaded)

Would you suggest shading? If so which libraries should be shaded?

See https://github.com/geoHeil/geomesaDependenciesProblem for details

## debugging dependencies

```
gradle dependencyInsight --dependency xerces                                                               [±master ●●]

> Task :dependencyInsight
xerces:xercesImpl:2.9.1 (conflict resolution)
   variant "runtime" [
      Requested attributes not found in the selected variant:
         org.gradle.usage = java-api
   ]
\--- org.apache.hadoop:hadoop-hdfs:2.7.3.2.6.4.9-3
     \--- org.apache.hadoop:hadoop-client:2.7.3.2.6.4.9-3
          \--- org.apache.spark:spark-core_2.11:2.2.0.2.6.4.9-3
               +--- compileClasspath
               +--- org.apache.spark:spark-sql_2.11:2.2.0.2.6.4.9-3
               |    \--- compileClasspath
               \--- org.apache.spark:spark-catalyst_2.11:2.2.0.2.6.4.9-3
                    \--- org.apache.spark:spark-sql_2.11:2.2.0.2.6.4.9-3 (*)

xerces:xercesImpl:2.4.0 -> 2.9.1
   variant "runtime" [
      Requested attributes not found in the selected variant:
         org.gradle.usage = java-api
   ]
\--- com.vividsolutions:jts:1.12
     \--- org.jaitools:jt-utils:1.4.0
          +--- org.geotools:gt-coverage:18.0
          |    \--- org.geotools:gt-process:18.0
          |         \--- org.geotools:gt-process-feature:18.0
          |              +--- org.locationtech.geomesa:geomesa-feature-kryo_2.11:2.0.1
          |              |    \--- org.locationtech.geomesa:geomesa-feature-all_2.11:2.0.1
          |              |         \--- org.locationtech.geomesa:geomesa-spark-core_2.11:2.0.1
          |              |              \--- org.locationtech.geomesa:geomesa-spark-sql_2.11:2.0.1
          |              |                   \--- compileClasspath
          |              \--- org.locationtech.geomesa:geomesa-feature-common_2.11:2.0.1
          |                   +--- org.locationtech.geomesa:geomesa-cqengine-datastore_2.11:2.0.1
          |                   |    \--- org.locationtech.geomesa:geomesa-spark-sql_2.11:2.0.1 (*)
          |                   +--- org.locationtech.geomesa:geomesa-feature-all_2.11:2.0.1 (*)
          |                   +--- org.locationtech.geomesa:geomesa-feature-kryo_2.11:2.0.1 (*)
          |                   \--- org.locationtech.geomesa:geomesa-feature-avro_2.11:2.0.1
          |                        \--- org.locationtech.geomesa:geomesa-feature-all_2.11:2.0.1 (*)
          \--- org.jaitools:jt-zonalstats:1.4.0
               \--- org.geotools:gt-coverage:18.0 (*)

(*) - dependencies omitted (listed previously)

A web-based, searchable dependency report is available by adding the --scan option.

BUILD SUCCESSFUL in 0s
1 actionable task: 1 executed

```

adding an exclusion rule
```
compile("org.locationtech.geomesa:geomesa-spark-sql_2.11:2.0.1") {
        exclude group: 'xerces', module: 'xercesImpl'
    }
```
fixes the problem - but I am unsure if this (i.e. the more current version at runtime) will break geotools.

## link

See https://github.com/geoHeil/geomesaDependenciesProblem for details
