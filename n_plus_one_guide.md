# Guide to Redundant/N+1 SQL queries

## Definition

A **redundant query** is a query that is not necessary to fetch the data you need. It's a query that could be avoided by using data that is already in memory. It's extremely common in application servers. We'll work with django as an example here. 

**N+1 queries** are a type of redundant query that occurs when your application fetches data in a list independent queries that could have been fetched in a single query. For instance, making one query for a list of N books and N more queries to fetch each book's author, rather than just fetching books and their authors in one or two queries. 


## Non-N+1 Redundant query examples

The main source of redundant queries is using an ORM to fetch data you've already fetched. Django's ORM syntax is often more readable than dealing with python lists, so it's easy to fall into this trap. Take the following example:

```python
all_records = Record.objects.all()
first_record = all_records.first() # equivalently, all_records[0]
```

Some might think the first line will perform a query and the second won't, but it's actually the opposite. Querysets are "lazy", so the first line won't perform any query. The second line will perform a `LIMIT 1` query to only fetch the first record. If you need all records and also want to single out the first, you should use python list logic instead:

```python
queryset = Record.objects.all() # no queries, lazy! 
all_records_list = list(queryset) # one query 
first_record = all_records_list[0] # no new query
```

Using the ORM to filter a queryset you've already fetched is also a common source of redundant queries. Django's queryset filtering requires the database todo the filtering, it doesn't actually know how to turn filter expressions into python logic. 

```python
qs = Record.objects.all()
all_records_list = list(qs) # one query

# wrong approach: filter always returns a new queryset and fires a query
special_records_list = list(qs.filter(special=True)) 
# good approach: use python list logic
special_records_list = [ r for r in all_records_list if r.special ]
```

Other examples of common redundant queries are counting or checking if a queryset is empty:

```python
qs = Record.objects.all()
all_records_list = list(qs) # one query

record_count = qs.count() # bad, count always fires off a query
record_count = len(all_records_list) # good, no new query

if qs: # bad! this will fire off a query
    ...

is_empty = qs.exists() # bad, exists always fires off a query
is_empty = len(all_records_list) == 0 # good, no new query
```

You shouldn't avoid `count()` or `exists()` altogether, they are more efficient than fetching many records and counting them, but if you need to fetch the records anyway, you should just use `len()` instead.

The general advice here is to use basic python list logic to manipulate and traverse data you already have loaded in memory. In general, querysets are not caching any function calls. 

At the end of the day these kind of mistakes aren't usually a big deal. Having a view fetch a bunch of records and fire off an extra couple queries to find the first record or count the number of records is not going to be a big performance hit. In [big-O notation](https://en.wikipedia.org/wiki/Big_O_notation), these are constant, so we're only adding O(1) queries. What _is_ a big deal is making queries during iteration (e.g. loops). These are called N+1 queries and they are so slow that django added tools to avoid them. 

## Related data, caching and eager-loading

Every foreign key has a source and a destination. Let's refer to the source as the child and the destination as the parent, e.g. a book's parent is its author, and the author's children are their books.

The ORM makes accessing parents very easy, it's just an attribute lookup, you don't even have to call a method. These attribute lookups will perform a query, unless they're already cached

```python
child = Children.objects.first() # a query
child.parent # another query
child.parent # no query, cached! 
```

Of course, this doesn't help much if you're iterating a bunch of chilren. Even if they all have the same parent, you'll still be firing off a query for each child because the caching is per-instance. Django doesn't globally keep track of which IDs it already fetched.

```python
for child in Children.objects.all(): # one query
    print(child.parent) # one query per loop iteration. N+1 problem! 
```

You can prefetch (and cache) all of the parents by using `select_related`. These methods will perform a single query with a SQL join to fetch all of the parents. 

```python
# one query (SELECT * FROM children JOIN parent ON children.parent_id = parent.id) 
child = Children.objects.select_related('parent').first() 
child.parent # no query, cache is pre-populated by the join

for child in Children.objects.select_related('parent').all(): # one query
    print(child.parent) # no extra queries in the loop

```

Fetching parents is easy, fetching children is a tiny bit more complicated. Accessing children is not just an attribute lookup, it's usually done by using the related manager, via the `related_name` property of the foreign key field. Accessing children has all of the problems of accessing parents, but it's worse because the ORM doesn't cache the related manager calls. 

```python
parent = Parent.objects.first() # one query
list(parent.children.all()) # another query
list(parent.children.all()) # duplicate query, children relationships are not cached

for parent of Parent.objects.all(): # one query
    print(list(parent.children.all())) # one query per loop iteration. N+1 problem! 
```

Fortunately you can prefetch and cache all of the children by using `prefetch_related`. These methods will perform a single extra query to fetch all of the children and cache them. 

```python
parent = Parent.objects.prefetch_related('children').first() # two queries 
list(parent.children.all()) # no query, it's in the prefetch cache
list(parent.children.all()) # no query, same reason

for parent of Parent.objects.prefetch_related('children'): # two queries
    print(list(parent.children.all())) # no additional query

```

What's the difference between `select_related` and `prefetch_related`? Well, `select_related` performs an SQL join, and so only work on parent-relationships. prefetch_related performs a separate query with a big `IN` clause, and so can work on child-relationships (and many-to-many relationships). Prefetch _also_ works on parent relationships, so you can use in both cases. 

If you want to prefetch _filtered_ children for each parent, you can use special `Prefetch` objects. Warning: these can surprise you!

```python
prefetch_obj = Prefetch('children', queryset=Children.objects.filter(special=True))
for parent of Parent.objects.prefetch_related(prefetch_obj): # two queries
    print(list(parent.children.filter(special=True))) # additional query! 


for parent of Parent.objects.prefetch_related(prefetch_obj): # two queries
    print(list(parent.children.all())) # no additional query, the all() actually returns prefetched filtered children! 
```

Above, we see that the `.all()` method actually returns the prefetched filtered children, and the filter() call isn't cached and fires off a query. Both these cases are surprising.  

If you really need to filter, annotate or otherwise mess with prefetch-query, I recommend using the `to_attr` keyword so the filtered children get stored as an explicit list,

```python
prefetch_obj = Prefetch('children', queryset=Children.objects.filter(special=True), to_attr='special_children')
for parent of Parent.objects.prefetch_related(prefetch_obj): # two queries
    print(parent.special_children) # no additional query
```

And if you _really_ need to, you can get even more advanced and compose `Prefetch` and custom attributes to traverse multiple levels of relationships. 

```python
children_prefetch = Prefetch('children', queryset=Children.objects.filter(special=True), to_attr='special_children')
grandchildren_prefetch = Prefetch('special_children__children', queryset=GrandChildren.objects.filter(special=True), to_attr='special_grandchildren')

for parent of Parent.objects.prefetch_related(children_prefetch, grandchildren_prefetch): # three queries
    print(parent.special_children) # no additional query
    print(parent.special_children[0].special_grandchildren) # no additional query
```

`prefetch_related` is powerful, but it can get confusing quickly. When your views and templates start relying on these loosely defined custom `to_attr` attributes, you start to couple your view/presentation logic against the fetching. 



## Level two: Storing data in context 

To prevent redundant queries, it's common to assemble some temporary data-structures (plain old dicts and lists) in your views, so you can process them before sending off data for rendering. This often involves iterating over querysets, indexing records by ID in a dictionary. 

This works well if you're doing everything in one function, but these functions tend to grow big, and you'll find yourself repeating the same logic in different views. How can you split it up without repeating queries?  Your view could just do all the fetching and pass this data to the template and any helpers, but then you'll couple your view logic to your template and helper logic.

One neat solution I like is memoizing query functions that use the request object as a cache. I've created django-data-fetcher for this. 

```python
# query_helpers.py

from data_fetcher import cache_within_request

@cache_within_request
def get_product_types_by_id():
    products = ProductType.objects.all()
    products_by_id = {p.id: p for p in products}
    return products_by_id

```

Now a view can call `get_products_types_by_id` if it needs a product, and utilities and template helpers can also call this function without repeating the queries. This is especially useful for "lookup" data, like product types, categories, tags. 

This decorator also supports arguments, so long as they're valid dictionary keys (i.e. immutable), as they'll be used to index the cache. 


```python
@cache_within_request
def get_products_for_order(order_id:int):
    products = Product.objects.filter(order_id=order_id)
    return list(products)
```


Memoizing functions like this is a great way to avoid redundant queries while keeping your code clean, but it's awkward for batching logic. Function values are cached based on their arguments, so you can't pass them a list of IDs. You could pass a set of IDs, but if consumers want to fetch data for individual IDs, you have to manually populate a cached dictionary somewhere. There is a cleaner way, though, using data-fetchers.

## Level three: Request-scoped data fetchers (advanced)

When batching using request-cached functions gets messy, data-fetcher classes can help. However, they are a bit complicated. 

The idea is to split up data into resource units with unique IDs. These resources can be lists, single records, or anything. Consumers like the view, template, helpers, etc. request these resources via unique identifiers. The view can optionally pre-populate data-fetchers so that downstream consumers don't create extra queries. 

To use datafetchers, we subclass the `DataFetcher` class and define a `batch_load_dict` method. 

```python
from data_fetcher import DataFetcher

class ProductByIdFetcher(DataFetcher):
    def batch_load_dict(self,ids):
        products = Product.objects.filter(id__in=ids)
        return {p.id: p for p in products}
```

The `batch_load_dict` takes a list of unique identifiers and must return a dict of resources indexed by these identifiers. In practice, get-model-by-id is common enough to have shortcut subclasses. 

Datafetchers have a simple public API:

- `get(id)`: fetch a single unit
- `get_many(ids)`: fetch multiple units
- `prefetch_keys(ids)`: prefetch units for sake of performance

An instance caches data on its own local attributes, so we need to make sure we're using the same instance during the request. This is why we never construct instances directly, but instead use `DataFetcherClass.get_instance()`. This uses the same request-caching mechanism as `@cache_within_request` to use previously constructed instances. 

Request-cached-functions and datafetchers can be used together, and datafetchers can call each other too.

The advantage of datafetchers is that they can be used to fetch in bulk _or_ individually. They even have a `prime(key,value)` method that can be used to populate their caches with data. If you're feeling especially stingy with queries, you can have your `ChildrenByParentFetcher` populate a `ChildFetcher` with the children it fetches, and then your templates and views can just use fetchers everywhere without worrying about repeating queries.  

Bulk logic can often be complex, so it's recommended to unit-test data fetchers. 

## An example with various levels of optimization

**Level zero: N+1 queries**

Let's start with an example. We want to display a list of parents, but we want to highlight the ones that have `special=True` children. The dumb 101 way to do this would be with N+1 queries:

```jinja
{# template.jinja2 #}
<ul>
{% for parent in object_list %}
    <li>
        {{ parent.name }}
        {% if parent.children.filter(special=True).exists() %}
            <span class="badge">Has special children!</span>
        {% endif %}
    </li>
{% endfor %}
</ul>
```
(We're using jinja here because it allows for us to call methods, with django templates, you can do something analogous if you add helpers or methods to your models)

Obviously, this is going to create N+1 queries. So we can optimize this by prefetching the children in the view. 

**Level one: Using django's eager-loading tools**

Because we're using a filtered relationship, we can't just use a plain prefetch. We need to use a `Prefetch` object to filter the children and store it on a custom attribute, then change which attribute we use in the template.

```python
# view.py

def parent_list(request):
    parents = Parent.objects.prefetch_related(Prefetch('children', queryset=Children.objects.filter(special=True), to_attr="special_children")).all()
    return render(request, 'parent_list.html', {'object_list': parents})
```

```jinja
{# template.jinja2 #}
{% for parent in object_list %}
    ...
        {% if parent.special_children %}
    ...
```

This solution will only issue 2 queries, but it's rather brittle. To copy this behavior in another view/template, we need to copy the logic from both the view and template. 
 
**Level two: Memoizing query logic with the request-cache decorator**

There are many ways to do this. One is quite easy, it just involves writing to a dictionary. We're essentially using a singleton dict here. Since a dict is mutable, we can just have other functions write to it. 

```python

# helpers.py
@cache_within_request
def _special_parent_by_children_dict(cls):
    # underscored for 'privacy', probably shouldn't be accessed too broadly
    return {}

def get_bulk_special_children_by_parent(parent_ids):
    cache_dict = _parent_by_children_dict()
    missing_ids = [id for id in parent_ids if id not in cache_dict]
    if missing_ids:
        children = Children.objects.filter(special=True,parent_id__in=missing_ids)
        for child in children:
            cache_dict.setdefault(child.parent_id, []).append(child)

    return {id: cache_dict[id] for id in parent_ids}

def get_special_children_for_parent(parent_id):
    cache_dict = _parent_by_children_dict()

    if parent_id not in cache_dict:
        children = Children.objects.filter(special=True, parent_id=parent_id)
        cache_dict[parent_id] = list(children)

    return cache_dict[parent_id]

# views.py

def parents_list(request):
    parents = Parent.objects.all()
    get_bulk_special_children_by_parent([p.id for p in parents])
    return render(request, 'parent_list.html', {'object_list': parents})

```

```jinja
{# template.jinja2 #}
{% for parent in object_list %}
    ...
    {# how you access helpers depends on your template configuration #}
    {% if helpers.get_special_children_for_parent(parent.id) %}
    ...
```

This is _not_ the intended use of `@cache_within_request`, and it's a code-smell, but there's no other easy way of batching queries using a memoize decorator. If you really liked this approach and wanted to clean this up, you could group these 3 functions into a class, the decorator will work fine on static/class methods, just make sure you don't store any data on the class or instance, it might leak memory across requests. It's also painful to test this, there are many different ways to call the functions (bulk then single, single then bulk), you should test all of them. 


**Level three: Using datafetchers**

Although there are shortcut subclasses that make it even easier, let's write out a data-fetcher class:

```python
# helpers.py

class SpecialChildrenByParentIdFetcher(Datafetcher):
    @staticmethod
    def batch_load_dict(parent_ids):
        children = Children.objects.filter(special=True, parent_id__in=parent_ids)
        children_by_parent_id = defaultdict(list)
        for child in children:
            children_by_parent_id[child.parent_id].append(child)
        return children_by_parent_id

```

```python
# views.py

def parents_list(request):
    parents = Parent.objects.all()
    fetcher = SpecialChildrenByParentIdFetcher.get_instance()
    fetcher.prefetch_keys([p.id for p in parents])
    return render(request, 'parent_list.html', {'object_list': parents})
```

```jinja
{# template.jinja2 #}
{% for parent in object_list %}
    ...
    {% if helpers.SpecialChildrenByParentIdFetcher.get_instance().get(parent.id) %}
    ...

```

Because the base DataFetcher class is thoroughly tested, you need only test that your batch_load_dict method works as expected.
