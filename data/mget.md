# 获取多个文档

As fast as Elasticsearch is, it can be faster still. Combining multiple
requests into one avoids the network overhead of processing each request
individually. If you know that you need to retrieve multiple documents from
Elasticsearch, it is faster to retrieve them all in a single request using the
_multi-get_ or `mget` API, instead of document-by-document.

The `mget` API expects a `docs` array, each element of which specifies the
`_index`, `_type` and `_id` metadata of the document you wish to retrieve. You
can also specify a `_source` parameter if you just want to retrieve one or
more specific fields:

```js
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

The response body also contains a `docs` array which contains a response
per document, in the same order as specified in the request. Each of these
responses is the same response body that we would expect from an individual
<<get-doc,`get` request>>:

```js
{
   "docs" : [
      {
         "_index" :   "website",
         "_id" :      "2",
         "_type" :    "blog",
         "found" :    true,
         "_source" : {
            "text" :  "This is a piece of cake...",
            "title" : "My first external blog entry"
         },
         "_version" : 10
      },
      {
         "_index" :   "website",
         "_id" :      "1",
         "_type" :    "pageviews",
         "found" :    true,
         "_version" : 2,
         "_source" : {
            "views" : 2
         }
      }
   ]
}
```

If the documents you wish to retrieve are all in the same `_index` (and maybe
even of the same `_type`) then you can specify a default `/_index` or a
default `/_index/_type` in the URL.

You can still override these values in the individual requests:

```js
GET /website/blog/_mget
{
   "docs" : [
      { "_id" : 2 },
      { "_type" : "pageviews", "_id" :   1 }
   ]
}
```

In fact, if all the documents have the same `_index` and `_type`, then you
can just pass an array of `ids` instead of the full `docs` array:

```js
GET /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}
```

Note that the second document that we requested doesn't exist. We specified
type `blog`, but the document with ID `1` is of type `pageviews`. This
non-existence is reported in the response body:

```js
{
  "docs" : [
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "2",
      "_version" : 10,
      "found" :    true,
      "_source" : {
        "title":   "My first external blog entry",
        "text":    "This is a piece of cake..."
      }
    },
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "1",
      "found" :    false  <1>
    }
  ]
}
```
1. This document was not found.

The fact that the second document wasn't found didn't affect the retrieval of
the first document. Each doc is retrieved and reported on individually.

NOTE: The HTTP status code for the above request is `200`, even though one
document wasn't found. In fact, it would still be `200` if *none* of the
requested documents were found.  The reason for that is that the `mget`
request itself completed successfully. To determine the success or failure of
the individual documents, you need to check the `found` flag.
