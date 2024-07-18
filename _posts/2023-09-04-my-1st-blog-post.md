# Using MongoDB Bulk Operations in Spring Data MongoDB

This article describes how we used MongoDB bulk operations in Spring Data MongoDB to improve the performance of 
our application significantly. The performance improvement ranges from 10x to 100x for our different use cases
running on AWS. For local Java code using a remote MongoDB cluster, the improvement is even bigger. 
Our tests also prove that using bulk operations is even more important than distributing the processing to 
multiple pods using Kafka.

## What We Need to Do

### Business Requirements

- Need to batch process up to 5M+ documents as quickly as possible, daily.
- Process real time events. A single event may result in up to 50K+ document upserts.

### Technical Implementation
- MongoDB Atlas Cluster (M20-M50)
- Spring Boot for Java
- Spring Data MongoDB (https://www.mongodb.com/compatibility/spring-boot)

### Challenges We Faced
- Processing 5M+ documents took hours.
- Occasionally multiple attempts to process upstream events.

## Mixing MongoRepository and MongoOperations

Like any other Spring Data framework, Spring Data MongoDB provides MongoRepository for CRUD operations.
Although saveAll() allows us to do bulk insert in an ideal situation (please see the following code snippet), 
this method will upsert the items one by one if the primary key field ( annotated with @ID ) of any item is not 
null. This causes significant performance downgrade. 
As a result, in our project we decided to implement bulk upsert using MongoOperations and to implement other CRUD 
operations in the Repository interface.

### saveAll() in Spring Data MongoDB:

```
public <S extends T> List<S> saveAll(Iterable<S> entities) {
  Streamable<S> source = Streamable.of(entities);
  boolean allNew = source.stream().allMatch(entityInformation::isNew);

  if (allNew) {
    List<S> result = source.stream().collect(Collectors.toList());
    return new ArrayList<>(mongoOperations.insert(result, entityInformation.getCollectionName()));
  }

  return source.stream().map(this::save).collect(Collectors.toList());
}
```

### The Whole Picture of Our Implementation

In this article we use the following entity class:
```
@Document
public class Product {
    @Id
    private Integer productId;      // in MongoDB, it is mapped to '_id'
    private String partNumber;
    private Boolean isDeleted;

    .
    .
    .
    
    private LocalDateTime lastModifiedOn;
}
```
To take advantage of both MongoRepository and MongoOperations, for our ***product*** MongoDB collection, the following interface shows the whole picture:

```
public interface ProductRepository extends ProductDao, MongoRepository<Product, Integer> {

    .
    .
    .

}
```

Here, the idea is to use MongoRepository as much as possible, when performance is not an issue. This makes our code cleaner. Meanwhile the above strategy also allows us to use MongoOperations for bulk upsert.



### Implementing Bulk Upsert

```
@Repository
public class ProductDaoImpl implements ProductDao {
    private final MongoOperations mongoOperations;

    @Autowired
    public ProductDaoImpl(MongoOperations mongoOperations) {
        this.mongoOperations = mongoOperations;
    }

    @Override
    @Retryable(retryFor = Exception.class, maxAttempts = 4, backoff = @Backoff(delay = 2000, maxDelay = 16000, multiplier = 2))
    public BulkWriteResult bulkUpsert(Collection<Product> products) {
        if (products == null || products.isEmpty())
            return BulkWriteResult.acknowledged(0, 0, 0, 0, List.of(), List.of());

        BulkOperations bulkOperations = mongoOperations.bulkOps(BulkOperations.BulkMode.UNORDERED, Product.class);
        products.forEach(product -> {
            Query query = new Query().addCriteria(Criteria.where(ID).is(product.getProductId()));
            bulkOperations.replaceOne(query, product, FindAndReplaceOptions.options().upsert());
        });

        return bulkOperations.execute();
    }
}
```

The above bulkUpsert() implementation actually generates a native MongoDB
[bulkWrite() call](https://www.mongodb.com/docs/manual/reference/method/db.collection.bulkWrite/) and one or 
multiple replaceOne() calls will be included in the bulkWrite() call like the following:

```
db.product.bulkWrite([
    { replaceOne :  {  
      {_id: 1234}, {_id: 1234, partNumber: "part1", ...... },  { "upsert" : true} }
    },
    { replaceOne :  {  
      {_id: 2234}, {_id: 2234, partNumber: "part2", ...... },  { "upsert" : true} }
    },
    .
    .
    .
  ],
  { ordered : false }
)
```

### Implementing other CRUD methods

The following code snippet shows how we implement some other CRUD methods using MongoRepository:

```
public interface ProductRepository extends ProductDao, MongoRepository<Product, Integer> {

    @Query(value = "{ 'isDeleted' : false }", fields = "{ '_id' : 1}")
    List<Product> findAllNotDeletedProducts();

    @Query("{ '_id' : {$in: ?0}}")                      // filter part
    @Update("{ $set : { 'isDeleted' : true}, $currentDate: { 'lastModifiedOn' : true } }")
    void softDeleteByIds(List<Integer> productIds);

}
```

Here the 1st method is the same as the following native query:

```
db.product.find( {"isDeleted": false}, {"_id": 1})
```

please note that the projection part, `{"_id": 1}`, can significantly improve performance because of 2 factors: 1.
MongoDB returns far fewer bytes, only a single Integer per document. 2. If a compound index exists (`key: { isDeleted: 1, _id: 1}`), MongoDB will 
not scan any document and will simply return documents only containing "_id", because it is in the index. In other
words, this is a [covered query]( https://www.mongodb.com/docs/manual/core/query-optimization/#covered-query ).

The 2nd method in the above code snippet is essentially similar to the following:

```
db.product.updateMany( {"_id": {$in: [1234, 2234, 3234]}}, {$set: {"isDeleted": false}, $currentDate: { "lastModifiedOn" : true }} )
```

## Performance Improvement

### Benchmark tests

This is to upsert and soft delete 2864 product documents. For upserting, we will test two scenarios: 1. loop through
the prodcuts and upsert them one by one. To upsert N documents, we need to call MongoDB N times. 2 upsert all 
products in one call. For soft deleting, in addition to upserting documents one by one and bulk upserting, we will
also update those documents by setting the `isDeleted` to true ( and updating other fields like timestamp *etc*. if 
necessary ). Before we took advantage of MongoDB bulk operations, in the old code base we essentially used the 
following logic to soft-delete documents:

```
void softDeleteProducts(List<Integer> productIds) {
    productIds.forEach(id -> {
        Product productFromDB = productRepository.findProductById(id);
            if (productFromDB != null) {
                productFromDB.setDeleted(true);
                productRepository.save(productFromDB);
            }
    });
}
```

in this case, to soft-delete N documents we need to call MongoDB 2 * N times. The following table shows the test 
results for upserting and soft deleting the same 2864 products:

Processing Time (in seconds)

|               | 1 pod w/o bulk operations | 30 pods w/o bulk operations | 1 pod w/ bulk operations |
|---------------|:-------------------------:|:---------------------------:|:------------------------:|
| Upsert        |           57.67           |            6.54             |           4.08           |
| Soft delete   |           74.6            |            8.38             |           0.52           |


Notes:

- The Java application was running in our AWS DEV environment.
- Each item in the above table is the average value of 3 tests.
- For upserting with bulk operations, currently in order to get the 2864 documents we must call another 
 service 14 times to avoid the request containing more than 2048 characters. On an average, the 14 calls take 
 about 1.25 seconds. If we could get all 2864 documents with a single call, the improvement would be even better.

#### Discussion:
From the "Soft delete" row in the above table we can see updating is even faster than upserting. Although in 
both situations the Java code only calls MongoDB once respectively, within MongoDB the cluster needs to 
execute 2864 replaceOne commands to finish the upsert, but only one call to finish the updating. In addition,
the size of the upserting request is much bigger than that of the updating request. On the average, each `product`
document is takes about 7K bytes.


### Local tests

Similar to the above "Benchmark tests", this part also only focuses on upserting and updating multiple documents. 
However, there is something special as follows:

* We only test soft deletion.
* The Java testing code runs on a local PC.
* Performance measurement is much more accurate.

Processing Time (in milliseconds)

|                                 | Update one by one | Bulk upsert | Soft delete all |
|---------------------------------|:-----------------:|:-----------:|:---------------:|
| Remote M20 (1,000 documents)    |      39,295       |    1,432    |       165       |
| Local MongoDB (1,000 documents) |       1,087       |    648      |       65        |

Note: For each test, the result in the table is the average of 3 values.

#### Discussion:
1. From the above table and the previous one, we can see that bulk operations reduce network overhead 
 significantly. 

2. For a Java application running on AWS, because the MongoDB is also on AWS, the network latency is smaller than
 a local Java – remote MongoDB scenario. As a result, the improvement falling between the local and the remote 
 MongoDB clusters makes perfect sense.


### Batch process in production

Time used for upserting 5 million plus documents

| w/o bulk operations | w/ bulk upsert |
|---------------------|:---------------|
| > 300 minutes       | ~ 20 minutes   |

### stream() call

In addition to upserting documents, from time to time we also need to read back all the 5 million plus documents and 
publish them to Kafka. With a MongoOperations stream() call, the whole process takes about 40 minutes including 
publishing all the records. Obviously, the network overhead caused by this stream() call is very small.

## Conclusion

Using MongoDB bulk operations can significantly improve an application's performance. Avoiding returning not used 
fields can also improve the performance. In addition, using bulk operations also requires fewer MongoDB 
connections and reduces collection locking.
