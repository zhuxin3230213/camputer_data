### 3.4. 定义查询方法

repository 代理有两种方法去查询。一种是根据方法名或者自定义查询，可用的选项取决于实际的商店。然而,根据相应的策略来决定实际SQL的创建，让我们看看选择项吧。

#### 3.4.1. 查询查找策略

以下策略可供查询库基础设施来解决。您可以配置策略名称空间通过 `query-lookup-strategy`属性的XML配置或通过`queryLookupStrategy`启用的属性`${store}`库注释的Java配置。一些策略可能不支持特定的数据存储。

* `create` 试图构建一个能找到查询的查询方法名称。 通常的做法是把给定的一组注明前缀的方法名和解析的方法。

* `USE_DECLARED_QUERY`试图找到一个声明查询并将抛出一个异常情况。查询可以定义注释上。

* `CREATE_IF_NOT_FOUND`\(默认\)结合`CREATE`和`USE_DECLARED_QUERY`。 看起来一个声明查询第一,如果没有声明查询发现,它创建一个定制的基于名称的查询方法。这是默认查找策略,因此如果你不使用任何显式配置。 它允许快速查询定义的方法名,还custom-tuning这些查询通过引入需要查询。


#### 3.4.2  创建查询

query builder机制内置为构建约束查询库的实体。 带前缀的机制`findXXBy`,`readAXXBy`,`queryXXBy`,`countXXBy`, `getXXBy`自动解析的其余部分。进一步引入子句可以包含表达式等`Distinct`设置不同的条件创建查询。 然而,第一个`By`作为分隔符来表示实际的标准的开始。 在一个非常基础的查询,可以定义条件`And`或者`Or`。

例 13. 根据方法名创建查询

```java
public interface PersonRepository extends Repository<User, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}
```

实际结果的解析方法取决于你的持久性存储创建查询。-然而,也有一些一般要注意的事情。

* 遍历表达式通常结合运算符连接。您可以把表达式`And`和`Or`,`Between`,`LessThan`\(不超过\) , `GreaterThan`,`Like`等运算符，这些操作对不同的数据库可能有所不同，具体参考各参考文档

* 方法解析支持设置`IgnoreCase`在属性上面（如，`findByLastnameIgnoreCase(…)`\),或者支持查询所有属性忽略大小写（如，`findByLastnameAndFirstnameAllIgnoreCase(…)`\), 忽略大小写支持所有的数据库，其它的查询参考相关文档

* 您可以应用静态排序通过附加一个`OrderBy`基准进行排序,引用属性和方向提供了一个排序\(`asc`或 `Desc`\)。 创建一个支持动态排序的查询方法,明白了特殊参数处理 。


#### 3.4.3. 属性表达式

属性表达式只能引用的直接财产管理的实体,如前面的示例所示。 在创建查询时你已经确保解析房地产管理的域类的一个属性。 然而,您还可以定义约束通过遍历嵌套属性。 假设一个`Person`有一个`Address`与一个`Zipcode`。 在这种情况下一个方法的名称

```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```

创建属性遍历`x.address.zipCode`。方法执行首先解释整个部分\(`AddressZipCode`\)作为财产和检查的域类属性的名称\(小写形式\)。 分割源在驼峰式大小写部分从右侧头部和尾巴,试图找到对应的属性,在我们的例子中,分割为`AddressZip`和`Code`。 分裂不匹配,该算法分割点移动到左\(`Address`, `Zipcode`\)然后继续,

在大多数情况下，这种算法有可能会出现错误，您可以使用来解决这种模糊性 \_ 在方法名来手动定义遍历点。所以我们的方法名称最终将像这样:

```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```

如果你的属性名称包含下划线\(如。 first\_name 中下划线\),建议使用驼峰的方式来避免。

#### 3.4.4 特殊参数处理

处理参数查询只需方法参数定义为已经在上面的例子中。 除了基础查询将会认识到某些特定类型`Pageable`和`Sort`应用动态查询分页和排序

例 14. 使用 `Pageable`, `Slice` 和 `Sort`来查询

```java
Page<User> findByLastname(String lastname, Pageable pageable);

Slice<User> findByLastname(String lastname, Pageable pageable);

List<User> findByLastname(String lastname, Sort sort);

List<User> findByLastname(String lastname, Pageable pageable);
```

第一个方法允许在你的查询方法的静态定义查询中通过一个org.springframework.data.domain.Pageable实例来动态的添加分页。分页类知道元素的总数和可用页数。它通过基础库来触发一个统计查询计算所有的总数。由于这个查询可能对store消耗巨大，可以使用Slice来替代。Slice仅仅知道是否有下一个Slice可用，这对查询大数据已经足够了。

排序选项和分页的处理方式一样。如果你需要排序，简单的添加一个org.springframework.data.domain.Sort参数到你的方法即可。也正如你所见，简单的返回一个列表也是可以的，在这种情况下，生产的分页实例所需的附加元数据将不会被创建\(这也意味着额外的计数查询可能需要但不一定被公布\)。

> 要找到在你的查询中有多少页，你需要触发一个额外的计数查询。按照默认来说这个查询可以从你实际触发查询中衍生出来


#### 3.4.5. 限制查询结果

查询方法的结果可以通过关键字first或者top来限制,它们可以交替使用。在top/firest后添加数字来表示返回最大的结果数。如果没有数字，则默认假定1作为结果大小。 

示例15 用```Top```和```First```查询限制结果大小 

``` java
     User findFirstByOrderByLastnameAsc();

     User findTopByOrderByAgeDesc();

     Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

     Slice<User> findTop3ByLastname(String lastname, Pageable pageable);

     List<User> findFirst10ByLastname(String lastname, Sort sort);

     List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

限制表达式也支持Distinct关键字。对于限制查询的结果集定义到一个实例中包装这个结果到一个Optional中也是被支持的。

如果分页或者切片被应用到一个限制查询分页(计算多少页可用)则它也能应用于限制结果。

> 要注意结合通过Sort参数动态排序的限制结果容许表达查询的方法为“K”最小的，以及“K”最大的元素。

#### 3.4.6. 流查询结果

查询方法能对以JAVA 8的Stream为返回的结果进行逐步处理。而不是简单地包装查询结果在被用来执行流的流数据存储特定的方法。

例16 以JAVA 8的Stream来进行查询的流处理结果

```java

 @Query("select u from User u")

 Stream<User> findAllByCustomQueryAndStream();

 Stream<User> readAllByFirstnameNotNull();

 @Query("select u from User u")

 Stream<User> streamAllPaged(Pageable pageable);

```

> 一个数据流可能包裹底层数据存储特定资源，因此在使用后必须关闭。 你也可以使用close()方法或者JAVA 7 try-with-resources区块手动关闭数据流。

例17 在try-with-resources块中操作一个Stream<T>

```java

 try(Stream<User stream = repository.findAllByCustomQueryAndStream()){

 stream.forEach(...);

 }

```

> 当前不是所有的Spring Data模块都支持Stream<T>作为返回类型


#### 3.4.7. 异步查询结果

```java
@Async
Future<User> findByFirstname(String firstname);               (1)  

@Async
CompletableFuture<User> findOneByFirstname(String firstname);  (2)

@Async
ListenableFuture<User> findOneByLastname(String lastname);     (3) 
```

- (1) 使用 ```java.util.concurrent.Future``` 作为返回类型

- (2) 使用  ```Java 8``` ```java.util.concurrent.CompletableFuture``` 作为返回类型

- (3) 使用  ```org.springframework.util.concurrent.ListenableFuture``` 作为返回类型















