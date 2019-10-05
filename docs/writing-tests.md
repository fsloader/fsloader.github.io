---
layout: wide
title: Writing Tests
nav_bar_name: docs
show_comments: false
---
<div class="row">
  <div class="sidebar span3">
    <ul id="sidenav" class="nav nav-list affix">
      <li class="active"><a href="#unit-test">Unit Test</a></li>
      <li><a href="#data-diff-test">Data Diff Test</a></li>
      <li><a href="#invariant-test">Invariant Test</a></li>
      <li><a href="#schema-test">Schema Test</a></li>
      <li><a href="#timing-test">Timing Test</a></li>
    </ul>
  </div>
  <div class="span9">
    <div class="page-header">
      <h1>{{ page.title }}</h1>
    </div>
    <div markdown="1">
This page catalogs:

 *  different types of tests for data-driven systems
 *  the context within which they are/aren't useful
 *  examples of how they would look like in DbFit.

----

## Unit Test

### Example

    !|insert|source   |
    |col1   |col2|col3|
    |val1   |val2|val3|

    |execute procedure|some_procedure|
    |param1           |param2        |
    |val1             |val2          |

    |query   |select * from target|
    |col1    |col2    |col3       |
    |exp_val1|exp_val2|exp_val3   |

### Description

The unit test has 3 basic stages:

 1. Set up the input data (**arrange**).
 2. Execute a function or procedure (**act**).
 3. Run a query and compare actual vs expected data (**assert**).

### Pre-requisites

An development environment where it's OK to overwrite or truncate data.

### When are such tests useful?

 *  As part of your test-driven-development cycle.
 *  While unit-testing your transformation/procedure.
 *  For functional testing of your transformation.

### When are such tests not appropriate?

When no test environments are available and arrange-act-assert tests must be run in a shared staging environment. In such cases, the risk of irrevocably modifying or deleting existing data (and upsetting your fellow developers!) is usually too great.

### Useful table types

 *  [Insert](/dbfit/docs/reference.html#insert)
 *  [Update](/dbfit/docs/reference.html#update)
 *  [Execute procedure](/dbfit/docs/reference.html#execute-procedure)
 *  [Query](/dbfit/docs/reference.html#query)
 *  [Inspect query](/dbfit/docs/reference.html#inspect)

----

## Data Diff Test

### Example

#### For small datasets

    |Store query|!-SELECT col1, col2, col3 FROM source-!|source_query|

    |Store query|!-SELECT col1, col2, col3 FROM target-!|target_query|

    |Compare stored queries|source_query|target_query|
    |col1                  |col2        |col3        |

#### For large datasets

    !define q1 {SELECT col1
                     , col2
                     , col3
                     , 'q1' as source
                  FROM table1}

    !define q2 {SELECT col1
                     , col2
                     , col3
                     , 'q2' as source
                  FROM table2}

    |query|(${q1} MINUS ${q2}) UNION (${q2} MINUS ${q1})|
    |col1 |col2          |col3          |source         |

### Description

Data diff tests find differences between resultsets (for instance, comparing the output of the current production code, and the output of the release candidate code, after both are run on the same input). It's assumed that the transformations themselves has already been executed. Any unmatched row in either resultset is automatically visible in the test results and the test fails.

While the fundamental concept remains the same for large and small datasets, the distinction is important when implementing such tests using DbFit. Using `Compare stored queries` is fine for small datasets (< 10K rows), but the current DbFit row comparison implementation doesn't scale beyond that point. For larger datasets, the comparison is best pushed into the database (see the examples above to better understand how this is done).

### Pre-requisites

None.

### When are such tests useful?

When the test coverage provided by unit or regression tests is very low, data diff tests can provide very useful feedback during release testing when run on production-like data.

### When are such tests not appropriate?

When a data diff test fails, understanding the root cause of the failure is a manual proess which can be prohibitively expensive, since even one difference between the production and release candidate code can produce millions of differences in the results. It is generally a sounder strategy to rely on a large suite of unit tests to provide precise feedback and move away from data diff tests as the test coverage increases.

### Useful table types

 *  [Query](/dbfit/docs/reference.html#query)
 *  [Store query](/dbfit/docs/reference.html#store-query)
 *  [Compare stored queries](/dbfit/docs/reference.html#compare-stored-queries)

----

## Invariant Test

### Example

    |query   |select * from table where non_negative_col < 0|
    |col1    |col2    |col3                                 |

### Description

Invariant tests detect when an invariant (a rule or statement about the data that should always be true) no longer holds.

Examples of invariants:

 *  The transformation preserves the rowcount
 *  There are no gaps in a group of time intervals
 *  Some numeric column is never negative
 *  Some string column is never null or empty

These tests can executed after a transformation (in which case they are just a query table), or can trigger the invocation. They can even be included in the teardown for unit tests, to make sure that the invariants hold in various scenarios.

### Pre-requisites

None.

### When are such tests useful?

Invariant tests are very useful as regression tests, when executed on production-like datasets.

### When are such tests not appropriate?

-

### Useful table types

 *  [Query](/dbfit/docs/reference.html#query)

----

## Schema Test

### Example

Currently, DbFit does not have any specific functionality to support this test type.

### Description

If run after a change to the schema, schema tests can ensure that the environment is in the state assumed by the subsequent tests, and failure provides early feedback that the environment itself is not set up correctly.

### Pre-requisites

None.

### When are such tests useful?

In legacy environments, it is common for databases to be upgraded by making an anonymised clone of some production-like system, rather than using [schema migrations](http://en.wikipedia.org/wiki/Schema_migration).

### When are such tests not appropriate?

If the risk around schema upgrades is low, schema tests are probably an overhead.

----

## Timing Test

### Example

<span class="label label-warning">Note:</span> this example is Oracle-specific, but can easily be adapted to any database.

    |Set parameter|threshold|10|

    !|Query|select TO_CHAR(SYSTIMESTAMP, 'SSSSS.SS') AS start_time FROM DUAL|
    |start_time?                                                            |
    |>>start_time                                                           |

    !|Execute procedure|some_procedure|

    !|Query|SELECT (TO_CHAR(SYSTIMESTAMP, 'SSSSS.SS') - :start_time) AS duration FROM DUAL|
    |duration?                                                                            |
    |>>duration                                                                           |

    !|query stats                               |
    |table name|where                 |is empty?|
    |DUAL      |:duration > :threshold|yes      |

### Description

This test measures the time it takes to run a transformation and fails if the time exceeds a certian threshold.

### Pre-requisites

For stable timings, it's best to run such tests on large datasets.

### When are such tests useful?

Such tests are useful to measure/track trends in performance and to flag potential performance problems.

### When are such tests not appropriate?

As there is an overhead of executing stored procedures through DbFit (1-2 seconds), this mechanism for measuring executing time is not accurate if the stored procedure finishes quickly.

### Useful table types

 *  [Set parameter](/dbfit/docs/reference.html#working-with-parameters)
 *  [Query](/dbfit/docs/reference.html#query)
 *  [Execute procedure](/dbfit/docs/reference.html#execute-procedure)

</div>
  </div>
</div>
