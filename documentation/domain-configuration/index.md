---
layout: page
category: Documentation
title: Domain Configuration
submenu: domain-configuration
sidebar-url: docs-sidebar.html
---

None of us like to configure tools but don’t worry — JaVers knows it and
does the hard work to minimize the configuration efforts on your side.

As we stated before, JaVers configuration is very concise.
You can start with zero config and give JaVers a chance to infer all the facts about your domain model.

Take a look how JaVers deals with your data. If it’s fine, let the defaults work for you.
Add more configuration when you want to change the default behavior.

There are two logical areas of the the configuration,
[domain model mapping](/documentation/domain-configuration#domain-model-mapping)
and
[repository setup](/documentation/repository-configuration).
Proper mapping is important for both JaVers features, the object diff and the data audit (JaversRepository).

The object diff algorithm is the core of JaVers. When two objects are compared, JaVers needs to know what
type they are.
We distinguish between the following types: Entities, Value Objects, Values, Containers, and Primitives.
Each type has a different comparing style.

JaVers can infer the type of your classes, but if it goes wrong, the diff result might be strange.
In this case you should tune the type mapping.

For now, we support both the Java config via [JaversBuilder]({{ site.javadoc_url }}index.html?org/javers/core/JaversBuilder.html)
and the annotations config.

<h2 id="domain-model-mapping">Domain model mapping</h2>

**Why domain model mapping is important?**<br/>
Many frameworks which deal with user domain model (aka data model) use some kind of <b>mapping</b>.
For example JPA uses annotations in order to map user classes into a relational database.
Plenty of XML and JSON serializers use various approaches to mapping, usually based on annotations.

When combined together, all of those framework-specific annotations could be a pain and a
pollution in your business domain code.

Mapping is also a case in JaVers but don’t worry:

* JaVers wants to know only a few basic facts about your domain model classes.
* Mapping is done mainly on the class level — property level mapping is required only for
  choosing Entity ID.
* JaVers scans well-known annotations sets like JPA and Hibernate (see [supported annotations table](#supported-annotations)).
  So if your classes are already annotated with these sets, you are lucky.
  If not, JaVers provides its [own annotations set]({{ site.javadoc_url }}org/javers/core/metamodel/annotation/package-summary.html).
* If you’d rather keep your domain model classes framework agnostic,
  use [`JaversBuilder`]({{ site.javadoc_url }}index.html?org/javers/core/JaversBuilder.html).
* JaVers uses reasonable defaults and takes advantage of a *type inferring algorithm*.
  So for a quick start just let JaVers do the mapping for you.
  Later on, it would be advisable to refine the mapping in order to optimize the diff semantic.

Proper mapping is essential for the diff algorithm, for example JaVers needs to know if a given object
should be compared property-by-property or using equals().

### JaVers Types
The JaVers type system is based on *Entity* and *Value Objects* notions, following Eric Evans
Domain Driven Design terminology (DDD).
Furthermore, JaVers uses *Value*, *Primitive*, and *Container* notions.
The last two types are JaVers internals and can’t be mapped by a user.

To make long story short, JaVers needs to know
the [`JaversType`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/type/JaversType.html)
for each of your classes spotted in runtime (see [mapping configuration](#mapping-configuration)).

Let’s examine the fundamental types more closely.

<h3 id="entity">Entity</h3>
JaVers Entity has exactly the same semantic as DDD Entity or JPA Entity.
It’s a big domain object (for example Person, Company, Car).

Usually, each Entity instance represents a concrete physical object
which can be identified.
An Entity has a list of mutable properties and its own identity held in *ID property*.

For example:

```groovy
@TypeName("Person")
class Person {
    @Id String name
    String position
}

def "should map Person as EntityType"(){
  given:
  def bob = new Person(name: "Bob", position: "dev")
  def javers = JaversBuilder.javers().build()

  def personType = javers.getTypeMapping(Person)
  def bobId = personType.createIdFromInstance(bob)

  expect:
  bobId.value() == "Person/Bob"
  bobId instanceof InstanceId
  
  println "JaversType of Person: " + personType.prettyPrint()
  println "ID of bob: '${bobId.value()}'"
}
```

output:

```text
JaversType of Person: EntityType{
  baseType: class org.javers.core.examples.CustomToStringExample$Person
  typeName: Person
  managedProperties:
    Field String name; //declared in Person
    Field String position; //declared in Person
  idProperty: name
}
ID of bob: 'Person/Bob'
```

**Comparing strategy** for Entity state is property-by-property. 
For Entity references, comparing is based on InstanceId.

Entity can contain Value Objects, Entity references, Containers, Values and Primitives.

<h3 id="entity-id">Entity ID</h3>

Entity ID has a special role in JaVers. It identifies an Entity instance.
You need to choose **exactly one** property as ID for each of your Entity classes
(we call it ID property).
 
For each Entity instance, JaVers creates a global identifier called
[InstanceId]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/object/InstanceId.html).
It’s a String composed of an Entity type name and an ID value.

Since InstanceId value is a String, each object used as Entity ID
should have a good String representation.

There are three ways of creating a **String representation** for an Entity ID:

* For Primitives (and we recommend using Primitives here), JaVers simply calls `Object.toString()`. 

* For Objects, JaVers uses the [`reflectiveToString()`]({{ site.javadoc_url }}org/javers/common/reflection/ReflectionUtil.html#reflectiveToString-java.lang.Object-)
function by default, which concatenates `toString()` called on
all properties of a given Object ID. 

* For Objects, you can also register a custom function to be used instead of `reflectiveToString()`.

For example:

```groovy
@TypeName("Entity")
class Entity {
    @Id Point id
    String data
}

class Point {
    double x
    double y

    String myToString() {
        "("+ (int)x +"," +(int)y + ")"
    }
}
    
def "should use custom toString function for complex Id"(){
  given:
  Entity entity = new Entity(id: new Point(x: 1/3, y: 4/3))

  when: "default reflectiveToString function"
  def javers = JaversBuilder.javers()
          .build()
  GlobalId id = javers.getTypeMapping(Entity).createIdFromInstance(entity)

  then:
  id.value() == "Entity/0.3333333333,1.3333333333"

  when: "custom toString function"
  javers = JaversBuilder.javers()
          .registerValueWithCustomToString(Point, {it.myToString()})
          .build()
  id = javers.getTypeMapping(Entity).createIdFromInstance(entity)

  then:
  id.value() == "Entity/(0,1)"
}
```   

**ProTip:** The JaversType of an ID property type is mapped automatically
to Value if it’s an Object or to Primitive if it’s a Java primitive.
So &mdash; technically &mdash; you can’t have a Value Object type here. 

**ProTip:** JaVers doesn’t distinguish between JPA `@Id` and `@EmbeddedId` annotations,
so you can use them interchangeably. 

<h3 id="value-object">Value Object</h3>
JaVers Value Object is similar to DDD Value Object and JPA Embeddable.
It’s a complex value holder with a list of mutable properties but without a unique identifier.

Value Object instances has a 'best effort' global identifier called
[ValueObjectId]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/object/ValueObjectId.html).
It consists of the owning Entity InstanceId and
the path in an object subtree.

**Comparing strategy** for Value Objects is property-by-property.

**ProTip:** In a strict DDD approach, Value Object can’t exist independently and has to be bound to an Entity instance
(as a part of an Aggregate). JaVers is not so radical and supports both embedded and dangling Value Objects.
So in JaVers, Value Object is just an Entity without identity.

**Examples** of Value Objects: Address, Point.

<h3 id="ValueType">Value</h3>
JaVers [Value]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/type/ValueType.html)
is a simple (scalar) value holder.

**Comparing strategy** for Values is (by default) based on the **`Object.equals()`**.
So it’s highly important to implement this method properly,
it should compare the underlying state of given objects.

Well-known Value types like BigDecimal or LocalDate have it already.
Remember to implement `equals()` for all your Value classes.

If you don’t control the Value implementation, 
you can still change the comparing strategy by registering a [CustomValueComparator]({{ site.javadoc_url }}index.html?org/javers/core/diff/custom/CustomValueComparator.html)
function.
For example, if you want to compare BigDecimals using only the integer part:
 
```java
Javers javers = JaversBuilder.javers()
        .registerValue(BigDecimal.class, (a, b) -> a.intValue() == b.intValue()).build();

``` 

**Examples** of Values: BigDecimal, LocalDate.

For Values, it’s advisable to customize JSON serialization by implementing *Type Adapters*
(see [custom JSON serialization](/documentation/repository-configuration/#custom-json-serialization)).

<h2 id="mapping-configuration">Mapping configuration</h2>
Your task is to identify *Entities*, *Value Objects* and *Values* in your domain model
and make sure that JaVers has got it. So what should you do?

There are three ways to map a class:

1. Explicitly, with the JaversBuilder methods :
    * [`registerEntity(Class<?>)`]({{ site.javadoc_url }}org/javers/core/JaversBuilder.html#registerEntity-java.lang.Class-),
      `@ID` annotation points to *Id-property*
    * [`registerEntity(Class<?> entityClass, String idPropertyName)`]({{ site.javadoc_url }}org/javers/core/JaversBuilder.html#registerEntity-java.lang.Class-java.lang.String-), Id-property given by name
    * [`registerValueObject(Class<?>)`]({{ site.javadoc_url }}org/javers/core/JaversBuilder.html#registerValueObject-java.lang.Class-)
    * [`registerValue(Class<?>)`]({{ site.javadoc_url }}org/javers/core/JaversBuilder.html#registerValue-java.lang.Class-)
1. Explicitly, with JaVers or JPA annotations, 
   see [class level annotations](#class-level-annotations)
1. Implicitly, using the *type inferring algorithm* based on the class inheritance hierarchy.
   JaVers **propagates** the class mapping down to the inheritance hierarchy.
   If a class is not mapped (by method 1 or 2),
   JaVers maps this class the in same way as its nearest supertype (superclass or interface)

**Priorities** <br/>
JaversBuilder `register...()` methods have a higher priority 
then annotations &mdash; JaVers ignores type annotations when a class is already
registered in JaversBuilder.
Type inferring algorithm has the lowest priority.

**Mapping ProTips**

* First, try to map high level abstract classes.
  For example, if all of your Entities extend some abstract class,
  you can map only this class with `@Entity`.
* JaVers automatically scans JPA annotations
  and maps classes with `@Entity` annotation as Entities
  and classes with `@Embeddable` as Value Objects. So if you are using frameworks like Hibernate,
  your mapping is probably almost done.
* Use [`@TypeName`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/TypeName.html)
  annotation for Entities, it gives you freedom of class names refactoring. 
* In some cases, annotations could be misleading for JaVers.
  For example [Morphia](https://github.com/mongodb/morphia) framework uses `@Entity` annotation for each persistent class
  (even for Value Objects). This could cause incorrect JaVers mapping.
  As a solution, use explicit mapping with the JaversBuilder methods,
  as it has the highest priority.
* For an Entity, a type of its Id-property is mapped as Value by default.
* If JaVers knows nothing about a class, it maps that class as Value Object **by default**.
* JaVers compares objects deeply. It can cause performance problems for large object graphs.
  Use `@ShallowReference` and `@DiffIgnore` to [ignoring things](#ignoring-things).
* If you are not sure how JaVers maps your class, check effective mapping using
  [`getTypeMapping(Class<?>)`]({{ site.javadoc_url }}org/javers/core/Javers.html#getTypeMapping-java.lang.Class-) method.
  Once you have JaversType for your class, you can pretty-print it:
  `println( javers.getTypeMapping(YourClass.class).prettyPrint() )`   

<h2 id="supported-annotations">Supported annotations</h2>

JaVers supports two sets of annotations: JaVers native (recommended) and JPA to some extent.

**ProTip:** JaVers ignores package names and cares only about simple class names.
So you can use any annotation as long as its name matches one of the JPA or JaVers names.

<h3 id="class-level-annotations">Class level annotations</h3>

There are six class level annotations in JaVers:

* [`@Entity`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/Entity.html)
  &mdash;
  declares a given class (and all its subclasses) as the [Entity](#entity) type.
   
* [`@ValueObject`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/ValueObject.html)
  &mdash;
  declares a given class (and all its subclasses) as the [Value Object](#value-object) type.
  
* [`@Value`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/Value.html)
  &mdash;
  declares a given class (and all its subclasses) as the [Value](#ValueType) type.
  
* [`@DiffIgnore`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/DiffIgnore.html)
  &mdash;
  declares a given class as totally ignored by JaVers. 
  All properties with ignored type are ignored.<br/>
  Use it for **limiting depth** of object graphs to compare. See [ignoring things](#ignoring-things). 

* [`@ShallowReference`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/ShallowReference.html)
  &mdash;
  declares a given class as the ShallowReference type.
  It’s a tricky variant of the Entity type with all properties except Id ignored.<br/>
  Use it as the less radical alternative to @DiffIgnore
  for **limiting depth** of object graphs to compare. See [ignoring things](#ignoring-things). 
  
* [`@IgnoreDeclaredProperties`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/IgnoreDeclaredProperties.html)
  &mdash;
  use it to mark all properties **declared** in a given class as ignored by JaVers. <br/>
  JaVers still tracks changes done on properties inherited from a superclass.
  See [ignoring things](#ignoring-things). 
  
* [`@TypeName`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/TypeName.html)
  &mdash;
  a convenient way to name Entities and Value Objects.
  We recommend using this annotation for all Entities and Value Objects.
  Otherwise, Javers uses fully-qualified class names 
  in GlobalIds, which hinders refactoring classes committed to JaversRepository.  

Three **JPA** Class level annotations are interpreted as synonyms of JaVers annotations:

* `@javax.persistence.Entity` and `@javax.persistence.MappedSuperclass`
  are synonyms to JaVers `@Entity`,
* `@javax.persistence.Embeddable` is the synonym of `@ValueObject`.

<h3 id="property-level-annotations">Property level annotations</h3>

There are three property level annotations:

* [`@Id`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/Id.html)
  &mdash; 
  declares the [Id-property](#entity-id-property) of an Entity. 
 
* [`@DiffIgnore`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/DiffIgnore.html)
  &mdash;
  declares a property as ignored by JaVers.

* [`@DiffInclude`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/DiffInclude.html)
  &mdash;
  declares a property as visible by JaVers. Other 
  properties in a given class are ignored (unless explicitly included).
  Including is opposite approach to Ignoring, like blacklisting vs whitelisting.
  You can use only one approach for a given class.
 
* [`@ShallowReference`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/ShallowReference.html)
    &mdash;
  declares a property as `ShallowReference`.
  Can be used only for Entity type properties.
  All properties of a target Entity instance, except Id, are ignored.
  
* [`@PropertyName`]({{ site.javadoc_url }}index.html?org/javers/core/metamodel/annotation/PropertyName.html)
    &mdash; 
   gives an arbitrary name to a property.
   When not found, Javers infers a property name from a field or getter name.
   `@PropertyName` could be useful when refactoring clases that are already
   committed to JaversRepository.

**ProTip**: when property level @Id is found in a class, JaVers maps it automatically
as Entity. So when you use @Id, class level @Entity is optional.

Two **JPA** property level annotations are interpreted as synonyms of JaVers annotations:

* `@javax.persistence.Id` is the synonym of JaVers `@Id`,
* `@javax.persistence.Transient` is the synonym of `@DiffIgnore`.

<h2 id="entity-mapping-example">Mapping example</h2>
 
Consider the following Entity mapping example:

```groovy
import org.bson.types.ObjectId
import org.mongodb.morphia.annotations.Id
import org.mongodb.morphia.annotations.Property
import org.mongodb.morphia.annotations.Entity

class MongoStoredEntity {
    @Id
    private ObjectId id

    @Property("description")
    private String description

    // ... 
}    
```

With zero config, JaVers maps:

- `MongoStoredEntity` class as Entity, since the `@Id` annotation is present,
- `ObjectId` class as Value, since it’s the type of the ID property and it’s not a Primitive.

**So far so good**. This mapping is OK for calculating diffs.
Nevertheless, if you plan to use JaversRepository,
consider providing custom JSON `TypeAdapters`
for each of yours Value types,
especially Value types used as Entity ID (see [JSON TypeAdapters](/documentation/repository-configuration/#custom-json-serialization)).

<h2 id="property-mapping-style">Property mapping style</h2>
There are two mapping styles in JaVers `FIELD` and `BEAN`.
FIELD style is the default one. We recommend not changing it, as it’s suitable in most cases.

BEAN style is useful for domain models compliant with *Java Bean* convention.

When using FIELD style, JaVers accesses object state directly from fields.
In this case, `@Id` annotation should be placed at the field level. For example:

```java
public class User {
    @Id
    private String login;
    private String name;
    //...
}
```

When using BEAN style, JaVers accesses object state by calling **getters**.
`@Id` annotation should be placed at the method level. For example:

```java
public class User {
    @Id
    public String getLogin(){
        //...
    }

    public String getName(){
        //...
    }
    //...
}
```

BEAN mapping style is selected in `JaversBuilder` as follows:

```java
Javers javers = JaversBuilder
               .javers()
               .withMappingStyle(MappingStyle.BEAN)
               .build();
```

In both styles, access modifiers are not important, it could be private ;)

<h2 id="ignoring-things">Ignoring things</h2>

The ideal domain model contains only business relevant data and no technical clutter.
Such a model is compact and neat. All domain objects and their properties are important and worth being persisted.

In the real world, domain objects often contain various kind of noisy properties you don’t want to audit,
such as dynamic proxies (like Hibernate lazy loading proxies), duplicated data, technical flags,
auto-generated data and so on.

It’s important to exclude these things from the JaVers mapping, simply to save the storage and CPU.
This can be done by marking them as ignored.
Ignored properties are omitted by both the JaVers diff algorithm and JaversRepository.

Sometimes ignoring certain properties can dramatically improve performance.
Imagine that you have a technical property updated every time an object is touched
by some external system, for example:

```java
public class User {
   ...
   // updated daily to currentdate, when object is exported to DWH
   private Timestamp lastSyncWithDWH; 
   ...
}
```

Whenever a User is committed to JaversRepository,
`lastSyncWithDWH` is likely to *cause* a new version of the User object, even if no important data are changed.
Each new version means a new User snapshot persisted to JaversRepository
and one more DB insert in your commit.

**The rule of thumb:**<br/>
check JaVers log messages with commit statistics, e.g.

```
23:49:01.155 [main] INFO  org.javers.core.Javers - Commit(id:1.0, snapshots:2, changes - NewObject:2)

```
If numbers looks suspicious, configure JaVers to ignore data not important in your business.

There are a few ways to do this:

**Use property-level** `@DiffIgnore` or `@ShallowReference`
to ignore non-important properties. Alternatively, use `@DiffInclude`
to mark all important properties. See [property annotations](#property-level-annotations).

**Use class-level** `@DiffIgnore`, `@ShallowReference` or `@IgnoreDeclaredProperties`
(see [class annotations](#class-level-annotations)).

* `@DiffIgnore` is strongest and means *I don’t care, just ignore all objects with this type.*

* `@ShallowReference` is moderate and means *Do shallow diff, bother me only when referenced Id is changed.*

* `@IgnoreDeclaredProperties` is the least radical and means *Ignore all properties <b>declared</b> in this class but take care about all <b>inherited</b> properties.*

Annotations are the recommended way for managing domain objects mapping,
but if you’re not willing to use them, map your classes in `JaversBuilder`.

For example:

```java
JaversBuilder.javers()
    .registerEntity(new EntityDefinition(User.class, "someId", Arrays.asList("lastSyncWithDWH")))
    .build();
```

<h2 id="hooks">Object Hooks</h2>

Hooks are a way to interact with your objects during JaVers processing.
Currently, JaVers provides the object-access hook:

<h3 id="hooks-on-access">Object-access Hook</h3>

This hook is called just before JaVers tries to access your domain object.
You can use it for example to initialize an object before processing.

To add object-access hook to JaVers instance use `withObjectAccessHook()` method.

```java
JaversBuilder.javers().withObjectAccessHook(new MyObjectAccessHook()).build()
```

JaVers comes with Hibernate Unproxy implementation of object-access hook,
see [hibernate integration](/documentation/spring-integration/#hibernate-unproxy-hook). 
