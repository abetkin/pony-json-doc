## Overview

[JSON](https://en.wikipedia.org/wiki/JSON) support has been added lately in all major database providers.
Being able to have dynamic fields in your entities  
Pony adds support for JSON fields as well.

JSON support introduces dynamic data structures commonly found in NoSQL databases.
Usually they are used when working with highly varying data or
when the exact data structure is hard to predict.

Pony allows you to store
such data in your database without sacrificing rich data searching and
filtering capabilities.

### Declaring a JSON field

Usually you define an attribute passing the underlying Python type, such as `str` or `bool`
as a parameter. With Json, you pass `Json` type instead:

```python
from pony.orm import Json

class TShirt(db.Entity):
    article = PrimaryKey(str)
    color = Optional(str)
    sizes = Required(Json)
    extra_info = Optional(Json)

```

In the example above we have a T-Shirts database, where we have some required attributes
like color and sizes left as well as some extra info that can be provided by the manufacturer.
`sizes` attribute is supposed to be a set. With JSON, a way to model a set is to use a
boolean dict:

```python
def sizes(*values):
    return {size: '+' for size in values}
``` 

Probably, you can guess how we can add a T-Shirt item:
```python
with db_session:
    TShirt(article='A-235',
           color='magenta',
           sizes=sizes('S', 'M'),
           extra_info={
            'laundry_temp': 40, 'ironing_temp': 200,
           })
```

### Reading and writing of JSON attributes

You can read and write JSON attributes as any others. Suppose, one day the S-sized t-shirt
found its customer:

```python
with db_session:
    ts = TShirt['A-235']
    assert('S' in ts.sizes)
    new_sizes = [size for size in ts.sizes if size != 'S')
    ts.sizes = sizes(*new_sizes)
    # changes will be flushed into the database
```

### Modifying JSON data

Besides setting a new value for JSON field, you can also modify the chunks of it.
For example, in the example above we could write:

```python
with db_session:
    ts = TShirt['A-235']
    ts.sizes.pop('S')
    # changes will be flushed into the database
```
Pony will track changes to `sizes` attribute properly.

`Note`: Although in Python `dict` and `list` instances are mutable,
Pony still can track changes when the attributes are modified.
In order to do this, custom `dict` and `list` objects are returned
when Json attributes are fetched.

### Querying JSON paths

What native database support of JSON really provides to us is querying JSON data.

When querying with Python generators, you can specify JSON path,
using dictionary item access, either int or str.
Eg.: `obj.data['items'][0]['title']`.

Taking our clothes store example, suppose we want to query for all M-sized t-shirts:

```python
>>> select(ts.article for ts in TShirt if ts.sizes['M'] == '+')[:]
['A-235']
```

Within queries, `dict` is associated with JSON type. For example, here is how you can find element
by its JSON value:
```python
get(o for o in MyEntity if o.data == {'some': 'data'})
```
The `list` type will probably be used for the array type in databases supporting it, eg., Postgres,
so currently you should wrap it in a `Json` object:

```python
select(o for o in MyEntity if o.list_data == Json(['some', 'data']))
```

#### "Contains JSON" and "has key" queries 

Pony has support for both these PostgresQL features
(`<@` and `?` operators, read more in PostgreSQL [docs](http://www.postgresql.org/docs/9.5/static/functions-json.html#FUNCTIONS-JSONB-OP-TABLE)
).

 With "contains" (`<@`) you can query all the instances containing a given subset of data,
 while "has key" (`?`) checks json contains the specified key at the top level.
 
Pony syntax for both queries is Python `in` operator.

Using "has key" feature, we can select all S-sized t-shirts with
```python
select(t for t in TShirt if 'S' in t.sizes)
```

In the same way we can check JSON contains some part

```python
select(t for t in TShirt if {'laundry_temp': 40} in t.extra_info)
```

`Supported in:` PostgreSQL.

#### JSON concatenation

In PostgreSQL, you can concatenate JSON data using `||` operator
(read more in PostgreSQL [docs](http://www.postgresql.org/docs/9.5/static/functions-json.html#FUNCTIONS-JSONB-OP-TABLE))
.

```python
select(e.data | e.extra_data for e in MyEntity)
```
Besides an attribute, `extra_data` could be a Python dict as well.

`Supported in:` PostgreSQL. 


#### Wildcards in JSON paths

A nice feature is offered by Oracle/MySql: you can use a wildcard to represent "any value".
The Python syntax for this depends on whether you want to substitute integer or string.
In the former case you should use an empty slice(`[:]`), in the latter - an Ellipsis slice
(`[...]`).

Let's have a look at some example. Suppose our t-shirt store is really an online one
and has to show some pictures for the t-shirts for sale.

```python
class TShirt(db.Entity):
    # ...
    pictures = Optional(Json)

with db_session:
    ts = TShirt['A-235']
    ts.pictures = [
        {
            'url': 'http://mypictures.com/3213',
            'description': 'Front side',
        },
        {
            'url': 'http://mypictures.com/3214',
            'description': 'Back side',
        },
    ]
```

Then suppose you want to fetch all pictures. You can select all the URLs with:
```python
>>> select(ts.pictures[:]['url'] for ts in TShirt)[:]
[['http://mypictures.com/3213', 'http://mypictures.com/3214']]
```

In a similar way you can substitute a string key with a wildcard. Something like
```python
obj.items_dict[...]['nested attribute']
```



`Supported in:` MySql, Oracle

#### Length of json value

You can reference the length of JSON data in queries:

```python
select(obj for obj in MyEntity if len(obj.data) == 1)
```

`Note:` Not available in sqlite when used without the `json1` extension.

### Notes on database support

#### SQLite

With SQLite, it is highly recommended to use `json1` extension. Otherwise, there
will be severe performance penalty for JSON paths queries. 