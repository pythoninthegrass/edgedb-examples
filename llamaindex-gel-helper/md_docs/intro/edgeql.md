# EdgeQL

EdgeQL is the query language of Gel. It’s intended as a spiritual successor to SQL that solves some of its biggest design limitations. This page is intended as a rapid-fire overview so you can hit the ground running with Gel. Refer to the linked pages for more in-depth documentation.

The examples below also demonstrate how to express the query with the TypeScript client’s query builder, which lets you express arbitrary EdgeQL queries in a code-first, typesafe way.

## Scalar literals

Gel has a rich primitive type system consisting of the following data types.

| --- | --- |
| Strings | str |
| Booleans | bool |
| Numbers | int16 int32 int64 float32 float64 bigint decimal |
| UUID | uuid |
| JSON | json |
| Dates and times | datetime cal::local_datetime cal::local_date cal::local_time |
| Durations | duration cal::relative_duration cal::date_duration |
| Binary data | bytes |
| Auto-incrementing counters | sequence |
| Enums | enum<x, y, z> |

Basic literals can be declared using familiar syntax.

### edgeql-repl

```edgeql-repl
db> select "I ❤️ EdgeQL"; # str
{'U ❤️ EdgeQL'}
db> select false; # bool
{false}
db> select 42; # int64
{42}
db> select 3.14; # float64
{3.14}
db> select 12345678n; # bigint
{12345678n}
db> select 15.0e+100n;  # decimal
{15.0e+100n}
db> select b'bina\\x01ry'; # bytes
{b'bina\\x01ry'}
```

### typescript

```typescript
e.str("I ❤️ EdgeQL")
// string
e.bool(false)
// boolean
e.int64(42)
// number
e.float64(3.14)
// number
e.bigint(BigInt(12345678))
// bigint
e.decimal("1234.4567")
// n/a (not supported by JS clients)
e.bytes(Buffer.from("bina\\x01ry"))
// Buffer
```

Other type literals are declared by casting an appropriately structured string.

### edgeql-repl

```edgeql-repl
db> select <uuid>'a5ea6360-75bd-4c20-b69c-8f317b0d2857';
{a5ea6360-75bd-4c20-b69c-8f317b0d2857}
db> select <datetime>'1999-03-31T15:17:00Z';
{<datetime>'1999-03-31T15:17:00Z'}
db> select <duration>'5 hours 4 minutes 3 seconds';
{<duration>'5:04:03'}
db> select <cal::relative_duration>'2 years 18 days';
{<cal::relative_duration>'P2Y18D'}
```

### typescript

```typescript
e.uuid("a5ea6360-75bd-4c20-b69c-8f317b0d2857")
// string
e.datetime("1999-03-31T15:17:00Z")
// Date
e.duration("5 hours 4 minutes 3 seconds")
// gel.Duration (custom class)
e.cal.relative_duration("2 years 18 days")
// gel.RelativeDuration (custom class)
```

Primitive data can be composed into arrays and tuples, which can themselves be nested.

### edgeql-repl

```edgeql-repl
db> select ['hello', 'world'];
{['hello', 'world']}
db> select ('Apple', 7, true);
{('Apple', 7, true)} # unnamed tuple
db> select (fruit := 'Apple', quantity := 3.14, fresh := true);
{(fruit := 'Apple', quantity := 3.14, fresh := true)} # named tuple
db> select <json>["this", "is", "an", "array"];
{"[\"this\", \"is\", \"an\", \"array\"]"}
```

### typescript

```typescript
e.array(["hello", "world"]);
// string[]
e.tuple(["Apple", 7, true]);
// [string, number, boolean]
e.tuple({fruit: "Apple", quantity: 3.14, fresh: true});
// {fruit: string; quantity: number; fresh: boolean}
e.json(["this", "is", "an", "array"]);
// unknown
```

Gel also supports a special json type for representing unstructured data. Primitive data structures can be converted to JSON using a type cast (<json>). Alternatively, a properly JSON-encoded string can be converted to json with the built-in to_json function. Indexing a json value returns another json value.

### edgeql-repl

```edgeql-repl
gel> select <json>5;
{"5"}
gel> select <json>[1,2,3];
{"[1, 2, 3]"}
gel> select to_json('[{ "name": "Peter Parker" }]');
{"[{\"name\": \"Peter Parker\"}]"}
gel> select to_json('[{ "name": "Peter Parker" }]')[0]['name'];
{"\"Peter Parker\""}
```

### typescript

```typescript
/*
  The result of an query returning `json` is represented
  with `unknown` in TypeScript.
*/
e.json(5);  // => unknown
e.json([1, 2, 3]);  // => unknown
e.to_json('[{ "name": "Peter Parker" }]');  // => unknown
e.to_json('[{ "name": "Peter Parker" }]')[0]["name"];  // => unknown
```

Refer to Docs > EdgeQL > Literals for complete docs.

## Functions and operators

Gel provides a rich standard library of functions to operate and manipulate various data types.

### edgeql-repl

```edgeql-repl
db> select str_upper('oh hi mark');
{'OH HI MARK'}
db> select len('oh hi mark');
{10}
db> select uuid_generate_v1mc();
{c68e3836-0d59-11ed-9379-fb98e50038bb}
db> select contains(['a', 'b', 'c'], 'd');
{false}
```

### typescript

```typescript
e.str_upper("oh hi mark");
// string
e.len("oh hi mark");
// number
e.uuid_generate_v1mc();
// string
e.contains(["a", "b", "c"], "d");
// boolean
```

Similarly, it provides a comprehensive set of built-in operators.

### edgeql-repl

```edgeql-repl
db> select not true;
{false}
db> select exists 'hi';
{true}
db> select 2 + 2;
{4}
db> select 'Hello' ++ ' world!';
{'Hello world!'}
db> select '😄' if true else '😢';
{'😄'}
db> select <duration>'5 minutes' + <duration>'2 hours';
{<duration>'2:05:00'}
```

### typescript

```typescript
e.op("not", e.bool(true));
// booolean
e.op("exists", e.set("hi"));
// boolean
e.op("exists", e.cast(e.str, e.set()));
// boolean
e.op(e.int64(2), "+", e.int64(2));
// number
e.op(e.str("Hello "), "++", e.str("World!"));
// string
e.op(e.str("😄"), "if", e.bool(true), "else", e.str("😢"));
// string
e.op(e.duration("5 minutes"), "+", e.duration("2 hours"))
```

See Docs > Standard Library for reference documentation on all built-in types, including the functions and operators that apply to them.

## Insert an object

Objects are created using insert. The insert statement relies on developer-friendly syntax like curly braces and the := operator.

### edgeql

```edgeql
insert Movie {
  title := 'Doctor Strange 2',
  release_year := 2022
};
```

### typescript

```typescript
const query = e.insert(e.Movie, {
  title: 'Doctor Strange 2',
  release_year: 2022
});

const result = await query.run(client);
// {id: string}
// by default INSERT only returns
// the id of the new object
```

See Docs > EdgeQL > Insert.

## Nested inserts

One of EdgeQL’s greatest features is that it’s easy to compose. Nested inserts are easily achieved with subqueries.

### edgeql

```edgeql
insert Movie {
  title := 'Doctor Strange 2',
  release_year := 2022,
  director := (insert Person {
    name := 'Sam Raimi'
  })
};
```

### typescript

```typescript
const query = e.insert(e.Movie, {
  title: 'Doctor Strange 2',
  release_year: 2022,
  director: e.insert(e.Person, {
    name: 'Sam Raimi'
  })
});

const result = await query.run(client);
// {id: string}
// by default INSERT only returns
// the id of the new object
```

## Select objects

Use a shape to define which properties to select from the given object type.

### edgeql

```edgeql
select Movie {
  id,
  title
};
```

### typescript

```typescript
const query = e.select(e.Movie, () => ({
  id: true,
  title: true
}));
const result = await query.run(client);
// {id: string; title: string; }[]

// To select all properties of an object, use the
// spread operator with the special "*"" property:
const query = e.select(e.Movie, () => ({
  ...e.Movie['*']
}));
```

Fetch linked objects with a nested shape.

### edgeql

```edgeql
select Movie {
  id,
  title,
  actors: {
    name
  }
};
```

### typescript

```typescript
const query = e.select(e.Movie, () => ({
  id: true,
  title: true,
  actors: {
    name: true,
  }
}));

const result = await query.run(client);
// {id: string; title: string, actors: {name: string}[]}[]
```

See Docs > EdgeQL > Select > Shapes.

## Filtering, ordering, and pagination

The select statement can be augmented with filter, order by, offset, and limit clauses (in that order).

### edgeql

```edgeql
select Movie {
  id,
  title
}
filter .release_year > 2017
order by .title
offset 10
limit 10;
```

### typescript

```typescript
const query = e.select(e.Movie, (movie) => ({
  id: true,
  title: true,
  filter: e.op(movie.release_year, ">", 1999),
  order_by: movie.title,
  offset: 10,
  limit: 10,
}));

const result = await query.run(client);
// {id: string; title: number}[]
```

Note that you reference properties of the object to include in your select by prepending the property name with a period: .release_year. This is known as leading dot notation.

Every new set of curly braces introduces a new scope. You can add filter, limit, and offset clauses to nested shapes.

### edgeql

```edgeql
select Movie {
  title,
  actors: {
    name
  } filter .name ilike 'chris%'
}
filter .title ilike '%avengers%';
```

### typescript

```typescript
e.select(e.Movie, movie => ({
  title: true,
  characters: c => ({
    name: true,
    filter: e.op(c.name, "ilike", "chris%"),
  }),
  filter: e.op(movie.title, "ilike", "%avengers%"),
}));
// => { characters: { name: string; }[]; title: string; }[]

const result = await query.run(client);
// {id: string; title: number}[]
```

See Filtering, Ordering, and Pagination.

## Query composition

We’ve seen how to insert and select. How do we do both in one query? Answer: query composition. EdgeQL’s syntax is designed to be composable, like any good programming language.

### edgeql

```edgeql
select (
  insert Movie { title := 'The Marvels' }
) {
  id,
  title
};
```

### typescript

```typescript
const newMovie = e.insert(e.Movie, {
  title: "The Marvels"
});
const query = e.select(newMovie, () => ({
  id: true,
  title: true
}));

const result = await query.run(client);
// {id: string; title: string}
```

We can clean up this query by pulling out the insert statement into a with block. A with block is useful for composing complex multi-step queries, like a script.

### edgeql

```edgeql
with new_movie := (insert Movie { title := 'The Marvels' })
select new_movie {
  id,
  title
};
```

### typescript

```typescript
/*
  Same as above.

  In the query builder, explicit ``with`` blocks aren't necessary!
  Just assign your EdgeQL subqueries to variables and compose them as you
  like. The query builder automatically convert your top-level query to an
  EdgeQL expression with proper ``with`` blocks.
*/
```

## Computed properties

Selection shapes can contain computed properties.

### edgeql

```edgeql
select Movie {
  title,
  title_upper := str_upper(.title),
  cast_size := count(.actors)
};
```

### typescript

```typescript
e.select(e.Movie, movie => ({
  title: true,
  title_upper: e.str_upper(movie.title),
  cast_size: e.count(movie.actors)
}))
// {title: string; title_upper: string; cast_size: number}[]
```

A common use for computed properties is to query a link in reverse; this is known as a backlink and it has special syntax.

### edgeql

```edgeql
select Person {
  name,
  acted_in := .<actors[is Content] {
    title
  }
};
```

### typescript

```typescript
e.select(e.Person, person => ({
  name: true,
  acted_in: e.select(person["<actors[is Content]"], () => ({
    title: true,
  })),
}));
// {name: string; acted_in: {title: string}[];}[]
```

See Docs > EdgeQL > Select > Computed fields and Docs > EdgeQL > Select > Backlinks.

## Update objects

The update statement accepts a filter clause up-front, followed by a set shape indicating how the matching objects should be updated.

### edgeql

```edgeql
update Movie
filter .title = "Doctor Strange 2"
set {
  title := "Doctor Strange in the Multiverse of Madness"
};
```

### typescript

```typescript
const query = e.update(e.Movie, (movie) => ({
  filter: e.op(movie.title, '=', 'Doctor Strange 2'),
  set: {
    title: 'Doctor Strange in the Multiverse of Madness',
  },
}));

const result = await query.run(client);
// {id: string}
```

When updating links, the set of linked objects can be added to with +=, subtracted from with -=, or overwritten with :=.

### edgeql

```edgeql
update Movie
filter .title = "Doctor Strange 2"
set {
  actors += (select Person filter .name = "Rachel McAdams")
};
```

### typescript

```typescript
e.update(e.Movie, (movie) => ({
  filter: e.op(movie.title, '=', 'Doctor Strange 2'),
  set: {
    actors: {
      "+=": e.select(e.Person, person => ({
        filter: e.op(person.name, "=", "Rachel McAdams")
      }))
    }
  },
}));
```

See Docs > EdgeQL > Update.

## Delete objects

The delete statement can contain filter, order by, offset, and limit clauses.

### edgeql

```edgeql
delete Movie
filter .title ilike "the avengers%"
limit 3;
```

### typescript

```typescript
const query = e.delete(e.Movie, (movie) => ({
  filter: e.op(movie.title, 'ilike', "the avengers%"),
}));

const result = await query.run(client);
// {id: string}[]
```

See Docs > EdgeQL > Delete.

## Query parameters

You can reference query parameters in your queries with $<name> notation. Since EdgeQL is a strongly typed language, all query parameters must be prepending with a type cast to indicate the expected type.

Scalars like str, int64, and json are supported. Tuples, arrays, and object types are not.

### edgeql

```edgeql
insert Movie {
  title := <str>$title,
  release_year := <int64>$release_year
};
```

### typescript

```typescript
const query = e.params({ title: e.str, release_year: e.int64 }, ($) => {
  return e.insert(e.Movie, {
    title: $.title,
    release_year: $.release_year,
  }))
};

const result = await query.run(client, {
  title: 'Thor: Love and Thunder',
  release_year: 2022,
});
// {id: string}
```

All client libraries provide a dedicated API for specifying parameters when executing a query.

### javascript

```javascript
import {createClient} from "gel";

const client = createClient();
const result = await client.query(`select <str>$param`, {
  param: "Play it, Sam."
});
// => "Play it, Sam."
```

### python

```python
import gel

client = gel.create_async_client()

async def main():
    result = await client.query("select <str>$param", param="Play it, Sam")
    # => "Play it, Sam"
```

### go

```go
package main

import (
    "context"
    "log"

    "github.com/geldata/gel-go"
)

func main() {
    ctx := context.Background()
    client, err := gel.CreateClient(ctx, gel.Options{})
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    var (
        param     string = "Play it, Sam."
        result  string
    )

    query := "select <str>$0"
    err = client.Query(ctx, query, &result, param)
    // ...
}
```

### rust

```rust
// [dependencies]
// gel-tokio = "0.5.0"
// tokio = { version = "1.28.1", features = ["macros", "rt-multi-thread"] }

#[tokio::main]
async fn main() {
    let conn = gel_tokio::create_client()
        .await
        .expect("Client initiation");
    let param = "Play it, Sam.";
    let val = conn
        .query_required_single::<String, _>("select <str>$0", &(param,))
        .await
        .expect("Returning value");
    println!("{val}");
}
```

See Docs > EdgeQL > Parameters.

## Subqueries

Unlike SQL, EdgeQL is composable; queries can be naturally nested. This is useful, for instance, when performing nested mutations.

### edgeql

```edgeql
with
  dr_strange := (select Movie filter .title = "Doctor Strange"),
  benedicts := (select Person filter .name in {
    'Benedict Cumberbatch',
    'Benedict Wong'
  })
update dr_strange
set {
  actors += benedicts
};
```

### typescript

```typescript
// select Doctor Strange
const drStrange = e.select(e.Movie, movie => ({
  filter: e.op(movie.title, '=', "Doctor Strange")
}));

// select actors
const actors = e.select(e.Person, person => ({
  filter: e.op(person.name, 'in', e.set(
    'Benedict Cumberbatch',
    'Benedict Wong'
  ))
}));

// add actors to cast of drStrange
const query = e.update(drStrange, ()=>({
  actors: { "+=": actors }
}));
```

We can also use subqueries to fetch properties of an object we just inserted.

### edgeql

```edgeql
 with new_movie := (insert Movie {
   title := "Avengers: The Kang Dynasty",
   release_year := 2025
 })
 select new_movie {
  title, release_year
};
```

### typescript

```typescript
// "with" blocks are added automatically
// in the generated query!

const newMovie = e.insert(e.Movie, {
  title: "Avengers: The Kang Dynasty",
  release_year: 2025
});

const query = e.select(newMovie, ()=>({
  title: true,
  release_year: true,
}));

const result = await query.run(client);
// {title: string; release_year: number;}
```

See Docs > EdgeQL > Select > Subqueries.

## Polymorphic queries

Consider the following schema.

```sdl
abstract type Content {
  required title: str;
}

type Movie extending Content {
  release_year: int64;
}

type TVShow extending Content {
  num_seasons: int64;
}
```

We can select the abstract type Content to simultaneously fetch all objects that extend it, and use the [is <type>] syntax to select properties from known subtypes.

### edgeql

```edgeql
select Content {
  title,
  [is TVShow].num_seasons,
  [is Movie].release_year
};
```

### typescript

```typescript
const query = e.select(e.Content, (content) => ({
  title: true,
  ...e.is(e.Movie, {release_year: true}),
  ...e.is(e.TVShow, {num_seasons: true}),
}));
/* {
  title: string;
  release_year: number | null;
  num_seasons: number | null;
}[] */
```

See Docs > EdgeQL > Select > Polymorphic queries.

## Grouping objects

Unlike SQL, EdgeQL provides a top-level group statement to compute groupings of objects.

### edgeql

```edgeql
group Movie { title, actors: { name }}
by .release_year;
```

### typescript

```typescript
e.group(e.Movie, (movie) => {
  const release_year = movie.release_year;
  return {
    title: true,
    by: {release_year},
  };
});
/* {
  grouping: string[];
  key: { release_year: number | null };
  elements: { title: string; }[];
}[] */
```

See Docs > EdgeQL > Group.

