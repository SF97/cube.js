---
title: Pre-aggregations
permalink: /schema/reference/pre-aggregations
scope: cubejs
category: Reference
subCategory: Reference
menuOrder: 8
redirect_from:
  - /pre-aggregations
---

<!-- prettier-ignore-start -->
[[info |]]
| The Cube.js pre-aggregations workshop is on August 18th at 9-11 am PST! If you
| want to learn why/when you want to use pre-aggregations, how to get started,
| tips & tricks, you will want to attend this event 😀 <br/> You can register
| for the workshop at [the event
| page](https://cube.dev/events/pre-aggregations/).
<!-- prettier-ignore-end -->

Pre-aggregations are materialized query results persisted as tables. Cube.js has
an ability to analyze queries against a defined set of pre-aggregation rules in
order to choose the optimal one that will be used to create pre-aggregation
table.

If Cube.js finds a suitable pre-aggregation rule, database querying becomes a
multi-stage process:

1. Cube.js checks if an up-to-date copy of the pre-aggregation exists.

2. Cube.js will execute a query against the pre-aggregated tables instead of the
   raw data.

Pre-aggregations can be defined in the `preAggregations` available on each cube.

## Naming

Pre-aggregations must have, at minimum, a name and a type. This name, along with
the name of the cube will be used as a prefix for pre-aggregation tables created
in the database.

Pre-aggregation names should:

- Be unique within a cube
- Start with a lowercase letter
- Consist of numbers, letters and `_`

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    ordersByStatus: {
      dimensions: [CUBE.status],
      measures: [CUBE.count],
    },
  },
});
```

Pre-aggregations must include all dimensions, measures, and filters you will
query with.

## Parameters

### type

Cube.js supports three types of pre-aggregations:

- [`rollup`][self-rollup]
- [`originalSql`][self-originalsql]
- [`rollupJoin`][self-rollupjoin]

Default type is `rollup`.

<h4 id="parameters-type-rollup">
rollup
</h4>

Rollup pre-aggregations are the most effective way to boost performance of any
analytical application. The blazing fast performance of tools like Google
Analytics or Mixpanel are backed by a similar concept. The theory behind it lies
in multi-dimensional analysis, and a rollup pre-aggregation is the result of a
[roll-up operation on an OLAP cube][wiki-olap-ops]. A rollup pre-aggregation is
essentially the summarized data of the original cube grouped by any selected
dimensions of interest.

The most performant kind of rollup pre-aggregation is an **additive** rollup:
all measures of which are based on [decomposable aggregate
functions][wiki-composable-agg-fn]. Additive measure types are: `count`, `sum`,
`min`, `max` or `countDistinctApprox`. The performance boost in this case is
based on two main properties of additive rollup pre-aggregations:

1. A rollup pre-aggregation table usually contains many fewer rows than its'
   corresponding original fact table. The fewer dimensions that are selected for
   roll-up means fewer rows in the materialized result. A smaller number of rows
   therefore means less time to query rollup pre-aggregation tables.

2. If your query is a subset of dimensions and measures of an additive rollup,
   then it can be used to calculate a query without accessing the raw data. The
   more dimensions and measures are selected for roll-up, the more queries can
   use this particular rollup.

Rollup definitions can contain members from a single cube as well as from
multiple cubes. In case of multiple cubes being involved, the join query will be
built according to the standard rules of cubes joining.

Rollups are selected for querying based on properties found in queries made to
the Cube.js REST API. A thorough explanation can be found under [Getting Started
with Pre-Aggregations][ref-caching-preaggs-target].

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    ordersByCompany: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
    },
  },
});
```

<h4 id="parameters-type-originalsql">
originalSql
</h4>

As the name suggests, it persists the results of the `sql` property of the cube.
Pre-aggregations of type `originalSql` should **only** be used when the cube's
`sql` is a complex query (i.e. nested, window functions and/or multiple joins).
We **strongly** recommend only persisting results of `originalSql` back to the
source database. They often do not provide much in the way of performance
directly, but there are two specific applications:

1. They can be used in tandem with the
   [`useOriginalSqlPreAggregations`][self-origsql-preaggs] option in other
   rollup pre-aggregations.

2. Situations where it is not possible to use a `rollup` pre-aggregations, such
   as [funnels][ref-recipe-funnels].

For example, to pre-aggregate all completed orders, you could do the following:

```javascript
cube(`CompletedOrders`, {
  sql: `select * from orders where completed = true`,

  ...,

  preAggregations: {
    main: {
      type: `originalSql`
    },
  },
});
```

<h4 id="parameters-type-rollupjoin">
rollupJoin
</h4>

<!-- prettier-ignore-start -->
[[warning | 🐣 &nbsp;&nbsp; Preview]]
| `rollupJoin` is currently in Preview, and the API may change in a
| future version.
<!-- prettier-ignore-end -->

Cube.js is capable of performing joins between separate pre-aggregations,
thereby avoiding a call to the source database. This functionality also allows
for cross-database joins; you can have a data schema for a MySQL database,
another for Postgres, and then use `rollupJoin` to join their pre-aggregations:

```javascript
// `schema/Users.js` - a schema representing all users, retrieved from MySQL
cube(`Users`, {
  dataSource: 'mysql',
  sql: `SELECT * from users`,

  measures: {
    count: {
      type: `count`,
    },
  },

  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primaryKey: true,
    },
    name: {
      sql: `CONCAT(first_name, ' ', last_name)`,
      type: `string`,
    },
  },

  preAggregations: {
    usersRollup: {
      dimensions: [CUBE.id, CUBE.name],
    },
  },
});

// `schema/Orders.js` - a schema representing all orders, retrieved from Postgres
cube('Orders', {
  dataSource: 'postgres',
  sql: `select * from orders`,
  joins: {
    Users: {
      relationship: `belongsTo`,
      sql: `${CUBE}.user_id = ${Users}.id`,
    },
  },
  measures: {
    count: {
      type: `count`,
    },
  },
  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primaryKey: true,
    },
    status: {
      sql: `status`,
      type: `string`,
    },
    createdAt: {
      sql: `created_at`,
      type: `time`,
    },
    completedAt: {
      sql: `completed_at`,
      type: `time`,
    },
  },
  preAggregations: {
    ordersRollup: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      timeDimension: CUBE.createdAt,
    },
    // Here we add a new pre-aggregation of type `rollupJoin`
    ordersWithUsersRollup: {
      type: `rollupJoin`,
      measures: [CUBE.count],
      dimensions: [Users.name],
      rollups: [Users.usersRollup, CUBE.ordersRollup],
    },
  },
});
```

### measures

The `measures` property is an array of [measures from the
cube][ref-schema-measures] that should be included in the pre-aggregation:

```javascript
cube('Orders', {
  sql: `select * from orders`,

  measures: {
    count: {
      type: `count`,
    },
  },

  preAggregations: {
    usersRollup: {
      measures: [CUBE.count],
    },
  },
});
```

### dimensions

The `dimensions` property is an array of [dimensions from the
cube][ref-schema-dimensions] that should be included in the pre-aggregation:

```javascript
cube('Orders', {
  sql: `select * from orders`,

  dimensions: {
    status: {
      type: `string`,
      sql: `status`,
    },
  },

  preAggregations: {
    usersRollup: {
      dimensions: [CUBE.status],
    },
  },
});
```

### timeDimension

The `timeDimension` property can be any [`dimension`][ref-schema-dimensions] of
type [`time`][ref-schema-types-dim-time]. All other measures and dimensions in
the schema are aggregated This property is an extremely useful tool for
improving performance with massive datasets.

```javascript
cube('Orders', {
  sql: `select * from orders`,

  measures: {
    count: {
      type: `count`,
    },
  },

  dimensions: {
    status: {
      type: `string`,
      sql: `status`,
    },
    createdAt: {
      type: `time`,
      sql: `created_at`,
    },
  },

  preAggregations: {
    ordersByStatus: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
    },
  },
});
```

A [`granularity`][self-granularity] **must** also be included in the
pre-aggregation definition.

### granularity

The `granularity` property defines the granularity of data _within_ the
pre-aggregation. If set to `week`, for example, then Cube.js will pre-aggregate
the data by week and persist it to Cube Store.

```javascript
cube('Orders', {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    usersRollupByWeek: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      timeDimension: CUBE.createdAt,
      granularity: `week`,
    },
  },
});
```

The value can be one of `hour`, `day`, `week`, `month`, `year`. This property is
required when using [`timeDimension`][self-timedimension].

### segments

The `segments` property is an array of [segments from the
cube][ref-schema-segments] that can target the pre-aggregation:

<!-- prettier-ignore-start -->
[[warning | ]]
| The relevant measures/dimensions that make up the segment **must** be
| included in the pre-aggregation definition.
<!-- prettier-ignore-end -->

```javascript
cube(`Orders`, {
  sql: `SELECT * FROM orders`,

  ...,

  segments: {
    onlyComplete: {
      sql: `${CUBE}.status = 'complete'`,
    },
  },

  preAggregations: {
    main: {
      dimensions: [CUBE.status],
      segments: [CUBE.onlyComplete],
    },
  },
});
```

### partitionGranularity

The `partitionGranularity` defines the granularity for each
[partition][ref-caching-partitioning] of the pre-aggregation:

```javascript
cube('Orders', {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    usersRollup: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      partitionGranularity: `month`,
    },
  },
});
```

The value can be one of `hour`, `day`, `week`, `month`, `year`. A
[`timeDimension`][self-timedimension] and [`granularity`][self-granularity]
**must** also be included in the pre-aggregation definition. This property is
required when using [partitioned pre-aggregations][ref-caching-partitioning].

### refreshKey

Cube.js can also take care of keeping pre-aggregations up to date with the
`refreshKey` property. By default, it is set to `every: '1 hour'`.

<h4 id="parameters-refresh-key-sql">
sql
</h4>

You can set up a custom refresh check strategy by using the `sql` property:

```javascript
cube(`Orders`, {
  sql: `SELECT * FROM orders`,

  preAggregations: {
    main: {
      measures: [CUBE.count],
      refreshKey: {
        sql: `SELECT MAX(created_at) FROM orders`,
      },
    },
  },
});
```

<h4 id="parameters-refresh-key-every">
every
</h4>

The `refreshKey` can define an `every` property which can be used to refresh
pre-aggregations based on a time interval.

<!-- prettier-ignore-start -->
[[warning | ]]
| The `every` parameter **does not** force Cube.js to fetch `refreshKey` based
| on an interval. It instead generates a SQL query whose result should change
| at least once per defined interval and adjusts `refreshKeyRenewalThreshold`
| accordingly. [Learn more][ref-cube-refreshkey].
<!-- prettier-ignore-end -->

For example:

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  preAggregations: {
    main: {
      measures: [CUBE.count],
      refreshKey: {
        every: `1 day`,
      },
    },
  },
});
```

For possible `every` parameter values please refer to
[`refreshKey`][ref-cube-refreshkey] documentation.

<h4 id="parameters-refresh-key-incremental">
incremental
</h4>

You can incrementally refresh partitioned rollups by setting
`incremental: true`. This option defaults to `true`.

<!-- prettier-ignore-start -->
[[warning | ]]
| Partition tables are refreshed as a whole. When a new partition table is
| available, it replaces the old one. Old partition tables are collected by
| [Garbage Collection][ref-caching-garbage-collection]. Append is never used to
| add new rows to the existing tables.
<!-- prettier-ignore-end -->

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    main: {
      measure: [CUBE.count],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      partitionGranularity: `day`,
      refreshKey: {
        every: `1 day`,
        incremental: true,
      },
    },
  },
});
```

<h4 id="parameters-refresh-key-updatewindow">
updateWindow
</h4>

The `incremental: true` flag generates a special `refreshKey` SQL query which
triggers a refresh for partitions where the end date lies within the
`updateWindow` from the current time. In the example below, it will refresh
today's and the last 7 days of partitions once a day. Partitions before the
`7 day` interval **will not** be refreshed once they are built unless the rollup
SQL is changed.

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    main: {
      measure: [CUBE.count],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      partitionGranularity: `day`,
      refreshKey: {
        every: `1 day`,
        incremental: true,
        updateWindow: `7 day`,
      },
    },
  },
});
```

This property is required when using [`incremental`][self-incremental]
refreshes.

### useOriginalSqlPreAggregations

Cube.js supports multi-stage pre-aggregations by reusing original SQL
pre-aggregations in rollups through the `useOriginalSqlPreAggregations`
property. It is helpful in cases where you want to re-use a heavy SQL query
calculation in multiple `rollup` pre-aggregations. Without
`useOriginalSqlPreAggregations` enabled, Cube.js will always re-execute all
underlying SQL calculations every time it builds new rollup tables.

<!-- prettier-ignore-start -->
[[warning |]]
| `originalSql` pre-aggregations **must only** be used when [storing
| pre-aggregations on the source database][ref-caching-using-preaggs-internal].
| This also means that `originalSql` pre-aggregations require
| [`readOnly: false`][ref-caching-readonly].
<!-- prettier-ignore-end -->

```javascript
cube(`Orders`, {
  sql: `
    select * from orders1
    UNION ALL
    select * from orders2
    UNION ALL
    select * from orders3
    `,

  ...,

  preAggregations: {
    main: {
      type: `originalSql`,
    },
    statuses: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      useOriginalSqlPreAggregations: true,
    },
    completedOrders: {
      measures: [CUBE.count],
      timeDimension: CUBE.completedAt,
      granularity: `day`,
      useOriginalSqlPreAggregations: true,
    },
  },
});
```

### scheduledRefresh

To always keep pre-aggregations up-to-date, you can set
`scheduledRefresh: true`. This option defaults to `true`. If set to `false`,
pre-aggregations will always be built on-demand. The
[`refreshKey`][self-refreshkey] is used to determine if there's a need to update
specific pre-aggregations on each scheduled refresh run. For partitioned
pre-aggregations, `min` and `max` dates for
[`timeDimension`][self-timedimension] are checked to determine range for the
refresh.

Each time a scheduled refresh is run, it takes every pre-aggregation partition
starting with most recent ones in time and checks if the
[`refreshKey`][self-refreshkey] has changed. If a change was detected, then that
partition will be refreshed.

In development mode, Cube.js runs the background refresh by default and will
refresh all pre-aggregations which have `scheduledRefresh: true`.

Please consult [Production Checklist][ref-production-checklist-refresh] for best
practices on running background refresh in production environments.

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  // ...

  preAggregations: {
    ordersByStatus: {
      measures: [CUBE.count],
      dimensions: [CUBE.status],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      partitionGranularity: `month`,
      scheduledRefresh: true,
    },
  },
});
```

### buildRangeStart and buildRangeEnd

The build range defines what partitions should be built by a scheduled refresh.
Scheduled refreshes will **never** look beyond this range.

It can be used together with `updateWindow` to define granular update settings.
Set the `updateWindow` property to the interval in which your data can change
and `buildRangeStart` to the earliest point of time when history should be
available. For example if `updateWindow` is `1 week` and `buildRangeStart` is
`SELECT NOW() - interval '365 day'` scheduled refresh will build historic
partitions for 365 days in past and will refresh only one last week according to
the `refreshKey` setting.

The refresh range for partitioned pre-aggregations can be controlled using
`buildRangeStart` and `buildRangeEnd` properties:

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    categoryAndDate: {
      measures: [CUBE.count],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      partitionGranularity: `month`,
      buildRangeStart: {
        sql: `SELECT NOW() - interval '300 day'`,
      },
      buildRangeEnd: {
        sql: `SELECT NOW()`,
      },
    },
  },
});
```

### indexes

In case of pre-aggregation tables having significant cardinality, you might want
to create indexes for them in databases which support it. This is can be done as
follows:

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    categoryAndDate: {
      measures: [CUBE.count],
      dimensions: [CUBE.category],
      timeDimension: CUBE.createdAt,
      granularity: `day`,
      indexes: {
        categoryIndex: {
          columns: [CUBE.category],
        },
      },
    },
  },
});
```

For `originalSql` pre-aggregations, the original column names as strings can be
used:

```javascript
cube(`Orders`, {
  sql: `select * from orders`,

  ...,

  preAggregations: {
    main: {
      type: `originalSql`,
      indexes: {
        timestampIndex: {
          columns: ['timestamp'],
        },
      },
    },
  },
});
```

[ref-caching-partitioning]: /caching/using-pre-aggregations#partitioning
[ref-caching-garbage-collection]:
  /caching/using-pre-aggregations#garbage-collection
[ref-caching-preaggs-target]:
  /caching/pre-aggregations/getting-started#ensuring-pre-aggregations-are-targeted-by-queries
[ref-caching-readonly]: /caching/using-pre-aggregations#read-only-data-source
[ref-caching-using-preaggs-internal]:
  /caching/using-pre-aggregations#pre-aggregations-storage
[ref-config-driverfactory]: /config/#options-reference-driver-factory
[ref-config-preagg-schema]: /config#options-reference-pre-aggregations-schema
[ref-cube-refreshkey]: /schema/reference/cube#parameters-refresh-key
[ref-production-checklist-refresh]:
  /deployment/production-checklist#set-up-refresh-worker
[ref-recipe-funnels]: /recipes/funnels
[ref-sqlalias]: /schema/reference/cube#parameters-sql-alias
[ref-schema-dimensions]: /schema/reference/dimensions
[ref-schema-measures]: /schema/reference/measures
[ref-schema-segments]: /schema/reference/segments
[ref-schema-types-dim-time]:
  /schema/reference/types-and-formats#dimensions-types-time
[self-granularity]: #parameters-granularity
[self-incremental]: #parameters-refresh-key-incremental
[self-origsql-preaggs]: #use-original-sql-pre-aggregations
[self-originalsql]: #parameters-type-originalsql
[self-refreshkey]: #parameters-refresh-key
[self-rollup]: #parameters-type-rollup
[self-rollupjoin]: #parameters-type-rollupjoin
[self-timedimension]: #parameters-time-dimension
[wiki-olap-ops]: https://en.wikipedia.org/wiki/OLAP_cube#Operations
[wiki-composable-agg-fn]:
  https://en.wikipedia.org/wiki/Aggregate_function#Decomposable_aggregate_functions
