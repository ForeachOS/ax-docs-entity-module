[[entity-query]]
== EntityQuery infrastructure
{module-name} provides an abstraction layer for querying entities.
This abstraction is built around the concept of an `EntityQuery`.

This chapter gives some insight in the setup and how to customize if so wanted.

[[entity-query-executor]]
=== EntityQueryExecutor
The `EntityQueryExecutor` is responsible for executing an `EntityQuery` and returning the entities requested.
An `EntityConfiguration` can have a single `EntityQueryExecutor.class` attribute holding the executor instance.

Default implementations exist for `JpaSpecificationExecutor` and `QueryDslPredicateExecutor`.
This means that any entity configurations having a repository of this type will get an `EntityQueryExecutor` created automatically.

NOTE: Since the `EntityQueryExecutor` is backed by a specific repository implementation, supported functionality also depends on the actual backing repository.

The presence of the `EntityQueryExecutor` is a requirement for the default <<entity-views,entity views>>.

==== AssociatedEntityQueryExecutor
Like `EntityQueryExecutor` that is registered on the `EntityConfiguration`, every `EntityAssociation` can have an `AssociatedEntityQueryExecutor` registered.
The `AssociatedEntityQueryExecutor` allows executing queries in the context of a single parent object.

Like the `EntityQueryExecutor`, the `AssociatedEntityQueryExecutor` is usually added automatically.

The presence of the `AssociatedEntityQueryExecutor` is a requirement for the default association <<entity-views,entity views>>.

[[entity-query-language]]
=== EntityQuery Language (EQL)

Apart from building an `EntityQuery` in code, one can be specified using the EntityQuery Language (EQL).
EQL provides an SQL-like syntax for building queries.  Some examples could be:

* `name = 'john'`
* `name = 'john' and birthday >= '1980-01-01'`
* `(name in ('john', 'jane') and birthday = today()) or birthday is EMPTY`
* `children contains 'john' or children is EMPTY`

A single clause of a query consists of a _field_, followed by an _operator_, followed by one or more _values_ or _functions_.
Multiple _values_ for one field are grouped together inside parentheses.
Clauses can be combined using either _and_ or _or_.
Operator precedence can be enforced by using parentheses around a clause.

A _field_ of a query clause usually corresponds with a single property you want to query.

NOTE: Special characters in string literals should be escaped using the \ (backslash) character.
In case of like conditions, they should be double-escaped.

==== Supported operators

===== Equality operators
Usable on most property types.

|===

|*=*  | equals +

|*!=* | not equals +

|===

===== IN/NOT IN
Usable on most property types.
These are the only operators that take a group of values as argument.

|===

|*IN*  | equal to any of the argument values +

|*NOT IN* | not equal to any of the argument values +

|===

===== Comparing operators
Usable on numeric and date property types.

|===

|*>*  | greater than +

|*>=* | greater than or equal to +

|*<*  | less than +

|*<=* | less than or equal to +

|===

===== LIKE operators
Usable on string property types.
The argument specifies the pattern that property value should match.
The pattern can contain simple wildcard specified using *%*.

|===

|*LIKE*  | should match the pattern specified as argument +

|*NOT LIKE* | should not match the pattern specified as argument +

|*ILIKE*  | case insensitive match of the pattern specified as argument +

|*NOT ILIKE* | should not match the pattern specified as argument (case insensitive) +

|===

NOTE: Special characters in LIKE statements should be double-escaped using the \ (backslash) character.
This includes the literal % (percentage) character.

.Case insensitive property matching
When using *ILIKE* conditions, a case insensitive lookup will be performed in the datastore.
If you want to force a property to always have a case insensitive lookup, you can do so by configuring a `EntityQueryConditionTranslator` attribute on that property.

[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
configuration
 .withType( Group.class )
 .properties(
    props -> props.property( "name" ).attribute( EntityQueryConditionTranslator.class, EntityQueryConditionTranslator.ignoreCase() )
 )
----

This will convert any equality or like operator to the equivalent case insensitive lookup.

WARNING: Whenever possible, it is probably best to use collaction settings of your datastore to ensure case insentive querying of properties.
Configuring it on a datastore level will almost certainly give you much better performance.

===== CONTAINS/NOT CONTAINS
Usable on collection property types.

|===

|*CONTAINS*  | argument should be present in the collection +

|*NOT CONTAINS* | argument should not be present in the collection +

|===

===== IS EMPTY/IS NOT EMPTY
Usable on most property types, although the behaviour is slightly different for collection types.
These operators to not take any additional arguments.

|===

|*IS EMPTY*  | property should be unset (null) or have no members (in case of collection)  +

|*IS NOT EMPTY* | property should be set (not null) or have at least one member (in case of collection) +

|===

==== Default EQL functions

.Security related functions
|===

|*currentUser()*  | returns the name of the current authenticated principal

|===


.Date and time functions
|===

|*now()*  | returns current timestamp +

|*today()* | returns date of today +

|===

==== EntityQueryParser

The `EntityQueryParser` is responsible for converting an EQL statement into a valid `EntityQuery`.
Any entity configuration with an `EntityQueryExecutor` registered will have an `EntityQueryParser` created automatically.

The parser will validate the EQL statement and convert it to a strongly typed `EntityQuery`.
The default `EntityQueryParser` uses the entity related `EntityPropertyRegistry` to validate the query clauses.

=== EQL filtering on list view
Any entity configuration having an `EntityQueryParser` and `EntityQueryExecutor` can enable an EQL based filter on its list views.
This can be done through the `entityQueryFilter()` method on the `EntityListViewFactoryBuilder.

The EQL filter will add a simple textbox that allows you to enter the EQL statement you want to use as filter for the list view.
Extend the `EntityQueryFilterProcessor` if you want to customize the default implementation.

.Enabling the default EQL filter
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
entities.withType( Group.class )
        .listView( lvb -> lvb.entityQueryFilter( true )	);
----

[[eql-predicate-on-list-view]]
=== EQL predicate on list view
List views also support a base predicate to be configured as an EQL statement.
This base predicate will always be applied to the query being executed if it uses the `DefaultEntityFetchingViewProcessor` or the `EntityQueryFilterProcessor`.

.Ensure deleted (flag) items are never shown
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
entities.withType( Group.class )
        .listView( lvb -> lvb.entityQueryPredicate( "deleted = false" )	);
----

Like EQL based filtering, this requires the entity configuration to have a valid `EntityQueryExecutor` infrastructure.

=== Extending EQL
The `EntityQuery` infrastructure provides some hooks you can use to extend the EQL support with application specific methods.

==== Custom value conversion

When converting an EQL query all value arguments are first converted to an `EQType` representation before being converted into their respective Java type.
Actual type conversion is then done via the Spring `ConversionService`.
To create a custom conversion you can simply register a `Converter` that converts from the relevant `EQType` to the property type.

The following table shows how EQL arguments will be converted to their respective `EQType`:

[cols=2, options="header"]
|===
|Argument value
|EQType

|name
| `EQValue`: name

|'name'
| `EQString`: name

|(name, 'name')
| `EQGroup` +
 - `EQValue`: name +
 - `EQString`: name

|users(name, 'name')
| `EQFunction`: users +
   [arguments] +
   - `EQValue`: name +
   - `EQString`: name

|===

By default {module-name} registers id-based lookups for all its registered entities.
So supposing you have an entity `User` with id 1 and you want to query on a property *creator* of type `User`, the following query would work: `creator = 1`.

When building the `EntityQuery` the value 1 would be used as the id to find the `User` instance, and the latter would be used as the argument for the final query.
If we want to replace the custom behavior and allow the user to be specified by username instead, we could easily register a custom converter.

[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
public class EQValueToUserConverter implements Converter<EQValue, User>
{
    ...

    @Override
    public User convert( EQValue source ) {
        return userRepository.findByUsername( source.getValue() );
    }
}

...

converterRegistry.addConverter( new EQValueToUserConverter(...) );
----

This would allow us to execute the queries like `creator = john` or  `creator in (john, jane)`.
Any type-specific converter will take precedence over the defaults.

NOTE: The example above would only work if the username can never contain any whitespace.
If it can, then we would have to specify it as a String instead and write a converter for `EQString` instead of `EQValue`.


==== Adding custom functions

An EQL function is represented by a unique name and can optionally take a number of arguments for its execution.
Adding custom functions is as easy as simply defining a `@Component` that implements the `EntityQueryFunctionHandler` interface.
All components of this type will be detected and checked when executing an EQL query.

The handler will be called with the required contextual data for the return type requested.
If you want to use a function to compare a property that has a `Date` type, your function should return a `Date` instance as well.

A single handler can support multiple functions and requested return types.

.Simple EntityQuery function that always returns the String hello
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
/**
 * Simple EntityQuery function that always returns the String 'hello'.
 * Example eql: name = hello() or name in (hello(), 'goodbye')
 */
@Component
public class HelloFunction implements EntityQueryFunctionHandler
{
	@Override
	public boolean accepts( String functionName, TypeDescriptor desiredType ) {
		return "hello".equals( functionName );
	}

	@Override
	public Object apply( String functionName,
	                     EQType[] arguments,
	                     TypeDescriptor desiredType,
	                     EQTypeConverter argumentConverter ) {
		return "hello";
	}
}
----

==== Custom EQL translation
You can register an `EntityQueryConditionTranslator` attribute on any entity property.
If a translator instance is present, it will be called during the parsing phase of an EQL into an `EntityQuery`.

.Example registering a translator
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
configuration
 .withType( Group.class )
 .properties(
    props -> props.property( "name" ).attribute( EntityQueryConditionTranslator.class, EntityQueryConditionTranslator.ignoreCase() )
 )
----
