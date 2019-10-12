---
title: "Rob'n Go security pearls: Database Language injections"
date: 2019-09-12T16:20:23+02:00
categories: ["Web security"]
tags: ["web","security","golang"]
authors: ["Rob"]
draft: true
---
> This post is part of my developer-friendly security series. You can find all other posts [here](https://blogtitle.github.io/categories/web-security/).

A database language injection is a vulnerability that in most cases leads to complete compromise of the service integrity, confidentiality and availability. This is one of the most critical vulnerabilities a service could have.

Usually when a program has to communicate with a database (DB) it performs so-called "queries" in a structured formal language. The most commonly known language to do this is SQL but it is not the only one and it is not the only one affected by this problem.

Queries usually come in the form of code and some data to run that code on:

```sql
SELECT name FROM users WHERE id = 'admin';
```
In this example:

* `SELECT`, `FROM`, `WHERE`, `=` and `''` are parts of code
* `name`, `users` and `id` are special kind of data: they are referring to the underlying schema, but they can be arbitrary words as long as they were pre-defined in the DB
* `admin` is just data

An **injection vulnerability** manifests when an attacker can make the service believe some data is code instead.

# A vulnerable example
Let's say we want to check if a given logged in user is also an admin. The following method queries for an admin with the given username and if at least one exists it returns `true`, `false` in all other cases (including errors, just to be sure).

```go
func (s *server) isAdmin(username string) bool {
  rows, err := s.db.Query("SELECT * FROM users WHERE isAdmin AND username = '" +
    username + "';")
  if err!=nil {
    return false
  }
  defer rows.Close()
  return rows.Next()
}
```
The problem with this code is that it doesn't **just** do what it is intended to.

If, for example, an attacker can choose their username, they could name themselves `' OR '1'='1`. This would make the query assemble in an unexpected way:
```sql
SELECT * FROM users WHERE isAdmin AND username = '' OR '1'='1';
```
This query will return **all users**, thus bypassing the check.

# Protecting from it
For sake of simplicity 


