# Using MongoDB Bulk Operations in Spring Data MongoDB
  
This article describes how we used MongoDB bulk operations in Spring Data MongoDB to improve the performance of our application significantly.
  
## Mixing MongoRepository and MongoTemplate

Like any other Spring Data framework, Spring Data MongoDB provides MongoRepository for CRUD operations. Although saveAll() allows us to do bulk insert in an ideal situation, this method will save the items one by one if the "_id" field of any item is not null. This causes significant performance downgrade. As a result, in our project we decided to implement bulk upsert using MongoTemplate and to implement other CRUD operations in our Repository interface.

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

The following code snippet shows how we implement some CRUD methods:

```
public interface ProductRepository extends ProductDao, MongoRepository<Product, Integer> {

    @Query(value = "{ 'isDeleted' : false }", fields = "{ '_id' : 1}")
    List<Product> findAllNotDeletedProducts();

    @Query("{ '_id' : {$in: ?0}}")                // filter part
    @Update("{$set: {'isDeleted': true}}")        // update part
    void unpublishProducts(List<Integer> productIds);

}
```

Here the 1st method is the same as the following native query:

```
db.product.find( {"isDeleted": false}, {"_id": 1})
```

please note that projection part, `{"_id": 1}`, may significantly improve performance because of 2 
factors: 1. If a compound index exists (`key: { isDeleted: 1, _id: 1}`), MongoDB will not scan any document and will simply return documents only containing "_id", because “_id” is in the index . 2. Network traffic may be much lower.

The 2nd method in the above code snippet is essentially similar to the following:

```
db.product.updateMany( {"_id": {$in: [1234, 2234]}}, {$set: {"isDeleted": true}} )
```

## Performance Improvement after Using Bulk Operations

Before we took advantage of MongoDB bulk operations, we essentially used the following logic to retire (soft delete) products:

```
void unpublishProducts(List<Integer> productIds) {
    productIds.forEach(id -> {
        Product productFromDB = productService.findProductById(id);
            if (productFromDB != null) {
                productFromDB.setDeleted(true);
                productService.save(productFromDB);
            }
    });
}
```

Obviously, the above code snippet is not efficient because of 2 resons: 1. for each Product record the code calls MongoDB twice; 2. the code loops through Product records, instead of bulk updating. Now we only need to call the 2nd method in the above ProductRepository interface. This is much faster and the code is also cleaner.

In addition to un-publishing products, we also need to publish products. Now publishing products can take advantage of our bulk upsert implementation. The following table shows the test results for publishing and unpublishing the same 2864 products:

Processing Time (in seconds)

|               | 1 pod w/o bulk operations | 30 pods w/o bulk operations | 1 pod w/ bulk operations |
|---------------|:-------------------------:|:---------------------------:|:------------------------:|
| Publish       |           59.07           |            1.98             |           3.8            |
| Unpublish     |            69             |            9.27             |           0.36           |



Notes:
- Each item in the above table is the average value of 3 tests.

- For unpublishing products, because all product records are already in our MongoDB, we don’t need to get the products themselves. Therefore, it is all about MongoDB and the room for improvement is bigger.

- For publishing products, we must make extra calls to get the product records themselves. Currently that REST call only allows GET, so in order to get all 2864 products, we must call the services multiple times to avoid the GET request being longer than 2048 characters. Based on our preliminary analysis, if we could get all 2864 products with one call, the improvement should be as good as 30 currently running pods or even better.

## Conclusion

Using MongoDB bulk operations can significantly improve an application's performance. In addition, avoiding returning not used fields can also improve the performance. 
