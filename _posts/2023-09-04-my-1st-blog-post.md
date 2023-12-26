# Using MongoDB Bulk Operations in Spring Data MongoDB

This article describes how we used MongoDB bulk operations in Spring Data MongoDB to improve the performance of our application significantly. As a typical example, when stopping selling 2864 products under one category, using bulk operations is about 140 times faster than the original code. If we need to stop selling more products, the improvement can be even greater. Our tests prove that using bulk operations is even more important than distributing the processing to multiple pods using Kafka.

## Mixing MongoRepository and MongoTemplate

Like any other Spring Data framework, Spring Data MongoDB provides MongoRepository for CRUD operations.
Although saveAll() allows us to do bulk insert in an ideal situation, this method will save the items one by one
if the primary key field ( annotated with @ID ) of any item is not null. This causes significant performance downgrade. As a result, in our project we decided to implement bulk upsert using MongoTemplate and to implement other CRUD operations in our Repository interface.

To take advantage of both MongoRepository and MongoTemplate, for our ***product*** MongoDB collection, the following interface shows the whole picture:

```
public interface ProductRepository extends ProductDao, MongoRepository<Product, Integer> {

    .
    .
    .

}
```

Here, the whole idea is to use MongoRepository as much as possible, when performance is not an issue. This makes our code cleaner. Meanwhile the above strategy also allows us to use MongoTemplate for bulk upsert.

### Implementing Bulk Upsert

```
@Repository
public class ProductDaoImpl implements ProductDao {
    private final MongoTemplate mongoTemplate;

    @Autowired
    public ProductDaoImpl(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    @Override
    @Retryable(retryFor = Exception.class, maxAttempts = 4, backoff = @Backoff(delay = 2000, maxDelay = 16000, multiplier = 2))
    public void bulkUpsert(Collection<Product> products) {
        if (products == null || products.isEmpty())
            return;

        BulkOperations bulkOperations = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, Product.class);
        products.forEach(product -> {
            Query query = new Query().addCriteria(Criteria.where(ID).is(product.getProductId()));
            bulkOperations.replaceOne(query, product, FindAndReplaceOptions.options().upsert());
        });

        bulkOperations.execute();
    }
}
```

The above bulkUpsert() implementation actually generates a native MongoDB bulkWrite() call (https://www.mongodb.com/docs/manual/reference/method/db.collection.bulkWrite/) and one or multiple replaceOne() calls will be included in the bulkWrite() call. Please note each replaceOne() call will perform an upsert operation.

The data model, Product.java, is pretty much as follows:

```
@Document
public class Product {
    @Id
    private Integer productId;
    private String partNumber;

    .
    .
    .
}
```

### Implementing other CRUD methods

The following code snippet shows how we implement some other CRUD methods:

```
public interface ProductRepository extends ProductDao, MongoRepository<Product, Integer> {

    @Query(value = "{ 'isDeleted' : false }", fields = "{ '_id' : 1}")
    List<Product> findAllNotDeletedProducts();

    @Query("{ '_id' : {$in: ?0}}")                      // filter part
    @Update("{$set: {'isDeleted': true}}")              // update part
    void unpublishProducts(List<Integer> productIds);   // unpublishing means we should stop selling the product(s)

}
```

Here the 1st method is the same as the following native query:

```
db.product.find( {"isDeleted": false}, {"_id": 1})
```

please note that projection part, `{"_id": 1}`, can significantly improve performance because of 2
factors: 1. If a compound index exists (`key: { isDeleted: 1, _id: 1}`), MongoDB will not scan any document and will simply return documents only containing "_id", because it is in the index. In other words, this is a covered query (https://www.mongodb.com/docs/manual/core/query-optimization/#covered-query). 2. Network traffic will be much lower.

The 2nd method in the above code snippet is essentially similar to the following:

```
db.product.updateMany( {"_id": {$in: [1234, 2234]}}, {$set: {"isDeleted": true}} )
```

## Performance Improvement after Using Bulk Operations

### Benchmark tests

Before we took advantage of MongoDB bulk operations, in the old code base we essentially used the following logic to soft-delete products:

```
void unpublishProducts(List<Integer> productIds) {      // unpublishing means we should stop selling the product(s)
    productIds.forEach(id -> {
        Product productFromDB = productService.findProductById(id);
            if (productFromDB != null) {
                productFromDB.setDeleted(true);
                productService.save(productFromDB);
            }
    });
}
```
At first glance, the above implementation seems fine. But careful analysis will reveal two problems: 1. for each Product record the Java code calls MongoDB twice, getting and saving the record, respectively; 2. the code loops through Product records, instead of bulk updating. On the other hand for our business requirement, when we need to stop selling a product we simply soft delete that product. As a result, calling the 2nd method in the above ProductRepository interface is the best choice.

For the old implementation, if we need to stop selling N products, we must call MongoDB 2 * N times. For our benchmark test, here N is 2864, but in reality N could easily become 50,000 or bigger. However, in the new implementation, no matter how many products we need to soft-delete we always call MongoDB once. In other words, the performance increase can easily exceed the 140x we mentioned in the 1st paragraph.

In addition to un-publishing (a business term meaning to stop selling) products, we also need to publish (another term meaning to prepare it for sale) new products. Now publishing products can take advantage of our bulk upsert implementation. The following table shows the test results for publishing and unpublishing the same 2864 products:

Processing Time (in seconds)

|               | 1 pod w/o bulk operations | 30 pods w/o bulk operations | 1 pod w/ bulk operations |
|---------------|:-------------------------:|:---------------------------:|:------------------------:|
| Publish       |           57.67           |            6.54             |           4.08           |
| Unpublish     |           74.6            |            8.38             |           0.52           |



Notes:
- Each item in the above table is the average value of 3 tests.

- For publishing products with bulk operations, currently we must call another service 14 times for the 2864 products to avoid the request containing more than 2048 characters. On the average, the 14 calls take about 1.25 seconds. If we could get all 2864 products with one call, the improvement would be even better.

### Batch process in production

In addition to processing real time events related to the product status, we also batch process the product data from a legacy system. Currently, we maintain more than 5.5 million products. Before we moved to MongoDB bulk operations, processing these 5.5 million products took more than 5 hours, but now it always takes less than 20 minutes, the fastest one only 17 minutes. Roughly it is 15 times faster and all products are included.

## Conclusion

Using MongoDB bulk operations can significantly improve an application's performance. In addition, avoiding returning not used fields can also improve the performance. 
