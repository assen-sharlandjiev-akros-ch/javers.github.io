---
layout: page
category: Documentation
title: Diff Configuration
submenu: diff-configuration
sidebar-url: docs-sidebar.html
---

JaVers’ diff algorithm has a pluggable construction.
It consists of the core comparators suite and optionally, [custom comparators](#custom-comparators).

You can fine-tune how the whole algorithm works by registering custom comparators
for certain types (custom comparators overrides core comparators).

For [comparing Lists](#list-algorithms), JaVers has three core comparators:
 **Simple** (default), **Levenshtein** distance, and **Set**. Pick one.

<h2 id="list-algorithms">List comparing algorithms</h2>
Generally, we recommend using **Levenshtein**, because it’s the smartest one.
But use it with caution, it could be slow for long lists,
say more then 300 elements.

The main advantage of **Simple** algorithm is speed, it has linear computation complexity.
The main disadvantage is the verbose output.

Choose the **Set** algorithm if you don’t care about the items ordering. 
JaVers will convert all Lists to Sets before comparision.
This algorithm produces the most concise output (only `ValueAdded` and `ValueRemoved`).   

You can switch to Levenshtein or Set in JaversBuilder:

```java
    Javers javers = JaversBuilder.javers()
        .withListCompareAlgorithm(ListCompareAlgorithm.LEVENSHTEIN_DISTANCE)
        .build();
```

or

```java
    Javers javers = JaversBuilder.javers()
        .withListCompareAlgorithm(ListCompareAlgorithm.AS_SET)
        .build();
```

<h3 id="simple-vs-levenshtein">Simple vs Levenshtein algorithm</h3>

Simple algorithm generates changes for shifted elements (in case when elements are inserted or removed in the middle of a list).
On the contrary, Levenshtein algorithm calculates short and clear change list even in case when elements are shifted.
It doesn’t care about index changes for shifted elements.

For example, when we remove one element from a list:

```java
javers.compare(['a','b','c','d','e'],
               ['a','c','d','e'])
```

the change list will be different, depending on chosen algorithm:

<table class="table" width="100%" style='word-wrap: break-word; font-family: monospace;'>
    <tr>
        <th>
        Output from Simple algorithm
        </th>
        <th>
            Output from Levenshtein algorithm
        </th>
    </tr>
    <tr>
        <td>
            (1). 'b'>>'c' <br />
            (2). 'c'>>'d' <br />
            (3). 'd'>>'e' <br />
            (4). removed:'e'
        </td>
        <td>
            (1). removed: 'b'
        </td>
    </tr>
</table>

But when both lists have the same size:

```java
javers.compare(['a','b','c','d'],
               ['a','g','e','i'])
```

the change list will the same:

<table class="table" width="100%" style='word-wrap: break-word; font-family: monospace;'>
    <tr>
        <th>
        Simple algorithm
        </th>
        <th>
            Levenshtein algorithm
        </th>
    </tr>
    <tr>
        <td>
            (1). 'b'>>'g' <br />
            (2). 'c'>>'e' <br />
            (3). 'd'>>'i' <br />
        </td>
        <td>
            (1). 'b'>>'g' <br />
            (2). 'c'>>'e' <br />
            (3). 'd'>>'i' <br />
        </td>
    </tr>
</table>

<h3 id="more-about-levenshtein">More about Levenshtein distance</h3>
The idea is based on the [Levenshtein edit distance](http://en.wikipedia.org/wiki/Levenshtein_distance)
algorithm, usually used for comparing Strings.
That is answering the question what changes should be done to go from one String to another?

Since a list of characters (i.e. String) is equal to a list of objects up to isomorphism
we can use the same algorithm for finding the Levenshtein edit distance for list of objects.

The algorithm is based on computing the shortest path in a DAG. It takes both `O(nm)` space
and time. Further work should improve it to take `O(n)` space and `O(nm)` time (n and m being
the length of both compared lists).

<h2 id="custom-comparators">Custom Comparators</h2>

There are cases where JaVers’ default diff algorithm isn’t appropriate.
A good example is custom collections, like Guava’s [Multimap](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Multimap.html),
which are not connected with Java Collections API.

Let’s focus on Guava’s Multimap.
JaVers doesn’t support it out of the box, because Multimap is not a subtype of `java.util.Map`.
Still, Multimap is quite popular and you could expect to
have your objects with Multimaps compared by JaVers.

JaVers is meant to be lightweight and can’t depend on the large Guava library.
Without a custom comparator, JaVers maps Multimap as ValueType and compares its internal fields property-by-property.
This isn’t very useful. What we would expect is MapType and a list of MapChanges as a diff result.

Custom comparators come to the rescue, as they give you full control over the JaVers diff algorithm.
You can register a custom comparator for any type (class or interface)
to bypass the JaVers type system and diff algorithm.

JaVers maps classes with custom comparators as `CustomTypes`, which pretty much means
*I don’t care what it is*.

**Implementation**<br/>

All you have to do is implement the
[CustomPropertyComparator]({{ site.javadoc_url }}index.html?org/javers/core/diff/custom/CustomPropertyComparator.html)
interface:

```java
/**
 * @param <T> custom type, e.g. Multimap
 * @param <C> concrete type of PropertyChange returned by a comparator
 */
public interface CustomPropertyComparator<T, C extends PropertyChange> {
    /**
     * @param left left (or old) property value
     * @param right right (or current) property value
     * @param affectedId Id of domain object being compared
     * @param property property being compared
     * @return should return null if compared objects have no differences
     */
    C compare(T left, T right, GlobalId affectedId, Property property);
}
```

and register it with
 [`JaversBuilder.registerCustomComparator()`]({{ site.javadoc_url }}org/javers/core/JaversBuilder.html#registerCustomComparator-org.javers.core.diff.custom.CustomPropertyComparator-java.lang.Class-).

Implementation should calculate a diff between two values of CustomType
and return a result as a concrete `Change` subclass, for example:

```java
public class GuavaCustomComparator implements CustomPropertyComparator<Multimap, MapChange> {
    public MapChange compare(Multimap left, Multimap right, GlobalId affectedId, Property property) {
        ... // omitted
    }
}
```

Register the custom comparator instance in JaversBuilder, for example:

```java
JaversBuilder.javers()
    .registerCustomComparator(new GuavaCustomComparator(), Multimap.class).build()
```

**Custom way to compare Value types**<br/>
The same rules apply if you want to change JaVers’ default diff algorithm
for existing Value type, for example `BigDecimal`.

For Values, JaVers simply uses `equals()`. If this isn’t appropriate for you,
override it with a Custom comparator.
For example, JaVers provides `CustomBigDecimalComparator`, which rounds BigDecimals before comparing them:

```java
/**
 * Compares BigDecimals with custom precision.
 * Before comparing, values are rounded (HALF_UP) to required scale.
 * <br/><br/>
 *
 * Usage example:
 * <pre>
 * JaversBuilder.javers()
 *     .registerCustomComparator(new CustomBigDecimalComparator(2), BigDecimal).build();
 * </pre>
 */
public class CustomBigDecimalComparator
    implements CustomPropertyComparator<BigDecimal, ValueChange>
{
    private int significantDecimalPlaces;

    public CustomBigDecimalComparator(int significantDecimalPlaces) {
        this.significantDecimalPlaces = significantDecimalPlaces;
    }

    @Override
    public ValueChange compare(BigDecimal left, BigDecimal right, GlobalId affectedId,
        Property property)
    {
        BigDecimal leftRounded = left.setScale(significantDecimalPlaces, ROUND_HALF_UP);
        BigDecimal rightRounded = right.setScale(significantDecimalPlaces, ROUND_HALF_UP);

        if (leftRounded.equals(rightRounded)){
            return null;
        }

        return new ValueChange(affectedId, property, left, right);
    }
}
```








