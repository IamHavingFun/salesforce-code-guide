# Apex Architecture Guide

This document explains some architectural guidelines to help with writing high
quality Apex code.

## Prefer specialized classes with focused responsibility

> Classes with focused responsibility are easier to understand and easier to
> maintain.

::: danger BAD

```apex
class GeneralUtils {
  List<Account> getAccountsByIds(Set<Id> accountIds) {}
  List<Contact> getContactsByIds(Set<Id> contactIds) {}
}
```

:::

::: tip BETTER

```apex
class AccountProvider {
  List<Account> getAccountsByIds(Set<Id> accountIds) {}
}
class ContactProvider {
  List<Contact> getContactsByIds(Set<Id> contactIds) {}
}
```

:::

Classes with too many different responsibilities typically emerge from the
evolution of a software project. Most developers care about splitting business
logic into reusable methods. However, a common mistake is to put these methods
into generic classes which end up being a dump for logic which is not related to
each other.

The problem with these "god classes" is that they can grow to hundreds and even
thousands of lines of code. This makes it hard for developers to understand the
scope of the classes which often leads to an indifferent attitude when it comes
to extension of the software: _"I didn't know where to put this new logic, so I
just dumped it there in our general class."_

Apart from being a magnet for arbitrary logic "god classes" can make it more
difficult to maintain the software. Almost no developer has a holistic overview
of all the logic contained in these classes. The classes are refactored or
cleaned up less often because nobody dares to touch them. This can lead to dead
code which just bloats the software with unnecessary complexity.

The recommendation is to create rather small classes with limited
responsibility. This is called the _single responsibility principle_. Each class
should be scoped to a very particular purpose which can often be composed from a
business/technical domain and a certain functionality within that domain. For
example, a class can serve as a provider for Account records, restricting its
scope to the Account domain and the functionality of querying the data. Be
careful with just plain domain-specific classes like a "product service". They
tend to become dumps for unrelated domain logic as well.

## Prefer decoupled logic in separate methods

> Separate methods enable better reusability and extensibility of business
> logic.

::: danger BAD

```apex
class AccountCleaner {
  void cleanupDuplicateAccounts(Integer minNumberOfEmployees) {
    List<Account> accounts;
    if (minNumberOfEmployees > 0) {
      accounts = [
        SELECT Name FROM Account
        WHERE NumberOfEmployees >= :minNumberOfEmployees
        ORDER BY CreatedDate DESC
      ];
    } else {
      accounts = [SELECT Name FROM Account ORDER BY CreatedDate DESC];
    }
    Set<String> visitedAccountNames = new Set<String>();
    List<Account> duplicateAccounts = new List<Account>();
    for (Account account : accounts) {
      if (visitedAccountNames.contains(account.Name)) {
        duplicateAccounts.add(account);
        continue;
      }
      visitedAccountNames.add(account.Name);
    }
    if (!duplicateAccounts.isEmpty()) {
      delete duplicateAccounts;
    }
  }
}
```

The issue is that the method encapsulates both the query of relevant Account
records and the logic for detecting and removing duplicates. If more query
criteria should be introduced in the future it can be tempting to just add more
method parameters to differentiate how the records are queried.

:::

::: tip BETTER

```apex
class AccountCleaner {
  List<Account> getAllAccountsLatestFirst() {
    return [SELECT Name FROM Account ORDER BY CreatedDate DESC];
  }
  List<Account> getAccountsWithMinEmployeesLatestFirst(
    Integer minNumberOfEmployees
  ) {
    return [
      SELECT Name FROM Account
      WHERE NumberOfEmployees >= :minNumberOfEmployees
      ORDER BY CreatedDate DESC
    ];
  }
  void deleteDuplicateAccounts(List<Account> accounts) {
    Set<String> visitedAccountNames = new Set<String>();
    List<Account> duplicateAccounts = new List<Account>();
    for (Account account : accounts) {
      if (visitedAccountNames.contains(account.Name)) {
        duplicateAccounts.add(account);
        continue;
      }
      visitedAccountNames.add(account.Name);
    }
    if (!duplicateAccounts.isEmpty()) {
      delete duplicateAccounts;
    }
  }
  void cleanupOlderDuplicateAccountsOfLargeCompanies() {
    List<Account> largeCompanyAccounts =
      getAccountsWithMinEmployeesLatestFirst(500);
    deleteDuplicateAccounts(largeCompanyAccounts);
  }
}
```

The method for deleting duplicate Account records can be reused without
modification while it is easy to introduce more types of queries for retrieving
the list of records.

The method `cleanupOlderDuplicateAccountsOfLargeCompanies` is a higher-level
method which composes lower-level methods.

:::

Most developers tend to create one large method with all necessary logic as a
first step because this is the easiest way to get things done quickly. However,
this often means that the method encapsulates different subtasks and suffers
from a high complexity. Due to the different levels of abstraction put together
in one method it is harder for developers to get an overview of what the
fundamental procedure of the method is.

Another drawback of large methods with too many integrated tasks is that they
are hard to extend because everything is hard-wired and does not allow for
external modification. A very naive approach is to introduce method parameters
to enable the outer logic to control the behavior of the method. But this leads
to more complexity inside the large method and makes it even harder to maintain.
It is a violation of the so called _open closed principle_.

It is recommended to write rather small methods for dedicated small tasks and
then compose these methods to higher-level business logic. These higher-level
methods are easier to understand because they invoke the lower-level methods to
accomplish their task. Developers do not have to care about how the lowest level
works if they just want to explore the higher-level procedure.

An example of a software design pattern to help with decoupling of logic is the
_strategy pattern_. Applied to the example above it looks as follows.

```apex
interface AccountProvider {
  List<Account> getAccounts();
}
class LatestAccountsProvider implements AccountProvider {
  public List<Account> getAccounts() {
    return [SELECT Name FROM Account ORDER BY CreatedDate DESC];
  }
}
class LatestAccountsMinEmployeesProvider implements AccountProvider {
  private final Integer minNumberOfEmployees;
  public LatestAccountsMinEmployeesProvider(Integer minNumberOfEmployees) {
    this.minNumberOfEmployees = minNumberOfEmployees;
  }
  public List<Account> getAccounts() {
    return [
      SELECT Name FROM Account
      WHERE NumberOfEmployees >= :minNumberOfEmployees
      ORDER BY CreatedDate DESC
    ];
  }
}
class AccountCleaner {
  private final AccountProvider provider;
  public AccountCleaner(AccountProvider provider) {
    this.provider = provider;
  }
  void cleanupDuplicateAccounts() {
    List<Account> accounts = provider.getAccounts();
    Set<String> visitedAccountNames = new Set<String>();
    List<Account> duplicateAccounts = new List<Account>();
    for (Account account : accounts) {
      if (visitedAccountNames.contains(account.Name)) {
        duplicateAccounts.add(account);
        continue;
      }
      visitedAccountNames.add(account.Name);
    }
    if (!duplicateAccounts.isEmpty()) {
      delete duplicateAccounts;
    }
  }
}
```

The cleanup procedure is strictly separated from the logic providing the Account
records. The higher-level business logic can create the appropriate provider and
then pass it to the cleaner instance.

```apex
AccountProvider latestAccountsProvider = new LatestAccountsMinEmployeesProvider(500);
new AccountCleaner(latestAccountsProvider).cleanupDuplicateAccounts();
```

## Prefer instance methods over static methods

> Instance methods support better extensibility and unit testing.

::: danger BAD

```apex
class AccountProvider {
  public static List<Account> getAccountsByName(Set<String> names) {
    return [SELECT Name FROM Account WHERE Name IN :names];
  }
}
class AccountProcessor {
  void processAccounts() {
    Set<String> accountNames = new Set<String>{ 'foo', 'bar' };
    List<Account> accounts = AccountProvider.getAccountsByName(accountNames);
    // Do something with accounts
  }
}
```

The processor is hard-wired with the provider making it impossible to extend how
Account records are loaded. Testing the processor means that the provider is
implicitly tested as well.

:::

::: tip BETTER

```apex
class AccountProvider {
  public List<Account> getAccountsByName(Set<String> names) {
    return [SELECT Name FROM Account WHERE Name IN :names];
  }
}
class AccountProcessor {
  private final AccountProvider provider;
  public AccountProcessor(AccountProvider provider) {
    this.provider = provider;
  }
  void processAccounts() {
    Set<String> accountNames = new Set<String>{ 'foo', 'bar' };
    List<Account> accounts = provider.getAccountsByName(accountNames);
    // Do something with accounts
  }
}
```

The provider can be injected into the processor which enables extensibility and
allows to plug in a fake implementation during unit tests.

:::

Composing higher-level business logic using static methods leads to a hard-wired
graph of dependencies among classes. This makes it impossible to exchange only
certain parts of the lower-level logic while reusing the rest.

Moreover, testing higher-level classes becomes more complicated because their
methods always invoke the lower-level logic as well. The unit tests of such
higher-level classes are typically bloated with test setup code to establish the
proper test conditions which meet the requirements of all the lower-level
classes.

It is recommended to use instance methods because they promote loose coupling
which gives the flexibility to replace certain parts of the logic to achieve a
different result. For example, being able to exchange the Account provider in
the example above makes the processor extensible for different use cases.

Furthermore, when a class uses instance methods of its dependencies the behavior
of those dependencies can be mocked during unit tests. This means that the real
implementation is replaced with a fake implementation to simulate a cetain test
scenario. This can save a lot of test setup code in the higher-level unit tests.

See the
[FFLib ApexMocks Framework](https://github.com/apex-enterprise-patterns/fflib-apex-mocks)
for an easy to use mocking library.
