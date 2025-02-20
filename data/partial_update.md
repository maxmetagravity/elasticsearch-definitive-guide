# 更新文档中的一部分

In <<update-doc>>, we said that the way to update a document is to retrieve
it, change it, then reindex the whole document. This is true. However, using
the `update` API, we can make partial updates like incrementing a counter in a
single request.

We also said that documents are immutable -- they cannot be changed, only
replaced.  The `update` API *must* obey the same rules.  Externally, it
appears as though we are partially updating a document in place. Internally,
however, the `update` API simply manages the same _retrieve-change-reindex_
process that we have already described. The difference is that this process
happens within a shard, thus avoiding the network overhead of multiple
requests. By reducing the time between the _retrieve_ and _reindex_ steps, we
also reduce the likelihood of there being conflicting changes from other
processes.

The simplest form of the `update` request accepts a partial document as the
`"doc"` parameter which just gets merged with the existing document -- objects
are merged together, existing scalar fields are overwritten and new fields are
added. For instance, we could add a `tags` field and a `views` field to our
blog post with:

```js
POST /website/blog/1/_update
{
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
···

If the request succeeds, we see a response similar to that
of the `index` request:

```js
{
   "_index" :   "website",
   "_id" :      "1",
   "_type" :    "blog",
   "_version" : 3
}
```

Retrieving the document shows the updated `_source` field:

```js
{
   "_index":    "website",
   "_type":     "blog",
   "_id":       "1",
   "_version":  3,
   "found":     true,
   "_source": {
      "title":  "My first blog entry",
      "text":   "Starting to get the hang of this...",
      "tags": [ "testing" ], <1>
      "views":  0 <1>
   }
}
```

1. Our new fields have been added to the `_source`.

#### Using scripts to make partial updates

****

We will discuss scripting in more detail in <<scripting>>, but for now it is
enough to know that scripts can be used in several places in Elasticsearch to
achieve custom actions that are not directly supported by the API. The
default scripting language is called MVEL, but Elasticsearch also supports
JavaScript, Groovy and Python.

MVEL is a simple fast Java-based dynamic scripting language with a syntax
similar to Javascript. You can read more about MVEL in the
http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-scripting.html[Elasticsearch scripting docs]
and on the http://mvel.codehaus.org/Getting+Started+for+2.0[MVEL website].

****

Scripts can be used in the `update` API to change the contents of the `_source`
field, which is referred to inside an update script as `ctx._source`. For
instance, we could use a script to increment the number of `views` that our
blog post has had:

```js
POST /website/blog/1/_update
{
   "script" : "ctx._source.views+=1"
}
```


We can also use a script to add a new tag to the `tags` array.  In this
example we specify the new tag as a parameter rather than hard coding it in
the script itself. This allows Elasticsearch to reuse the script in the
future, without having to compile a new script every time we want to add
another tag:

```js
POST /website/blog/1/_update
{
   "script" : "ctx._source.tags+=new_tag",
   "params" : {
      "new_tag" : "search"
   }
}
```


Fetching the document shows the effect of the last two requests:

```js
{
   "_index":    "website",
   "_type":     "blog",
   "_id":       "1",
   "_version":  5,
   "found":     true,
   "_source": {
      "title":  "My first blog entry",
      "text":   "Starting to get the hang of this...",
      "tags":  ["testing", "search"], <1>
      "views":  1 <2>
   }
}
```
1. The `search` tag has been appended to the `tags` array.
2. The `views` field has been incremented.

We can even choose to delete a document based on it contents,
by setting `ctx.op` to `delete`:

```js
POST /website/blog/1/_update
{
   "script" : "ctx.op = ctx._source.views == count ? 'delete' : 'none'",
    "params" : {
        "count": 1
    }
}
```

### Updating a document which may not yet exist

Imagine that we need to store a pageview counter in Elasticsearch. Every time
that a user views a page, we increment the counter for that page.  But if it
is a new page, we can't be sure that the counter already exists. If we try to
update a non-existent document, the update will fail.

In cases like these, we can use the `upsert` parameter to specify the
document that should be created if it doesn't already exist:

```js
POST /website/pageviews/1/_update
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 1
   }
}
```

The first time we run this request, the `upsert` value is indexed as a new
document, which  initializes the `views` field to `1`. On subsequent runs the
document already exists so the `script` update is applied instead,
incrementing the `views` counter.

### Updates and conflicts

In the introduction to this section, we said that the smaller the window between
the _retrieve_ and _reindex_ steps, the smaller the opportunity for
conflicting changes. But it doesn't eliminate the possibility completely. It
is still possible that a request from another process could change the
document before `update` has managed to reindex it.

To avoid losing data, the `update` API retrieves the current `_version`
of the document in the _retrieve_ step, and passes that to the `index` request
during the _reindex_ step.
If another process has changed the document in between _retrieve_ and _reindex_,
then the `_version` number won't match and the update request will fail.

For many uses of partial update, it doesn't matter that a document has been
changed.  For instance, if two processes are both incrementing the page
view counter, it doesn't matter in which order it happens -- if a conflict
occurs, the only thing we need to do is to reattempt the update.

This can be done automatically by setting the `retry_on_conflict` parameter to
the number of times that `update` should retry before failing -- it defaults
to `0`.

```js
POST /website/pageviews/1/_update?retry_on_conflict=5 <1>
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 0
   }
}
```
1. Retry this update 5 times before failing.

This works well for operations like incrementing a counter where the order of
increments does not matter, but there are other situations where the order of
changes *is* important. Like the <<index-doc,`index` API>>, the `update` API
adopts a ``last-write-wins'' approach by default, but it also accepts a
`version` parameter which allows you to use
<<optimistic-concurrency-control,optimistic concurrency control>> to specify
which version of the document you intend to update.

