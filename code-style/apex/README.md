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

## Prefer literal values over constants

Coming soon

## Prefer early returns over else blocks

Coming soon

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

Coming soon
