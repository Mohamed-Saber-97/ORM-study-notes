# Getting started with ORM

## Understanding Object Relational Mapping

* _Object persistence_ means individual objects can outlive the application process; they can be saved to a data store
  and be re-created at a later point in time.
* **_CAP theorem_** rule means that a distributed system cannot be _**consistent**_,
  **_available_**, and **_tolerant against partition failures_** all at the same time

> A system may guarantee that all nodes will see the same data at the same time and
> that data read and write requests are always answered. But when a part of the system fails due to a host, network, or
> data center problem, you must either give up
> strong consistency or 100% availability. In practice, this means you need a strategy
> that detects partition failures and restores either consistency or availability to a certain degree (for example, by
> making some part of the system temporarily unavailable
> so data synchronization can occur in the background). Often, the data, the user, or
> the operation will determine whether strong consistency is necessary.

* How Java and SQL defines different notions of sameness:
  * Instance identity (roughly equivalent to a memory location, checked with `a == b`)
  * Instance equality, as determined by the implementation of the `equals()` method (also called equality by value)
  * the identity of a database row is expressed as a comparison of primary key values, several non-identical
    instances in Java to simultaneously represent the same row of a database, such as in concurrently running
    application threads.
* use _surrogate keys_ whenever you can’t find a good _natural key_
* **_JPA_** specifies what must be done to persist objects. **_Hibernate_**, the most popular implementation of this
  specification, will make the persistence happen, **_Spring Data_** makes the implementation of
* JPA requires that collection-valued attributes be typed to an `interface` such as `java.util.Set` or `java.util.List` and not to an actual
  implementation such as `java.util.HashSet` (this is good practice anyway) 
* The persistence-capable class and any of its methods shouldn’t be final (this is a requirement of the JPA specification).
* Hibernate is not so strict, and it will allow you to declare `final` classes as entities or as entities with `final` methods that access persistent fields. However, this is not a good practice, as this will prevent Hibernate from using the proxy pattern for performance improvement
* follow the JPA requirements if you would like your application to remain portable between different JPA providers
* Hibernate and JPA require a constructor with no arguments for every persistent class, The constructor need not be public,
  but it has to be at least package-visible for Hibernate to use runtime-generated proxies
  for performance optimization
## Project Configuration

### Configuring JPA

```xml
<!--src/main/resources/META-INF/persistence.xml-->
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
    <persistence-unit name=" persistence_unit_name" transaction-type="RESOURCE_LOCAL">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
        <properties>
            <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/biddingschema"/>
            <property name="jakarta.persistence.jdbc.user" value="root"/>
            <property name="jakarta.persistence.jdbc.password" value="root"/>
            <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
            <property name="jakarta.persistence.schema-generation.database.action" value="drop-and-create"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```

### Writing a persistent class

```java

@Entity(name = "entity_name")
@Table(name = "table_name", catalog = "Schema_Name")
public class ClassName {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long Id;
    private String message;
    //...setter,getters and business code
}
```

1. we need an `EntityManagerFactory` to talk to the database,the factory is thread-safe, and all code in the
   application that accesses the database should share it
    * `EntityManagerFactory emf = Persistence.createEntityManagerFactory("persistence_unit_name");`
2. Begin a new session with the database by creating an `EntityManager`. This is the context for all persistence
   operations
    * `EntityManager em = emf.createEntityManager();`
3. Get access to the standard transaction API, and begin a transaction on this thread of execution and make sure to
    * `em.getTransaction().begin();`
4. Enlist the transient instance with the persistence context; we make it persistent.Hibernate now knows that we wish to
   store that data, but it doesn't necessarily call
   the database immediately
5. Every interaction with the database should occur within transaction boundaries,even if we’re only reading data, so we
   start a new transaction. Any potential failure
   appearing from now on will not affect the previously committed transaction
6. We can change the value of a property. Hibernate detects this automatically because the loaded Message is still
   attached to the persistence context it was loaded in
7. On commit, Hibernate checks the persistence context for _dirty state_, and it executes the SQL _**UPDATE**_
   automatically to synchronize in-memory objects with the
   database state
8. We created an EntityManager, so we must close it
9. We created an EntityManagerFactory, so we must close it

```java
public class Repository {
    private final EntityManagerFactory emf = Persistence.createEntityManagerFactory("persistence_unit_name");//1
    public void task()
    {
        try
        {
            EntityManager em = emf.createEntityManager();//2
            em.getTransaction().begin();//3
            ClassName className = new ClassName();
            em.persist(className);//4
            className.setMessage("Hello");//6
            em.getTransaction().commit();//7
            em.getTransaction().begin();
            //Class Name not database table name
            List<Message> messages = em.createQuery("from ClassName className", ClassName.class).getResultList();
            em.getTransaction().commit();
            em.close();//8
        }
        catch(Exception e)
        {
            em.getTransaction().rollback();
        }
        finally
        {
            emf.close();//9
        }
    }
}
```

### Native Hibernate configuration

```xml
<!--src/main/resources/hibernate.cfg.xml-->
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC "-//Hibernate/Hibernate Configuration DTD//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.url" value="jdbc:mysql://localhost:3306/biddingschema"/>
        <property name="hibernate.connection.username" value="root"/>
        <property name="hibernate.connection.password" value="root"/>
        <property name="hibernate.connection.pool_size">50</property>
        <property name="hibernate.connection.driver_class" value="com.mysql.cj.jdbc.Driver"/>
        <property name="hibernate.hbm2ddl.auto" value="create"/>
        <property name="show_sql" value="true"/>
        <property name="format_sql" value="true"/>
        <property name="use_sql_comments" value="true"/>
    </session-factory>
</hibernate-configuration>
```

```java
public class Repository {


    private static SessionFactory createSessionFactory() {
        Configuration configuration = new Configuration();
        configuration.configure().addAnnotatedClass(ClassName.class);
        ServiceRegistry serviceRegistry = new StandardServiceRegistryBuilder().applySettings(configuration.getProperties()).build();
        return configuration.buildSessionFactory(serviceRegistry);
    }
    public void task()
    {
        try (SessionFactory sessionFactory = createSessionFactory();
             Session session = sessionFactory.openSession())
        {
            session.beginTransaction();
            ClassName className = new ClassName();
            session.persist(message);
            session.getTransaction().commit();
            session.beginTransaction();
            CriteriaQuery<Message> criteriaQuery = session.getCriteriaBuilder().createQuery(Message.class);
            criteriaQuery.from(Message.class);
            List<Message> messages = session.createQuery(criteriaQuery).getResultList();
            session.getTransaction().commit();
        }
    }
}
```
###  Switching between JPA and Hibernate
* To obtain a `SessionFactory` from an `EntityManagerFactory`, you’ll have to unwrap the first one from the second one
```java
private static SessionFactory getSessionFactory(EntityManagerFactory entityManagerFactory) {
 return entityManagerFactory.unwrap(SessionFactory.class);
}
```
* creating an `EntityManagerFactory` from an initial Hibernate configuration
```java
private static EntityManagerFactory createEntityManagerFactory() {
    Configuration configuration = new Configuration();
    configuration.configure().addAnnotatedClass(Message.class);
    Map<String, String> properties = new HashMap<>();
    Enumeration<?> propertyNames = configuration.getProperties().propertyNames();
    while (propertyNames.hasMoreElements()) {
        String element = (String) propertyNames.nextElement();
        properties.put(element,
                configuration.getProperties().getProperty(element));
    }
    return Persistence.createEntityManagerFactory("persistence_unit_name", properties);
}
```
### Domain Model Example
```java
@Entity
public class User{
    private String firstName;
    private String lastName;
    public String getName() {
        return firstname + ' ' + lastname;
    }
    public void setName(String name) {
        StringTokenizer tokenizer = new StringTokenizer(name);
        firstname = tokenizer.nextToken();
        lastname = tokenizer.nextToken();
    }
}
@Entity
public class Item {
    @NotNull//SQL NOT NULL
    @Size(
            min = 2,
            max = 255,
            message = "Name is required, maximum 255 characters."
    )
    private String name;
    @Future
    private Date auctionEnd;
    private Set<Bid> bids = new HashSet<>();
    public Set<Bid> getBids() {
        //Or  use  Collections.unmodifiableCollection(bids)
        return Collections.unmodifiableSet(bids);// should not return a modifiable collection
    }
    public void addBid(Bid bid) {
        if (bid == null)
            throw new NullPointerException("Can't add null Bid");
        if (bid.getItem() != null)
            throw new IllegalStateException("Bid is already assigned to an Item");
        bids.add(bid);
        bid.setItem(this);
    }
}
@Entity
public class Bid {
    private Item item;
    public Item getItem() {
        return item;
    }
    public void setItem(Item item) {
        this.item = item;
    }
}
//package-info.java
@org.hibernate.annotations.NamedQueries({
        @org.hibernate.annotations.NamedQuery(
                name = "findItemsOrderByName",
                query = "select i from Item i order by i.name asc"
        )
        ,
        @org.hibernate.annotations.NamedQuery(
                name = "findItemBuyNowPriceGreaterThan",
                query = "select i from Item i where i.buyNowPrice > :price",
                timeout = 60, // Seconds!
                comment = "Custom SQL comment"
        )
})
```