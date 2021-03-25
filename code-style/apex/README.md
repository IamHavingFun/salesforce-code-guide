# Apex Code Style Guide

This document explains some code style guidelines to make Apex code more
readable and easier to understand.

## Prefer readable variable names

Coming soon

## Prefer readable method names

Coming soon

## Prefer literal values over constants

Coming soon

## Prefer early returns over else blocks

Coming soon

## Prefer methods without unexpected side effects

> Methods without unexpected side effects make it easier to understand how the
> software works.

::: danger BAD

```apex
class AccountProvider {
  Account getDefaultAccount() {
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
class AccountProvider {
  Account getDefaultAccount() {
    return [SELECT Id, Name FROM Account WHERE Name = 'Default' LIMIT 1];
  }
  Account createDefaultAccount() {
    Account defaultAccount = new Account(Name = 'Default');
    insert defaultAccount;
    return defaultAccount;
  }
  Account ensureDefaultAccountExists() {
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
