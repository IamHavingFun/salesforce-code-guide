# Apex Code Style Guide

This document explains some code style guidelines to make Apex code more
readable and easier to understand.

## Prefer readable variable names

> Readable variable names make it easier to understand which data the variable
> refers to.

::: danger BAD

```apex
Contact c = [SELECT FirstName, LastName FROM Contact WHERE Id = :contactId];
String fn = c.FirstName;
```

:::

::: tip BETTER

```apex
Contact theContact = [
  SELECT FirstName, LastName FROM Contact WHERE Id = :contactId
];
String firstName = theContact.FirstName;
```

:::

Variables with just a few letters make it hard for other developers to figure
out what the code does, especially which data is involved.

It is recommended to use readable, semantic names for variables. This way it is
much easier to understand which data is processed by the code.

Be careful with variable names matching SObject names or other types. It can
lead to weird errors.

```apex
Id id = theContact.Id;
Id otherId = Id.valueOf('...');  // Error because type Id is shadowed by variable
```

## Prefer readable method names

> Readable method names make it easier to understand the purpose and side
> effects of the methods.

::: danger BAD

```apex
public class AccountProvider {
  public List<Account> get(Set<String> names) {
    return [SELECT Id, Name FROM Account WHERE Name IN :names];
  }
}
public class AccountProcessor {
  public void process(List<Account> accounts, Integer numberOfEmployees) {
    for (Account account : accounts) {
      account.NumberOfEmployees = numberOfEmployees;
    }
    update accounts;
  }
}
```

:::

::: tip BETTER

```apex
public class AccountProvider {
  public List<Account> getAccountsByName(Set<String> names) {
    return [SELECT Id, Name FROM Account WHERE Name IN :names];
  }
}
public class AccountProcessor {
  public void updateNumberOfEmployees(
    List<Account> accounts, Integer numberOfEmployees
  ) {
    for (Account account : accounts) {
      account.NumberOfEmployees = numberOfEmployees;
    }
    update accounts;
  }
}
```

:::

Very short method names cannot properly express the purpose or the intended side
effects. This makes it hard to understand the effects of higher-level code which
invokes such methods.

It is recommended to use readable method names which express what the
encapsulated logic does (not how it does that). This makes higher-level code
much more understandable which helps avoid errors caused by misunderstanding.

## Prefer literals for immutable or local values

> Literal values contribute to more readable code by avoiding interruptions.

::: danger BAD

```apex
public class PriceFormatter {
  private static final Double ZERO_PRICE = 0.0;
  private static final String LINE_BREAK = '\n';
  public String formatPrice(Double price) {
    if (price == ZERO_PRICE) {
      return 'The price is' + LINE_BREAK +
        'absolute zero.';
    }
    return 'The price is:' + LINE_BREAK +
      String.valueOf(price);
  }
}
```

:::

::: tip BETTER

```apex
public class PriceFormatter {
  public String formatPrice(Double price) {
    if (price == 0.0) {
      return 'The price is\n' +
        'absolute zero.';
    }
    return 'The price is:\n' + String.valueOf(price);
  }
}
```

:::

Some developers seem to follow dogmatically the rule that all constant values
used in the code must be extracted to constants. However, this comes at a price.
Extracting constants makes the code harder to read because when developers
stumble upon a constant they will very likely jump to its declaration to find
out what the real value is. Then they have to navigate back to the position in
the code where they came from to continue reading. This is especially painful
when constants are introduced for values which are very unlikely to change
without altering their meaning, e.g. who would ever change `ZERO_PRICE` to a
value other `0.0`?

Performance optimization is also not a convincing argument because Salesforce
compiles Apex code to Java byte code before execution. One part of the Java
language specification is that constants initialized with static values are
inlined, i.e. the occurrences are replaced with the static value. Even if there
was a slight difference of some nanoseconds it probably is not worth the price
you have to pay in terms of readability.

It is recommended to use literals in the code if the values can be regarded
immutable or scoped to the method. Constants should only be considered for
extracting values which are intended to be used across several classes. If the
value is likely to change at a later point of time consider extracting a Custom
Metadata Type to enable configuration.

## Prefer early returns over nested blocks

> Early returns make logic easier to understand by avoiding nested blocks.

::: danger BAD

```apex
public class AccountProvider {
  public Set<Id> getAccountIdsByName(Set<String> names) {
    Set<Id> accountIds = new Set<Id>();
    if (names != null && !names.isEmpty()) {
      List<Account> accounts = [SELECT Id FROM Account WHERE Name IN :names];
      if (!accounts.isEmpty()) {
        for (Account currentAccount : accounts) {
          accountIds.add(currentAccount.Id);
        }
      }
    }
    return accountIds;
  }
}
```

:::

::: tip BETTER

```apex
public class AccountProvider {
  public Set<Id> getAccountIdsByName(Set<String> names) {
    Set<Id> accountIds = new Set<Id>();
    if (names == null || names.isEmpty()) {
      return accountIds;
    }
    List<Account> accounts = [SELECT Id FROM Account WHERE Name IN :names];
    if (accounts.isEmpty()) {
      return accountIds;
    }
    for (Account currentAccount : accounts) {
      accountIds.add(currentAccount.Id);
    }
    return accountIds;
  }
}
```

:::

Complex logic with several conditions can quickly end up in multiple nested code
blocks. This makes it harder for developers to find out what the result of the
method execution is. The reason is that fulfilling (or not fulfilling) certain
conditions may immediately determine the result but this is not obvious from the
code until developers have wrapped their head around all the nested blocks.

It is recommended to write logic in such a way that conditions determining the
result immediately cause the method to return right away. This avoids nested
blocks and makes it easier to explorer the logic step by step.

Also try to avoid `else` blocks as much as possible using this technique.

## Prefer methods without unexpected side effects

> Methods without unexpected side effects make it easier to understand how the
> software works.

::: danger BAD

```apex
public class AccountProvider {
  public Account getDefaultAccount() {
    try {
      return [SELECT Id, Name FROM Account WHERE Name = 'Default' LIMIT 1];
    } catch (Exception e) {
      Account defaultAccount = new Account(Name = 'Default');
      insert defaultAccount;
      return defaultAccount;
    }
  }
}
```

Due to the prefix `get` in the method name it is unexpected that it performs a
DML operation to insert a new Account record.

:::

::: tip BETTER

```apex
public class AccountProvider {
  public Account getDefaultAccount() {
    return [SELECT Id, Name FROM Account WHERE Name = 'Default' LIMIT 1];
  }
  public Account createDefaultAccount() {
    Account defaultAccount = new Account(Name = 'Default');
    insert defaultAccount;
    return defaultAccount;
  }
  public Account ensureDefaultAccountExists() {
    try {
      return getDefaultAccount();
    } catch (Exception e) {
      return createDefaultAccount();
    }
  }
}
```

With the method name `ensureDefaultAccountExists` it is more obvious that a DML
operation may be carried out.

:::

Writing resilient code is definitely important. Unfortunately, some developers
tend to implement measures for resilience as hidden side effects in methods
where other developers do not expect them. This can make it hard to understand
why the software behaves in a certain way or even introduce errors because
developers were not aware of the side effects of a particular method.

It is recommended to keep methods clean of side effects which are not suggested
by the method name. It makes the code more robust and easier to understand.

## Prefer complete assertions in unit tests

> Complete assertions help with early bug detection and increase trust in unit
> tests.

::: danger BAD

```apex
public class ContactProvider {
  public Contact getContactById(Id contactId) {
    return [SELECT FirstName, LastName FROM Contact WHERE Id = :contactId];
  }
}
@IsTest
class ContactProviderTest {
  @IsTest
  static void providesContactForValidId() {
    Contact testContact = new Contact(FirstName = 'John', LastName = 'Doe');
    insert testContact;
    ContactProvider provider = new ContactProvider();
    Contact resultContact = provider.getContactById(testContact.Id);
    System.assertEquals('John', resultContact.FirstName);
  }
}
```

:::

::: tip BETTER

```apex
public class ContactProvider {
  public Contact getContactById(Id contactId) {
    return [SELECT FirstName, LastName FROM Contact WHERE Id = :contactId];
  }
}
@IsTest
class ContactProviderTest {
  @IsTest
  static void providesContactForValidId() {
    Contact testContact = new Contact(FirstName = 'John', LastName = 'Doe');
    insert testContact;
    ContactProvider provider = new ContactProvider();
    Contact resultContact = provider.getContactById(testContact.Id);
    System.assertNotEquals(null, resultContact);
    System.assertEquals('John', resultContact.FirstName);
    System.assertEquals('Doe', resultContact.LastName);
  }
}
```

:::

Especially under the pressure of project deadlines developers may feel like
saving some time by not writing extensive assertions in their unit tests.
However, this is a dangerous path because incomplete assertions undermine the
reliability of the unit tests. Tests may not fail when the logic does not behave
as expected. This can delay the detection of bugs and developers will lose their
trust in unit tests.

It is recommended to formulate assertions for all the expected behavior of the
logic under test. Even if it is a little bit more effort in the beginning it
will give you confidence later on because you know you can rely on the tests.

## Prefer comments which explain decisions

> Comments are useful to add information which is not present in the code
> itself.

::: danger BAD

```apex
public class AccountCleanupJob implements Database.Batchable<SObject> {
  public Database.QueryLocator start(Database.BatchableContext context) {
    // Query all Account records
    return Database.getQueryLocator('SELECT Id, Name FROM Account');
  }
  public void execute(Database.BatchableContext context, List<SObject> scope) {
  }
  public void finish(Database.BatchableContext context) {
  }
}
```

The comment does not convey any information that developers cannot extract from
the code anyway.

:::

::: tip BETTER

```apex
public class AccountCleanupJob implements Database.Batchable<SObject> {
  public Database.QueryLocator start(Database.BatchableContext context) {
    // Use QueryLocator because the job must process more than 50,000 records
    return Database.getQueryLocator('SELECT Id, Name FROM Account');
  }
  public void execute(Database.BatchableContext context, List<SObject> scope) {
  }
  public void finish(Database.BatchableContext context) {
  }
}
```

The comment adds useful information about the problem domain.

:::

Adding comments which only describe _what_ the code does are of little value.
The problem is that such comments are prone to become inconsistent when the
logic is refactored because developers may not take their time to adjust the
comments as well. In the end the comments are more confusing than helpful.

It is recommended to use comments in the code to add information _why_ the code
was implemented this way. Such comments can help developers a lot to understand
the thoughts and decisions of the developer who wrote the code.

## Prefer methods with few parameters

> Fewer parameters make it easier to capture the meaning of method arguments.

::: danger BAD

```apex
@IsTest
public class TestAccountFactory {
  public static Account createAccount(
    String name, Integer numberOfEmployees, String industry
  ) {
    return new Account(
      Name = name, NumberOfEmployees = numberOfEmployees, Industry = industry
    );
  }
}
@IsTest
class AccountProviderTest {
  @IsTest
  static void providesAccountsForGivenNames() {
    Account testAccount = TestAccountFactory.createAccount(
      'Example Company', 520, 'Automotive'
    );
    // more test code
  }
}
```

It is difficult to figure out what the arguments mean when the factory method is
invoked in the provider test.

:::

::: tip BETTER

```apex
@IsTest
public class TestAccountBuilder {
  private String name;
  private Integer numberOfEmployees = 0;
  private String industry = 'General';
  public TestAccountBuilder name(String name) {
    this.name = name;
    return this;
  }
  public TestAccountBuilder numberOfEmployees(Integer numberOfEmployees) {
    this.numberOfEmployees = numberOfEmployees;
    return this;
  }
  public TestAccountBuilder industry(String industry) {
    this.industry = industry;
    return this;
  }
  public Account build() {
    return new Account(
      Name = name, NumberOfEmployees = numberOfEmployees, Industry = industry
    );
  }
}
@IsTest
class AccountProviderTest {
  @IsTest
  static void providesAccountsForGivenNames() {
    Account testAccount = new TestAccountBuilder()
      .name('Example Company')
      .numberOfEmployees(520)
      .industry('Automotive')
      .build();
    // more test code
  }
}
```

Applying the builder pattern makes the provider test code more readable.

:::

Application logic which depends on many external values may end up in a method
with lots of parameters. The dependant code invoking the method becomes more
confusing the more arguments are passed because from the pure values in the
argument list it can be hard to figure out their meaning.

It is recommended to keep parameter lists as short as possible. Many parameters
can also be an indicator that the method does too many different things and
should be split into separate methods.

For logic depending on the composition of multiple external values consider
applying the _builder pattern_ to allow setting the values one by one.
