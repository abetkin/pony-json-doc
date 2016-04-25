## Overview

[JSON](https://en.wikipedia.org/wiki/JSON) support has been added lately in all major database providers.
Being able to have dynamic fields in your entities  
Pony adds support for JSON fields as well.

JSON support introduces dynamic data structures commonly found in NoSQL databases.
Usually they are used when working with highly varying data or
if the exact data structure is hard to predict.

Pony allows you to store
such data in your database without sacrificing rich data search and
filtering capabilities.

### Declaring a JSON field

Usually you define an attribute passing the underlying Python type, such as `str` or `bool`
as a parameter. With Json, you pass `Json` type instead:

```python
from pony.orm import Json

class DynamicEntity(db.Entity):
    uid = Required(str, primary_key=True)
    data = Optional(Json)
```

Let's create an entity:
```python
with db_session:
    DynamicEntity(uid='id-1')
```
Since `data` field is optional, we can omit it for now. First, you can read and write
Json attributes as any others:

```python
with db_session:
    obj = DynamicEntity['id-1']
    assert(obj.data is None)
    obj.data = {'items': [1, 2, 3]}
    # changes will be flushed into the database
```

### Modifying JSON data

Besides setting a new value for JSON field, you can also modify the chunks of it:

```python
with db_session:
    obj = DynamicEntity['id-1']
    obj.data['items'] = []
```
Pony will track changes to `obj` properly.

`Note`: Although in Python `dict` and `list` instances are mutable,
Pony still can track changes when the attributes are modified.
In order to do this, custom `dict` and `list` objects are returned
when Json attributes are fetched.

### Querying JSON paths

What native database support of JSON really provides to us is querying such data.

When querying with Python generators, you can specify JSON path,
using dictionary item access, either int or str.
Eg.: `obj.data['items'][0]['title']`.

Let us see another example

```python
class DatabaseVendor(db.Entity):
    name = Required(str)
    commercial = Required(bool)
    features = Optional(Json)
    supported_os = Optional(Json)
    
with db_session:
    DatabaseVendor(name='oracle', commercial=True, features={
        'clustering': True,
        'json': True,
    })
```
Suppose we want to select all databases supporting clustering:
```python
>>> select((d.name, d.features['clustering']) for d in DatabaseVendor)[:]
[('oracle', True)]
```


Within queries, `dict` is associated with JSON type. For example, here is how you can find element
by its JSON value:
```python
get(o for o in MyEntity if o.data == {'some': 'data'})
```
The `list` type will probably be used for the array type in databases supporting it, eg., Postgres,
so currently you should wrap it in a `Json` object:

```python
with db_session:
    oracle = get(d for d in DatabaseVendor if d.name == 'oracle')
    oracle.supported_os = Json(['RHEL', 'Oracle Linux', 'SUSE'])
```

### "Contains" query

In PostgreSQL, you can query all the instances containing a given subset of data

`Note:`: Works only with PostgreSQL

### JSON concatenation

In PostgreSQL, you can concatenate JSON data:
```python
select(e.data | e.extra_data for e in MyEntity)
```

`Note:`: Works only with PostgreSQL

Besides an attribute, `extra_data` could be a Python dict as well.

### Length of json value

You can reference the length of JSON data in queries:

```python
select(obj for obj in MyEntity if len(obj.data) == 1)
```

`Note:` Not available in sqlite when used without the `json1` extension.

##  Database support

The common set of features supported for all databases is
- read/write operations for JSON fields
- querying JSON paths

In the examples below, data field is supposed to be of `Json` type.

### SQLite

You can benefit from better JSON support in sqlite if you have `json1` extension
installed. Otherwise, JSON values are fetched and parsed in an ineffective way.

### PostgreSQL

As shown above, PostgreSQL supports JSON concatenation and "JSON contains" features

### MySql and Oracle

MySql and Oracle support wildcards in JSON path, meaning "any value".
For example:
```python
select(o for o in MyEntity if o.data['items'][:]['visible'] == True)
```
Here we have used a wildcard for a numeric value of the item index. As you see,
Python empty slice is used for this.

Python `Ellipsis` represents a wildcard for a string JSON key, like this:
```python
select(o.data[...]['title'] for o in MyEntity)
```