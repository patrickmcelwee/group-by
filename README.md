## group-by

##### An XQuery library for computing aggregates over tuples in MarkLogic Server

*requires MarkLogic Server v7 or greater*

[MarkLogic Server](http://developer.marklogic.com/) provides a huge API surface for all kinds of queries, but selecting tabular result-sets and computing additional columns (i.e., performing SQL-style queries in XQuery) tends to create highly-coupled and overly complicated code.

<small>**simple example**: select the *average salary* and *median bonus* for each *position* at each *company* in a set of personnel records of the following structure:</small>

```xml
<record>
  <name>John Doe</name>
  <position>rock star ninja</position>
  <company>awesome startup</company>
  <salary>50000</salary>
  <bonus>5000</bonus>
</record>
```

```xquery
for $tuple in cts:value-tuples((
  cts:element-reference(xs:QName("position")),
  cts:element-reference(xs:QName("company"))
))
let $values := json:array-values($tuple)
let $query := cts:and-query((
  cts:element-range-query(xs:QName("position"), "=", $values[1]),
  cts:element-range-query(xs:QName("company"), "=", $values[2])
))
return (
  json:array-push($tuple,
    cts:avg-aggregate(cts:element-reference(xs:QName("salary")), (), $query)),
  json:array-push($tuple,
    cts:median(cts:values(cts:element-reference(xs:QName("bonus")), $query))),
  $tuple
)
```

Given the sample document (and the appropriate index configuration), that query would generate this result:

```javascript
[ 'rock star ninja', 'awesome startup', '50000', '5000' ]
```

While this example query is readable enough, the complexity of this approach increases dramatically as query requirements proliferate.

In contrast, the **group-by** library drastically simplifies these types of operations, providing a declarative syntax for aggregation strategies and computations.

```xquery
cts:group-by((
  cts:column("position", cts:element-reference(xs:QName("position"))),
  cts:column("company", cts:element-reference(xs:QName("company"))),
  cts:compute("avg", cts:element-reference(xs:QName("salary"))),
  cts:compute("median", cts:element-reference(xs:QName("bonus")))
))
```

###### implementation note:
<blockquote>Both of these queries depend on the <a href="http://docs.marklogic.com/cts:value-tuples"><code>cts:value-tuples</code></a> function, which returns each unique tuple from a set of lexicons: an "*n*-way co-occurrence" query. Each tuple represents a set of lexicon values that occur together in one-or-more fragments (usually documents). Documents containing multiple values from the same lexicon can lead to unexpected results, as there will be more than one tuple for such a document. <a href="http://docs.marklogic.com/guide/admin/fields#chapter">Field</a> and <a href="http://docs.marklogic.com/guide/admin/range_index#id_40666">path</a> range indices can be used to mitigate this effect, as they enable more granular element or attribute selection.</blockquote>

In the spirit of <a href="https://en.wikipedia.org/wiki/Polyfill">browser polyfills</a>, this module is placed in the `http://marklogic.com/cts` namespace. It provides three query strategies: `cts:group-by`, `cts:cross-product`, and `cts:cube` ([*warning, experimental*](#func_cube)). Each accepts (up to) three parameters:

| parameter | purpose |
| --------- | ------- |
| `$type-defs` | query specification composed of `cts:row`, `cts:column`, and `cts:compute` functions |
| `$options` | optional sequence of option strings formatted as `"_option-name_=_option-value_"`. |
| `$query` | optional `cts:query` object constraining the results |

| options | purpose |
| ----------- | ------- |
| [`cts:value-tuples` options](http://docs.marklogic.com/cts:value-tuples#options) | controls how tuples are generated from `cts:row` and `cts:column` specifications |
| `headers` | boolean, include column headers in results array |
| `format` | `array` or `map`, return results as a `json:array` or a `map:map` object |

#### <a name="func_group-by"/> cts:group-by
```xquery
cts:group-by(
  $type-defs as function(*)*,
  [$options as xs:string*],
  [$query as cts:query?]
) as item()*
```

###### notes

- `cts:row` and `cts:column` are treated identically by `cts:group-by`
- regardless of the order in the specification, all rows are processed before all columns
- each computation is constrained to the values of the rows and columns in that array
- `cts:group-by` is processed recursively, allowing for complex and arbitrarily-nested result-sets (see [advanced queries](#advanced)).

###### example output

```javascript
[
  'row-1_value-1',
  'row-2_value-1',
  'column-1_value-1',
  'compute-1'
]
```

#### <a name="func_cross-product"/> cts:cross-product
```xquery
cts:cross-product(
  $type-defs as function(*)*,
  [$options as xs:string*],
  [$query as cts:query?]
) as item()*
```

###### notes

- `cts:cross-product` calculates columns over rows
- regardless of the order in the specification, all rows are processed before all columns; both rows and columns are required
- each column is constrained to the row values preceding it
- each computation is constrained in the same manner as the columns, and additionally to the values of the columns in a given array

###### example output
```javascript
[
  'row-1_value-1',
  'row-2_value-1',
  [ 
    [ 'column-1_value-1', 'compute-1' ],
    [ 'column-1_value-2', 'compute-1' ],
    [ ... ]
  ]
]
```

#### <a name="func_cube"/> cts:cube
```xquery
cts:cube(
  $type-defs as function(*)*,
  [$options as xs:string*],
  [$query as cts:query?]
) as item()*
```

###### notes

**Warning**: `cts:cube` currently is in an experimental state, and can be very computationally expensive. It's output, while logically nested, is arcane and difficult to use. In the future, it will denormalize the results into a useful tabular structure.

- `cts:cube` is intended to allow for fine grained OLAP-style cube processing.
- regardless of the order in the specification, all rows are processed before all columns; both rows and columns are required
- each column is constrained to the row values preceding it, and the row values in ancestor arrays
- each computation is constrained in the same manner as the columns, and additionally to the values of the columns in a given array

###### example output
```javascript
[
  [
    'row-1_value-1',
    'compute-1',
    [
      [ 'column-1_value-1', 'compute-1' ],
      [ 'column-1_value-2', 'compute-1' ],
      [ ... ]
    ],
    [
      'row-2_value-1',
      'compute-1',
      [
        [ 'column-1_value-1', 'compute-1' ],
        [ 'column-1_value-2', 'compute-1' ],
        [ ... ]
      ]
    ]
  ]
]
```

### query specification

Query specifications are created from three function types: `cts:row`, `cts:column`, and `cts:compute`. Each type constructs a named `cts:reference` to a lexicon.

#### <a name="references"/> lexicon reference types

- [cts:collection-reference](http://docs.marklogic.com/cts:collection-reference)
- [cts:element-attribute-reference](http://docs.marklogic.com/cts:element-attribute-reference)
- [cts:element-reference](http://docs.marklogic.com/cts:element-reference)
- [cts:field-reference](http://docs.marklogic.com/cts:field-reference)
- [cts:geospatial-attribute-pair-reference](http://docs.marklogic.com/cts:geospatial-attribute-pair-reference)
- [cts:geospatial-element-child-reference](http://docs.marklogic.com/cts:geospatial-element-child-reference)
- [cts:geospatial-element-pair-reference](http://docs.marklogic.com/cts:geospatial-element-pair-reference)
- [cts:geospatial-element-reference](http://docs.marklogic.com/cts:geospatial-element-reference)
- [cts:geospatial-path-reference](http://docs.marklogic.com/cts:geospatial-path-reference)
- [cts:path-reference](http://docs.marklogic.com/cts:path-reference)
- [cts:uri-reference](http://docs.marklogic.com/cts:uri-reference)


#### <a name="func_row"/> cts:row
```xquery
cts:row(
  $alias as xs:string,
  $reference as cts:reference,
) as (function() as element(cts:row))
```

#### <a name="func_column"/> cts:column
```xquery
cts:column(
  $alias as xs:string,
  $reference as cts:reference,
) as (function() as element(cts:column))
```

###### notes

- the names "rows" and "columns" are understood in the spirit of spreadsheet pivot tables
- they are treated identically by `cts:group-by`; the distinction between them determines the nesting of result-sets in `cts:cross-product` and `cts:cube`

#### <a name="func_compute"/> cts:compute

In a addition to a reference[<sup>1</sup>](#1), `cts:compute` requires the name of an [aggregate function](#aggregates), which is applied to the referenced lexicon.

<sub><a name="1"/>[1] **Note**: three [built-in aggregates](#aggregates) require a sequence of two lexicons</sub>

```xquery
cts:compute(
  $function as xs:string,
  $reference as cts:reference*,
) as (function() as element(cts:compute))
```

or

```xquery
cts:compute(
  $alias as xs:string,
  $function as xs:string,
  $reference as cts:reference*,
  [$options as xs:string*]
) as (function() as element(cts:compute))
```

#### parameters

###### `$alias`

When `$alias` is not provided, the computation result is aliased to the name of the aggregate function. This can lead to property/key name-collisions when returning results with the option `"format=map"`.

###### `$function`

`$function` must be either the name of one of the built-in aggregate functions (as named in the [Search, REST and Java APIs](http://docs.marklogic.com/guide/search-dev/aggregate#id_39431)), or a [user-defined aggregate function](http://docs.marklogic.com/guide/app-dev/aggregateUDFs#id_36532), formatted as `native/_plugin-name_/_function-name_`.

**Note**: three of the built-in aggregate functions require two references

| <a name="aggregates"/>built-in aggregate functions | `$reference` cardinality |
| ---------------------------- | --------------------------:|
| avg | 1 |
| correlation | 2 |
| count | 1 |
| covariance | 2 |
| covariance-population | 2 |
| max | 1 |
| median | 1 |
| min | 1 |
| stddev | 1 |
| stddev-population | 1 |
| sum | 1 |
| variance | 1 |
| variance-population | 1 |

###### `$reference`

All of the built-in aggregate functions require the `$reference` parameter to be in reference to a lexicon of numeric type, except for `count`, which accepts a `$reference` of any type.

###### <a name="numeric-types"/> numeric lexicon types
- int
- unsignedInt
- long
- unsignedLong
- float
- double
- decimal

###### `$options`

The `$options` parameter is passed to [cts:aggregate](http://docs.marklogic.com/cts:aggregate#options), except in the case of the `median` function, when it is passed to [cts:values](http://docs.marklogic.com/cts:values#options).

##### <a name="advanced"/> Advanced Queries

*FIXME*

### License Information

Copyright (c) 2014 Joseph Bryan. All Rights Reserved.

Copyright (c) 2014 Gary Vidal. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0]
(http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

The use of the Apache License does not indicate that this project is
affiliated with the Apache Software Foundation.