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