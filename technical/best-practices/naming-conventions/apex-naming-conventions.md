# Apex Naming Conventions

{% hint style="info" %}
This section is still under development and needs to be put into more pragmatic terms.

Style guide files will be made available for supported IDE's.
{% endhint %}

### 1 - Introduction

This document serves as the Style Guide for Ideal Image in the Apex programming language.

Like other programming style guides, the issues covered span not only aesthetic issues of formatting, but other types of conventions or coding standards as well.

#### 1.1 - Terminology notes

In this document, unless otherwise clarified:

* The term _class_ is used inclusively to mean an "ordinary" class, enum class, interface, virtual or abstract class.
* The term _member_ (of a class) is used inclusively to mean a nested class, field, method, or constructor; that is, all top-level contents of a class except initializers and comments.
* The term _comment_ always refers to implementation comments. We do not use the phrase "documentation comments", instead using the common term "ApexDoc." Other "terminology notes" will appear occasionally throughout the document.

#### 1.2 - Guide notes

Example code in this document is non-normative. They may not illustrate the only stylish way to represent the code. Optional formatting choices made in examples should not be enforced as rules.

### 2 - Source file basics

#### 2.1 - File name

The source file name should not have prefixes or suffixes. See also section 5.2.1 for Apex Class naming.

#### 2.2 - File encoding: UTF-8

Source files are encoded in UTF-8.

#### 2.3 - Whitespace characters

Aside from the line terminator sequence, the ASCII horizontal space character (0x20) and ASCII horizontal tab character (0x9) are the only whitespace characters that appears anywhere in a source file. This implies that all other whitespace characters in string and character literals are escaped.

#### 2.4 - String class escape methods

The Apex String class defines several escape\* methods that can be used to include special characters in strings.

#### 2.5 - SOQL quoted string escape sequence

SOQL defines several escape sequences that are valid in queries so that you can include special characters in your queries.

### 3 - Source file structure

A source file consists of, in order:

Top level ApexDoc comments Class declaration No blank lines separate each section that is present.

#### 3.1 - Top level ApexDoc comments

Each top-level global or public class starts with an ApexDoc on the first line, containing a high level description of its purpose.

See section 7 for more about ApexDoc.

#### 3.2 - Class declaration

**3.2.1 - Ordering of class contents**

The order you choose for the members and initializers of your class can have a great effect on learnability. Different classes types may order their contents in different ways, but the default class order should be:

* Constants
* Variables
* Constructors
* Methods
* Sub-classes
* Exception classes

The contents of the ordering above should also be in alphabetical order and ordered by access modifiers;

* global
* public
* protected
* private

**3.2.2 - Overloads: never split**

When a class has multiple constructors, or multiple methods with the same name, these appear sequentially, with no other code in between (not even private members).

#### 3.3 - Methods

**3.3.1 - Method length**

It is recommended to keep methods as short a possible and doing only one thing. The length of a method, measured in lines of code, should never exceed the height of what is viewable in a IDE without scrolling. However, the recommended maximum length of a method is around 15 lines of code.

**3.3.2 - Method arguments**

Do not pass more than four parameters into a method. Try to avoid modifying the parameters inside the method, use result classes instead when there is a need to return multiple variables.

### 4 - Formatting

#### 4.1 - Braces

**4.1.1 - Braces are used where optional**

Braces are used with if, else, for, do and while statements, even when the body is empty or contains only a single statement. With the exception of a single statement of continue, throw, break, return, etc

Apex properties may be written like:

```
public integer MyReadOnlyProp { get; }
public double MyReadWriteProp { get; set; }
public string MyWriteOnlyProp { set; }
```

**4.1.2 Nonempty blocks: K & R style**

Braces follow the Kernighan and Ritchie style ("Egyptian brackets") for nonempty blocks and block-like constructs:

* Line break before the opening brace.
* Line break after the opening brace.
* Line break before the closing brace.
* Line break after the closing brace.

**Example:**

```
private void exampleMethod()
{
    if (getSomeValue())
    {
        callToSomeMethod();
    }
    else 
    {
        callToAnotherMethod();
    }
}
```

See section 4.5.2 for horizontal whitespace style.

**4.1.3 - Empty blocks: may be concise**

An empty block or block-like construct may be in K & R style (as described in Section 4.1.2). Alternatively, it may be closed immediately after it is opened, with no characters or line break in between ({}).

#### 4.2 - Block indentation: +1 tab of 4 spaces

Each time a new block or block-like construct is opened, the indent increases by one TAB of 4 spaces. When the block ends, the indent returns to the previous indent level. The indent level applies to both code and comments throughout the block.

Continuation indents are 2 TABs of total 8 spaces.

**Example:**

```
AccountsSelector.newInstance()
        .selectById(ids)
```

#### 4.3 - Column limit: 120

Apex code has a column limit of 120 characters. Except as noted below, any line that would exceed this limit must be line-wrapped, as explained in Section 4.4, Line-wrapping.

**Exceptions:**

* Lines where obeying the column limit is not possible (for example, a long URL in ApexDoc).
* Command lines in a comment that may be cut-and-pasted into a shell.

#### 4.4 - Line-wrapping

_Terminology Note:_ When code that might otherwise legally occupy a single line is divided into multiple lines, this activity is called line-wrapping.

There is no comprehensive, deterministic formula showing exactly how to line-wrap in every situation. Very often there are several valid ways to line-wrap the same piece of code.

> _Note:_ While the typical reason for line-wrapping is to avoid overflowing the column limit, even code that would in fact fit within the column limit may be line-wrapped at the author's discretion.

> _Tip:_ Extracting a method or local variable may solve the problem without the need to line-wrap.

#### 4.4.1 - Where to break

The prime directive of line-wrapping is: prefer to break at a higher syntactic level. Also:

* When a line is broken at a non-assignment operator the break comes before the symbol.
  * This also applies to the following "operator-like" symbol: the dot separator (.)
* When a line is broken at an assignment operator the break typically comes after the symbol, but either way is acceptable. This also applies to the "assignment-operator-like" colon in an enhanced for statement.
* A method or constructor name stays attached to the open parenthesis (() that follows it.
* A comma (,) stays attached to the token that precedes it.

> _Note:_ The primary goal for line wrapping is to have clear code, not necessarily code that fits in the smallest number of lines.

**4.4.1.1 - SOQL and SOSL statements**

Line breaks are optional but recommended with large queries. A line break comes before a reserved word.

List\<Account> accountList = \[SELECT Id, Name FROM Account];

```
List<Account> accountListWithNotes = 
[
    SELECT Id,
           Name,
           LastModifiedDate,
           (SELECT Title, Body FROM Notes
    WHERE LastModifiedDate = LAST_N_YEARS:5)
    FROM Account
    WHERE LastModifiedDate = LAST_N_MONTHS:6
    AND Phone != NULL
    ORDER BY Phone ASC
];
```

**4.4.1.2 - Lists, Sets and Objects**

When defining and populating an array, list, set or an object with multiple attributes, no trailing comma's are allowed and line breaks are recommended for larger objects.

```
// good
Account newAccount = new Account(
        Name = 'Acme',
        BillingCity = 'New York'
);

// bad
Account newAccount = new Account(
        Name = 'Acme'
        , BillingCity = 'New York'
        ,Industry = 'Pharma'
);

// good
List<String> Names = new List<String>
{
    'Ed',
    'Ann',
    'Jan',
    'Erik',
};

// bad
List<String> Names = new List<String>
{
    'Ed'
    , 'Ann'
    , 'Jan'
    , 'Erik'
};
```

#### 4.5 - Whitespace

**4.5.1 - Vertical Whitespace**

Two single blank line appears:

1. Between consecutive members or initializers of a class: fields, constructors, methods, nested classes, static initializers, and instance initializers.
   * Exception: A blank line between two consecutive fields (having no other code between them) is optional. Such blank lines are used as needed to create logical groupings of fields.
   * Exception: Blank lines between enum constants are covered in Section 4.5.1.
2. Between statements, as needed to organize the code into logical subsections.
3. Optionally before the first member or initializer, or after the last member or initializer of the class (neither encouraged nor discouraged).
4. As required by other sections of this document (such as Section 2, Source file structure).

Multiple consecutive blank lines are permitted, but never required (or encouraged).

**4.5.2 - Horizontal whitespace**

Beyond where required by the language or other style rules, and apart from literals, comments and ApexDoc, a single ASCII space also appears in the following places only.

1. Separating any reserved word, such as if, for or catch, from an open parenthesis (() that follows it on that line
2. Separating any reserved word, such as else or catch, from a closing curly brace (}) that precedes it on that line
3. On both sides of any binary or ternary operator. This also applies to the following "operator-like" symbol:

* the colon (:) in an enhanced for statement. But does not apply to:
* the dot separator (.), which is written like object.toString()
* the SOQL local variable reference, which is written like B = \[SELECT Id FROM Account WHERE Id = :A.Id];.

1. On both sides of the double slash (//) that begins an end-of-line comment. Here, multiple spaces are allowed, but not required.
2. Between the type and variable of a declaration: List list
3. Optional just inside both braces of an list initializer

* new List\<Integer> {5, 6} and new List\<Integer> { 5, 6 } are both valid

This rule is never interpreted as requiring or forbidding additional space at the start or end of a line; it addresses only interior space.

**4.5.3 - Horizontal alignment: never allowed**

> Terminology Note: Horizontal alignment is the practice of adding a variable number of additional spaces in your code with the goal of making certain tokens appear directly below certain other tokens on previous lines.

This practice is not allowed.

**Example:**

```
// good
private Integer x;
private String str;

// bad
private Integer x;
private String  str;
```

> _**Tip:**_ Alignment can aid readability, but it creates problems for future maintenance so we decided not to allow it. Consider a future change that needs to touch just one line. This change may leave the formerly-pleasing formatting mangled, and that is allowed. More often it prompts the coder (perhaps you) to adjust whitespace on nearby lines as well, possibly triggering a cascading series of reformattings. That one-line change now has a "blast radius." This can at worst result in pointless busywork, but at best it still corrupts version history information, slows down reviewers and exacerbates merge conflicts.

#### 4.5.4 - Trailing Spaces: never allowed

Sometimes in the course of editing files, you can end up with extra whitespace at the end of lines. These whitespace differences can be picked up by source control systems and flagged as diffs, causing frustration for developers. While this extra whitespace causes no functional issues, we require that trailing spaces be removed before check-in.

```
// good
private Integer x;
private String str;

// bad
private Integer x;∙
private String  str;∙∙
∙∙∙∙
```

#### 4.6 - Grouping parentheses: recommended

Optional grouping parentheses are omitted only when author and reviewer agree that there is no reasonable chance the code will be misinterpreted without them, nor would they have made the code easier to read. It is not reasonable to assume that every reader has the entire Apex operator precedence table memorized.

#### 4.7 - Specific constructs

**4.7.1 - Enums**

After each comma that follows an enum constant, a line break is optional. Additional blank lines (usually just one) are also allowed. This is one possibility:

```
private enum Answer {
    YES,

    NO,
    MAYBE
}
```

An enum with no documentation on its constants may optionally be formatted as if it were an list initializer (see [Section 4.7.3.1](https://github.com/wimvelzeboer/fflib-apex-extensions/blob/main/Apex-Styleguide.md#473---lists)) on list initializers).

private enum Suit { CLUBS, HEARTS, SPADES, DIAMONDS }

**4.7.2 - Variable declarations**

Local variables are not habitually declared at the start of their containing block or block-like construct. Instead, local variables are declared close to the point they are first used (within reason), to minimize their scope. Local variable declarations typically have initializers, or are initialized immediately after declaration.

**4.7.3 - Lists**

Any list initializer may optionally be formatted as if it were a "block-like construct." For example, the following are all valid:

```
// Only for short lists:
List<String> example = new List<String> { 'one', 'two', 'three' };

// For short and long lists
List<String> example = new List<String>
{
    'one',
    'two',
    'three'
};
```

**4.7.4 - Annotations**

Annotations applying to a class, method or constructor appear immediately after the documentation block, and on a line of its own. These line breaks do not constitute line-wrapping, so the indentation level is not increased.

```
@deprecated
@testVisible
private String getNameIfPresent() { ... }
```

Annotations with more than one argument should break up the arguments on multiple lines.

```
@InvocableMethod(
  label = 'Get Account Names'
  description = 'Returns the list of account names corresponding to the specified account IDs.'
)
public static List<String> getAccountNames(List<ID> ids) { ... }
```

#### 4.7.5 - Comments

This section addresses implementation comments. ApexDoc is addressed separately in [Section 7](https://github.com/wimvelzeboer/fflib-apex-extensions/blob/main/Apex-Styleguide.md#7---apexdoc), ApexDoc.

Any line break may be preceded by arbitrary whitespace followed by an implementation comment. Such a comment renders the line non-blank.

Block comments are indented at the same level as the surrounding code. They may be in /\* ... _/ style or // ... style. For multi-line /_ ... \*/ comments, subsequent lines must start with \* aligned with the \* on the previous line.

```
/*
* This is          // And so           /* Or you can
* okay.            // is this.          * even do this. */
*/
```

Comments are not enclosed in boxes drawn with asterisks or other characters.

> _**Tip:**_ When writing multi-line comments, use the /\* ... \*/ style if you want automatic code formatters to re-wrap the lines when necessary (paragraph-style). Most formatters don't re-wrap lines in // ... style comment blocks.

**4.7.6 - Modifiers**

Class and member modifiers, when present, appear in the order recommended by the Apex Language Specification:

private | protected | public | global virtual | abstract with sharing | without sharing

### 5 - Naming

#### 5.1 - Rules common to all identifiers

Identifiers use only ASCII letters and digits, and, in a small number of cases noted below, underscores. Thus each valid identifier name is matched by the regular expression \w+ .

The platform reserves use of two consecutive underscores in a name (double underscore). A double underscore cannot be used in a developer name.

In this style guide special prefixes or suffixes, like those seen in the examples name\_, mName, s\_name and kName, are not used.

#### 5.2 - Rules by identifier type

**5.2.1 - Class names**

Class names are written in UpperCamelCase.

Class names are typically nouns or noun phrases. For example, Character or ImmutableList. Interface names may also be nouns or noun phrases (for example, List), but may sometimes be adjectives or adjective phrases instead (for example, Readable).

Test classes are named starting with the name of the class they are testing, and ending with Test. For example, HashTest or HashIntegrationTest.

**5.2.2 - Method names**

Method names are written in lowerCamelCase.

Method names are typically verbs or verb phrases. For example, sendMessage or stop.

Underscores may appear in unit test method names to separate logical components of the name. One typical pattern is , for example constructor\_nullArgument\_expectArgumentNullException. There is no One Correct Way to name test methods.

**5.2.3 - Constant names**

Constant names use CONSTANT\_CASE: all uppercase letters, with words separated by underscores. But what is a constant, exactly?

Constants are static final fields whose contents are deeply immutable and whose methods have no detectable side effects. This includes primitives, Strings, immutable types, and immutable collections of immutable types. If any of the instance's observable state can change, it is not a constant. Merely intending to never mutate the object is not enough. Examples:

```
// Constants
public static final Integer NUMBER = 5;
public static final List<String> NAMES = new List<String> { 'Ed', 'Ann' };
public static final Map<String, Integer> AGES = 
    new Map<String, Integer> { 'Ed' => 35, 'Ann' => 32};
public enum SomeEnum { ENUM_CONSTANT };

// Not constants
private static String nonFinal = "non-final";
private final String nonStatic = "non-static";
private static final Set<String> mutableCollection = new Set<String>();
These names are typically nouns or noun phrases.
```

**5.2.4 - Non-constant field names**

Non-constant field names (static or otherwise) are written in lowerCamelCase.

These names are typically nouns or noun phrases. For example, computedValues or index.

**5.2.5 - Parameter names**

Parameter names are written in lowerCamelCase.

One-character parameter names should be avoided.

**5.2.6 - Local variable names**

Local variable names are written in lowerCamelCase.

Even when final and immutable, local variables are not considered to be constants, and should not be styled as constants.

**5.2.7 - Return variable names**

Variables used in the return within methods are typically named result or results.

**5.2.8 - Property names**

Property names are written in UpperCamelCase.

**5.2.9 - SOQL and SOSL reserved words**

All SOQL and SOSL reserved words are written in all uppercase letters.

**5.2.10 - Types**

All data types are written in UpperCamelCase, including Id, collections (Map, Set, List) or Objects (generic SObjects defined in Apex as Object).

Exception: enum should be written in all lowercase letters.

#### 5.3 - Camel case: defined

Sometimes there is more than one reasonable way to convert an English phrase into camel case, such as when acronyms or unusual constructs like "IPv6" or "iOS" are present. To improve predictability, this styleguide specifies the following (nearly) deterministic scheme.

Beginning with the prose form of the name:

1. Convert the phrase to plain ASCII and remove any apostrophes. For example, "Müller's algorithm" might become "Muellers algorithm".
2. Divide this result into words, splitting on spaces and any remaining punctuation (typically hyphens). Recommended: if any word already has a conventional camel-case appearance in common usage, split this into its constituent parts (e.g., "AdWords" becomes "ad words"). Note that a word such as "iOS" is not really in camel case per se; it defies any convention, so this recommendation does not apply.
3. Now lowercase everything (including acronyms), then uppercase only the first character of:

* ... each word, to yield upper camel case, or
* ... each word except the first, to yield lower camel case

1. Finally, join all the words into a single identifier.

> _**Note:**_ the casing of the original words is almost entirely disregarded. Examples:

| Prose Form              | Correct           | Incorrect         |
| ----------------------- | ----------------- | ----------------- |
| "XML HTTP request"      | XmlHttpRequest    | XMLHTTPRequest    |
| "new customer ID"       | newCustomerId     | newCustomerID     |
| "inner stopwatch"       | innerStopwatch    | innerStopWatch    |
| "supports IPv6 on iOS?" | supportsIpv6OnIos | supportsIPv6OnIOS |

> _**Note:**_ Some words are ambiguously hyphenated in the English language: for example "nonempty" and "non-empty" are both correct, so the method names checkNonempty and checkNonEmpty are likewise both correct.

### 6 - Programming Practices

#### 6.1 - Caught exceptions: not ignored

It is incorrect to do nothing in response to a caught exception. Aside from logging the exception, it should also be shown to the user in a user friendly manner. If there is a diagnostics framework in place, it should be handled by the framework.

### 7 - ApexDoc

#### 7.1 - Formatting

**7.1.1 - General form**

The basic formatting of ApexDoc blocks is as seen in this example:

```
/**
 * A description of the method's functionality would go here.
 *  
 * @param fieldName A description of what this parameter is used for.
 *
 * @return A description of what is returned.
 * @throws ArgumentNullException if fieldName is null.
 */
public Integer method(String fieldName) { ... }
```

**7.1.2 At-clauses**

Any of the standard "at-clauses" that are used appear in the order

1. @description, the "@description" can be ommitted and replaced by just the description itselff
2. @param
3. @return,
4. @throws,
5. @example,
6. @see.

When an at-clause doesn't fit on a single line, continuation lines are indented two (or more) spaces from the position of the @.

#### 7.2 The summary fragment

Each ApexDoc block begins with a brief summary fragment. This fragment is very important: it is the only part of the text that appears in certain contexts such as class and method indexes.

This is a fragment: a noun phrase or verb phrase, not a complete sentence. It does not begin with "A {@code Foo} is a...", or "This method returns...", nor does it form a complete imperative sentence like "Save the record.". However, the fragment is capitalized and punctuated as if it were a complete sentence.

#### 7.3 - Where ApexDoc is used

At the minimum, ApexDoc is present for every global, public class, and every global, public or protected member of such a class, with a few exceptions noted below.

**7.3.1 - Exception: self-explanatory methods**

ApexDoc is optional for "simple, obvious" methods like getFoo, in cases where there really and truly is nothing else worthwhile to say but "Returns the foo".

> _**Important:**_ it is not appropriate to cite this exception to justify omitting relevant information that a typical reader might need to know. For example, for a method named getCanonicalName, don't omit its documentation (with the rationale that it would say only /\*\* @description Returns the canonical name. \*/) if a typical reader may have no idea what the term "canonical name" means!

**7.3.2 - Exception: overrides**

ApexDoc is not always present on a method that overrides a supertype method.

**7.3.3 - Non-required ApexDoc**

Other classes and members have ApexDoc as needed or desired.

Whenever an implementation comment would be used to define the overall purpose or behavior of a class or member, that comment is written as ApexDoc instead (using /\*\*).

### 8 - Testing

#### 8.1 - Declaration

**8.1.1 - Test Classes**

Test classes are annotated with @isTest. This omits them from code coverage considerations at the time of deployment and packaging. All test classes are private.

```
@isTest
private class ExampleTest
{
    ...
}
```

**8.1.2 - Test Methods**

Each test is annotated with a simple @isTest on the line proceeding the method declaration. This keeps it consistent with the declaration of a test class.

The name of a test method is descriptive as to what is being tested, what conditions apply to the method under test, what the expected outcome is. It should always start with "itShould..."

```
@isTest
private static void itShouldReturnANewAccountRecordWhenAccountDoesNotExists()
{
    ...
}
```

or

```
static testMethod void itShouldReturnANewAccountRecordWhenAccountDoesNotExists()
{
    ...
}
```

**8.1.3 - SeeAllData**

The @isTest(SeeAllData=true) test setting should be avoided unless absolutely necessary. Part of writing safe, high quality tests is to ensure that your test data is unchanging.

This test setting allows your tests to use live data in your Salesforce org. Any user could change that data at any time and in turn, cause your test to fail.

**8.1.4- Starting and Stopping**

In a test method, the System.Test.startTest() and System.Test.stopTest() method calls are to be used to isolate the single operation under test from any test setup code, by resetting the limits.

```
@isTest
private static void itShouldReturnANewAccountRecordWhenAccountDoesNotExists()
{
    // GIVEN
    ExampleController testController = new ExampleController();

    // Inserts an SObject record to use in the execution of the test. Impacts governor limits.
    Id testRecordId = TestDataHelperClass.Instance.insertRecord().Id;

    // WHEN
    System.Test.startTest();
    SObject actualRecord = testController.getRecord(testRecordId);
    System.Test.stopTest();

    // THEN
    System.assertNotEquals(null, actualRecord, 'Did not expect to retrieve a null record.');
    System.assertEquals(testRecordId, actualRecord.Id, 'Expected to retrieve the test record.');
}
```

#### 8.2 - Mocking

When possible, utilize mocking functionality. Mocking cuts down on test execution time by decoupling your code from the Salesforce database when running tests which interact with SObject records.

One such way is via the fflib-apex-mocks project. This allows for convenient mocking.

**8.2.1 - Class Considerations**

When utilizing mocking, it is required by the platform to have access to a constructor which is public or global and contains zero arguments.

If you do not wish to expose a zero argument constructor in a given class, you can declare a @testVisible, protected constructor.

```
public class Example
{
    /**
     * @description A protected constructor solely for mocking purposes.
     */
    @testVisible
    protected Example() { }
}
```

**8.2.2 - Method Considerations**

When restricting code blocks for purposes of mocking, check if the mock instance is null prior to checking if System.Test.isRunningTest().

This lessens the opportunity for any such performance bottlenecks that could occur at runtime during the checking of System.Test.isRunningTest() as it is secondary to the mocking null check.

```
if (mockInstance != null && Test.isRunningTest())
{
    return mockInstance;
}
```
