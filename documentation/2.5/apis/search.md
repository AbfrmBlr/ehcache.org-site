---
---
# Ehcache Search API


 

## Introduction

The Ehcache Search API allows you to execute arbitrarily complex queries against either a standalone cache or a Terracotta
clustered cache with pre-built indexes. This allows development of alternative indexes on values so that data can be looked up based on multiple criteria instead of just keys.

Searchable attributes may be extracted from both keys and values. Keys, values,
or summary values (Aggregators) can all be returned.
 Here is a simple example: Search for 32-year-old males and return the cache values.

    Results results = cache.createQuery().includeValues()
      .addCriteria(age.eq(32).and(gender.eq("male"))).execute();

## What is Searchable?

Searches can be performed against Element keys and values.

Element keys and values are made searchable by extracting attributes with supported search types out of the keys and values.
It is the attributes themselves which are searchable.


## How to Make a Cache Searchable


### By Configuration

Caches are made searchable by adding a `<searchable/>` tag to the ehcache.xml.

~~~
<cache name="cache2" maxEntriesLocalHeap="10000" eternal="true" overflowToDisk="false">
<searchable/>
</cache>
~~~

This configuration will scan keys and vales and if they are of supported search types, add them as
attributes called "key" and "value" respectively. If you do not want automatic indexing of keys and values
you can disable it with:

~~~
<cache name="cache3" ...>
<searchable keys="false" values="false">
   ...
</searchable>
</cache>
~~~

You might want to do this if you have a mix of types for your keys or values. The automatic indexing will throw
an exception if types are mixed.
Often keys or values will not be directly searchable and instead you will need to extract searchable attributes out of them.
The following example shows this more typical case.  Attribute Extractors are explained in more detail in the following section.

~~~
<cache name="cache3" maxEntriesLocalHeap="10000" eternal="true" overflowToDisk="false">
<searchable>
   <searchAttribute name="age" class="net.sf.ehcache.search.TestAttributeExtractor"/>
   <searchAttribute name="gender" expression="value.getGender()"/>
</searchable>
</cache>
~~~


### Programmatically

The following example shows how to programmatically create the cache configuration, with search attributes.

    Configuration cacheManagerConfig = new Configuration();
    CacheConfiguration cacheConfig = new CacheConfiguration("myCache", 0).eternal(true);
    Searchable searchable = new Searchable();
    cacheConfig.addSearchable(searchable);
    // Create attributes to use in queries.
    searchable.addSearchAttribute(new SearchAttribute().name("age"));
    // Use an expression for accessing values.
    searchable.addSearchAttribute(new SearchAttribute()
        .name("first_name")
        .expression("value.getFirstName()"));
    searchable.addSearchAttribute(new SearchAttribute().name("last_name").expression("value.getLastName()"));
         searchable.addSearchAttribute(new SearchAttribute().name("zip_code").expression("value.getZipCode()"));
    cacheManager = new CacheManager(cacheManagerConfig);
    cacheManager.addCache(new Cache(cacheConfig));
    Ehcache myCache = cacheManager.getEhcache("myCache");
    // Now create the attributes and queries, then execute.
    ...

To learn more about the Ehcache Search API, see the `net.sf.ehcache.search*` packages in this [Javadoc](http://ehcache.org/apidocs/index.html).


## Attribute Extractors

  Attributes are extracted from keys or values. This is done during search or, if using Distributed Ehcache, on `put()` into the cache using `AttributeExtractor`s.
Extracted attributes must be one of the following supported types:

*   Boolean
*   Byte
*   Character
*   Double
*   Float
*   Integer
*   Long
*   Short
*   String
*   java.util.Date
*   java.sql.Date
*   Enum

If an attribute cannot be extracted due to not being found or being the wrong type, an AttributeExtractorException is thrown on search execution or, if using Distributed Ehcache, on `put()`.


### Well-known Attributes

The parts of an Element that are well-known attributes can be referenced by some predefined, well-known names.
If a keys and/or value is of a supported search type, they are added automatically as attributes with the names
"key" amd "value".
These well-known attributes have convenience constant attributes made available on the `Query` class.
So, for example, the attribute for "key" may be referenced in a query by `Query.KEY`. For even greater readability it is
recommended to statically import so that in this example you would just use `KEY`.

| Well-known Attribute Name | Attribute Constant |
| --- | --- |
| key               |     Query.KEY |
| value             |     Query.VALUE |


### ReflectionAttributeExtractor

The ReflectionAttributeExtractor is a built-in search attribute extractor which uses JavaBean conventions and also understands a simple form of expression. Where a JavaBean property is available and it is of a searchable type, it can be simply declared using:

    <cache>
      <searchable>
        <searchAttribute name="age"/>
      </searchable>
    </cache>

 Finally, when things get more complicated, we have an expression language using method/value dotted expression chains. The expression chain must start with one of either "key", "value", or "element". From the starting object a chain of either method calls or field names follows. Method calls and field names can be freely mixed in the chain. Some more examples:

    <cache>
      <searchable>
        <searchAttribute name="age" expression="value.person.getAge()"/>
      </searchable>
    </cache>
    <cache>
      <searchable>
         <searchAttribute name="name" expression="element.toString()"/>
      </searchable>
    </cache>

 The method and field name portions of the expression are case sensitive.


### Custom AttributeExtractor

 In more complex situations you can create your own attribute extractor by implementing the AttributeExtractor interface. Providing your extractor class is shown in the following example:

    <cache name="cache2" maxEntriesLocalHeap="0" eternal="true" overflowToDisk="false">
      <searchable>
         <searchAttribute name="age" class="net.sf.ehcache.search.TestAttributeExtractor"/>
      </searchable>
    </cache>

 If you need to pass state to your custom extractor you may do so with properties as shown in the following example:

    <cache>
      <searchable>
        <searchAttribute name="age"
        class="net.sf.ehcache.search.TestAttributeExtractor"
        properties="foo=this,bar=that,etc=12" />
      </searchable>
    </cache>

If properties are provided, then the attribute extractor implementation must have a public constructor that accepts a single `java.util.Properties` instance.

## Query API
Ehcache Search introduces a fluent Object Oriented query API, following DSL principles, which should feel familiar and natural to Java programmers.
Here is a simple example:

    Query query = cache.createQuery().addCriteria(age.eq(35)).includeKeys().end();
    Results results = query.execute();

### Using Attributes in Queries
 If declared and available, the well-known attributes are referenced by their names or the convenience attributes are used
  directly as shown in this example:

    Results results = cache.createQuery().addCriteria(Query.KEY.eq(35)).execute();
    Results results = cache.createQuery().addCriteria(Query.VALUE.lt(10)).execute();

Other attributes are referenced by the names given them in the configuration. For example:

    Attribute<Integer> age = cache.getSearchAttribute("age");
    Attribute<String> gender = cache.getSearchAttribute("gender");
    Attribute<String> name = cache.getSearchAttribute("name");


### Expressions

The Query to be searched for is built up using Expressions.
Expressions include logical operators such as &lt;and&gt; and &lt;or&gt;. It also includes comparison operators such as &lt;ge&gt; (&gt;=), &lt;between&gt;, and &lt;like&gt;.
`addCriteria(...)` is used to add a clause to a query. Adding a further clause automatically &lt;and&gt;s the clauses.

    query = cache.createQuery().includeKeys().addCriteria(age.le(65)).add(gender.eq("male")).end();

Both logical and comparison operators implement the `Criteria` interface.
To add a criteria with a different logical operator, explicitly nest it within a new logical operator Criteria Object. For example, to check for age = 35 or gender = female, do the following:

    query.addCriteria(new Or(age.eq(35),
                 gender.eq(Gender.FEMALE))
                );

More complex compound expressions can be further created with extra nesting.
See the [Expression JavaDoc](http://ehcache.org/xref/net/sf/ehcache/search/expression/package-frame.html) for a complete list.

### List of Operators
Operators are available as methods on attributes, so they are used by adding a ".". For example, "lt" means "less than" and is used as `age.lt(10)`, which is a shorthand way of saying `new LessThan(10)`.
The full listing of operator shorthand is shown below.

| Shorthand | Criteria Class | Description
|----------|--------------|--------------
| and    |  And          | The Boolean AND logical operator
| between | Between      | A comparison operator meaning between two values
| eq     | EqualTo       | A comparison operator meaning Java "equals to" condition
| gt     | GreaterThan   | A comparison operator meaning greater than.
| ge     | GreaterThanOrEqual | A comparison operator meaning greater than or equal to.
| in     | InCollection  | A comparison operator meaning in the collection given as an argument
| lt     | LessThan      | A comparison operator meaning less than.
| le     | LessThanOrEqual| A comparison operator meaning less than or equal to
| ilike  | ILike          | A regular expression matcher. '?' and "*" may be used. Note that placing a wildcard in front of the expression will cause a table scan. ILike is always case insensitive.
| not    | Not           | The Boolean NOT logical operator
| ne     | NotEqualTo    | A comparison operator meaning not the Java "equals to" condition
| or     | Or            | The Boolean OR logical operator

### Making Queries Immutable

 By default, a query can be executed and then modified and re-executed. If `end` is called,
 the query is made immutable.


## Search Results

Queries return a `Results` object which contains a list of objects of class `Result`. Each `Element` in the cache found with a query will be represented as a `Result` object. So if a query finds
350 elements there will be 350 `Result` objects. An exception to this would be if no keys or attributes are included but
aggregators are -- in this case, there will be exactly one `Result` present.
 
A Result object can contain:

*    the Element key - when `includeKeys()` is added to the query,
*    the Element value - when `includeValues()` is added to the query,
*    predefined attribute(s) extracted from an Element value - when `includeAttribute(...)` is added to the query. To access an attribute from Result, use `getAttribute(Attribute<T> attribute`.
*    aggregator results

    Aggregator results are summaries computed for the search. They are available through `Result.getAggregatorResults` which returns a list of `Aggregator`s in the same order in which they were used in the `Query`.

### Aggregators
Aggregators are added with `query.includeAggregator(\<attribute\>.\<aggregator\>)`.
For example, to find the sum of the age attribute:

    query.includeAggregator(age.sum());

See the [Aggregators JavaDoc](http://ehcache.org/xref/net/sf/ehcache/search/aggregator/package-frame.html) for a complete list.

### Ordering Results
Query results may be ordered in ascending or descending order by adding an `addOrderBy` clause to the query, which takes
as parameters the attribute to order by and the ordering direction.
For example, to order the results by ages in ascending order

    query.addOrderBy(age, Direction.ASCENDING);


### Limiting the Size of Results

Either all results can be returned using `results.all()` to get them all in one chunk, or page results can be returned with ranges
using `results.range(int start, int count)`.

By default a query will return an unlimited number of results. For example the following
query will return all keys in the cache.

    Query query = cache.createQuery();
    query.includeKeys();
    query.execute();

If too many results are returned, it could cause an OutOfMemoryError
The `maxResults` clause is used to limit the size of the results.
For example, to limit the above query to the first 100 elements found:

    Query query = cache.createQuery();
    query.includeKeys();
    query.maxResults(100);
    query.execute();

When you are done with the results, call `discard()` to free up resources.
In the distributed implementation with Terracotta, resources may be used to hold results for paging or return.

### Interrogating Results
To determine what was returned by a query, use one of the interrogation methods on `Results`:
 
* `hasKeys()`
* `hasValues()`
* `hasAttributes()`
* `hasAggregators()`


## Sample Application
We have created [a simple standalone sample application](http://github.com/sharrissf/Ehcache-Search-Sample/downloads/) with few dependencies for you to easily get started with Ehcache Search. You can also check out the source:

    git clone git://github.com/sharrissf/Ehcache-Search-Sample.git

The [Ehcache Test Sources](http://ehcache.org/xref-test/net/sf/ehcache/search/package-summary.html) page has further examples
on how to use each Ehcache Search feature.

## Scripting Environments
Ehcache Search is readily amenable to scripting. The following example shows how to use it with BeanShell:

    Interpreter i = new Interpreter();
    //Auto discover the search attributes and add them to the interpreter's context
    Map<String, SearchAttribute> attributes = cache.getCacheConfiguration().getSearchAttributes();
    for (Map.Entry<String, SearchAttribute> entry : attributes.entrySet()) {
    i.set(entry.getKey(), cache.getSearchAttribute(entry.getKey()));
    LOG.info("Setting attribute " + entry.getKey());
    "/>
    //Define the query and results. Add things which would be set in the GUI i.e.
    //includeKeys and add to context
    Query query = cache.createQuery().includeKeys();
    Results results = null;
    i.set("query", query);
    i.set("results", results);
    //This comes from the freeform text field
    String userDefinedQuery = "age.eq(35)";
    //Add the stuff on that we need
    String fullQueryString = "results = query.addCriteria(" + userDefinedQuery + ").execute()";
    i.eval(fullQueryString);
    results = (Results) i.get("results");
    assertTrue(2 == results.size());
    for (Result result : results.all()) {
    LOG.info("" + result.getKey());
    "/>

## Concurrency Considerations
Unlike cache operations which has selectable concurrency control and/or transactions, the Search API does not. This may change
in a future release, however our survey of prospective users showed that concurrency control in search indexes was not sought after.
The indexes are eventually consistent with the caches.

### Index Updating
Indexes will be updated asynchronously, so their state will lag slightly behind the state of the cache. The only exception is when the updating thread then performs a search.

For caches with concurrency control, an index will not reflect the new state of the cache until:

* The change has been applied to the cluster.
* For a cache with transactions, when `commit` has been called.

### Query Results
There are several ways unexpected results could present:

*   A search returns an Element reference which no longer exists.
*   Search criteria select an Element, but the Element has been updated and a new Search would no longer match the Element.
*   Aggregators, such as sum(), might disagree with the same calculation done by redoing the calculation yourself by re-accessing the cache for each key and repeating the calculation.
*   `includeValues` returns values. Under the covers the index contains a server value reference. The reference gets returned with the search and Terracotta supplies the matching value. Because the cache is always updated before the search index it is possible that a value reference may refer to a value that has been removed from the cache. If this happens the value will be null but the key and attributes which were supplied by the now stale cache index will be non-null. Because values in Ehcache are also allowed to be null, you cannot tell whether your value is null because it has been removed from the cache since the index was last updated or because it is a null value.

### Recommendations
Because the state of the cache can change between search executions it is recommended to add all of the Aggregators you want for a query at once so that the returned aggregators are consistent.
Use null guards when accessing a cache with a key returned from a search.

## Implementations

### Standalone Ehcache
The standalone Ehcache implementation does not use indexes. It uses fast iteration of the cache instead, relying on the very fast access to do the equivalent of a table scan for each query. Each element in the cache is only visited once. Attributes are not extracted ahead of time. They are done during query execution.

#### Performance
Search operations perform in O(n) time.
Checkout this [Maven-based performance test](http://svn.terracotta.org/svn/forge/offHeap-test/) showing standalone cache performance. This test shows search performance of an average of representative queries at 10ms per 10,000 entries. So, a typical query would take 1 second for a 1,000,000 entry cache. Accordingly, standalone implementation is suitable for development and testing.

For production it is recommended to only standalone search for caches that are less than 1 million elements. Performance of different `Criteria` vary. For example, here are some queries and their execute times on a 200,000 element cache. (Note that these results are all faster than the times given above because they execute a single Criteria).

<pre><code>final Query intQuery = cache.createQuery();
  intQuery.includeKeys();
  intQuery.addCriteria(age.eq(35));
  intQuery.end();
Execute Time: 62ms
final Query stringQuery = cache.createQuery();
  stringQuery.includeKeys();
  stringQuery.addCriteria(state.eq("CA"));
  stringQuery.end();
Execute Time: 125ms
final Query iLikeQuery = cache.createQuery();
  iLikeQuery.includeKeys();
  iLikeQuery.addCriteria(name.ilike("H*"));
  iLikeQuery.end();
Execute Time: 180ms
</code></pre>

### Ehcache Backed by the Terracotta Server Array
This implementation uses indexes which are maintained on each Terracotta server. In Ehcache EX the index is
on a single active server. In Ehcache FX the cache is sharded across the number of active nodes in the cluster. The index
for each shard is maintained on that shard's server.
Searches are performed using the Scatter-Gather pattern. The query executes on each node and the results are then aggregated
back in the Ehcache that initiated the search.

#### Performance
Search operations perform in O(log n / number of shards) time.
Performance is excellent and can be improved simply by adding more servers to the FX array.

#### Network Effects
Search results are returned over the network. The data returned could potentially be very large, so
techniques to limit return size are recommended such as:

* Limiting the results with `maxResults` or using the paging API `Results.range(int start, int length)`.
* Only including the data you need. Specifically only use `includeKeys()` and/or `includeAttribute()` if those values are actually required for your application logic.
* Using a built-in `Aggregator` function when you only need a summary statistic. `includeValues` rates a special mention. Once a query requiring values is executed we push the values from the server to the Ehcache CacheManager which requested it in batches for network efficiency. This is done ahead as soon as possible reducing the risk that `Result.getValue()` might have to wait for data over the network.
* Turning off key and value indexing if you are not going to search against them as they will just chew up space on the server.

    You do this as follows:

        <cache name="cache3" ...>
          <searchable keys="false" values="false">
          ...
          </searchable>
        </cache>