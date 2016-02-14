---
layout: post
title:  "Querying arrays of embedded documents in MongoDB"
date:   2016-02-14 12:30:00
tags: mongodb aggregation aggregation-framework spring-data
description: Querying documents inside arrays of embedded documents in MongoDB (and Spring Data MongoDB)
image: /assets/article_images/2016-02-14-querying-arrays-of-embedded-objects-mongodb/big-city.png
image2: /assets/article_images/2016-02-14-querying-arrays-of-embedded-objects-mongodb/big-city-mobile.png
---

This post explains how to query and fetch **embedded documents** from within an **array field** in a MongoDB collection, using the **Aggregation Framework**.

In order to fully utilize the power of a document-based database like MongoDB, many people model their **one-to-many relationships** with **embedded documents**. To quote the official MongoDB [manual page](https://docs.mongodb.org/v3.0/tutorial/model-embedded-one-to-many-relationships-between-documents/){:rel='nofollow' target='_blank'} on that topic:

>[The example] illustrates the advantage of embedding over referencing if you need to view many data entities in context of another.

The clichéd scenario that the quote above refers to is an entity which has a collection of other entities somehow linked to it, where usually all that information is fetched together.
```
{
   _id: "joe",
   name: "Joe Bookreader"
}

{
   person_id: "joe",
   street: "123 Fake Street",
   city: "Faketown"
}

{
   person_id: "joe",
   street: "1 Some Other Street",
   city: "Boston"
}
```
Here lets assume that our application frequently retrieves the address data when retrieving a person, so multiple queries are needed in order to resolve the references. A more **optimal schema** would be to embed the address data entities in the person data, as in the following document:
{% highlight js %}
{
   name: "Joe Bookreader",
   addresses: [{ street: "123 Fake Street", city: "Faketown" },
               { street: "1 Some Other Street", city: "Boston"}]
}
{% endhighlight %}
Since this is such a common scenario, there's surely a way to query for these embedded objects. And there is, the very simple
{% highlight js %}
personsCollection.find({'addresses.city': 'Boston'})
{% endhighlight %}
query would, in this case, return exactly the whole document from above, matching that one of it's addresses documents has a city field with the value of "Boston".

I was recently facing a rather similarly-sounding task, which however turned out to be a bit more complicated - querying and fetching only the **embedded documents from within an array field**. In this case, for example, such a request would be for "all addresses in Boston". If we take the following documents as our dataset
{% highlight js %}
{
   name: "Joe Bookreader",
   addresses: [{ street: "123 Fake Street", city: "Faketown" },
               { street: "1 Some Other Street", city: "Boston"}]
}

{
    name: "Mike Storywriter",
    addresses: [{ street: "2nd Street", city: "Sofia" },
               { street: "2 The Same Street", city: "Boston"}]
}
{% endhighlight %}
the result for Boston's addresses would be a variant of the following document:
{% highlight js %}
[{ street: "1 Some Other Street", city: "Boston"},
 { street: "2 The Same Street", city: "Boston"}]
{% endhighlight %}

Now that the requirements are cleared up and we know what the expected result is, lets construct the query. For starters we could go back to our simple query, namely `find({'addresses.city': 'Boston'})`. This works as expected (finds both of our person documents), but returns **all array items** in addresses, and not just the ones which match the query. We need a way to further filter these, and since there is not yet a built-in way for that, we could turn to the [Aggregation Framework](https://docs.mongodb.org/manual/core/aggregation-pipeline/){:rel='nofollow' target='_blank'}, available in MongoDB 2.2+. It provides the [$unwind](https://docs.mongodb.org/manual/reference/operator/aggregation/unwind/){:rel='nofollow' target='_blank'} operator can be used to separate our documents array into a stream of documents that can further be operated on. On our dataset, running
{% highlight js %}
personsCollection.aggregate({ $unwind : "$addresses" })
 {% endhighlight %}
would result in the following list of documents:
{% highlight js %}
{
    "name" : "Joe Bookreader",
    "addresses" : {"street": "123 Fake Street", "city": "Faketown"}
}
{
    "name" : "Joe Bookreader",
    "addresses": {"street": "1 Some Other Street", "city": "Boston"}
}
{
    "name" : "Mike Storywriter",
    "addresses" : {"street": "2nd Street", "city": "Sofia"}
}
{
    "name" : "Mike Storywriter",
    "addresses" : {"street": "2 The Same Street", "city": "Boston"}
}
{% endhighlight %}
On that resultset, filtering again with the same simple `{'addresses.city': 'Boston'}` query would leave us only with the desired documents, namely the second and the fourth ones from the list above.

This leaves us with one final task - getting **only the nested documents** from the addresses field, and not the whole person documents. Since we're in using aggregation, we could turn to the [$group](https://docs.mongodb.org/manual/reference/operator/aggregation/group/){:rel='nofollow' target='_blank'} stage of the pipelining process. We want to somehow group all documents into one and then somehow squash all nested address objects in one field. Both of these requirements are met with the following `$group` stage:
{% highlight js %}
{ $group : {'_id': null, 'content': { $addToSet: '$addresses' }}}
{% endhighlight %}
It groups all documents together, as per the manual:

>The _id field is mandatory; however, you can specify an _id value of null to calculate accumulated values for all the input documents as a whole.

and then simply adds all *addresses* documents in a set field named *content*.

All the pieces of the query finally add up to this:
{% highlight js %}
personsCollection.aggregate(
    { $match: {'addresses.city': 'Boston'} },
    { $unwind: '$addresses' },
    { $match: {'addresses.city': 'Boston'} },
    { $group: {'_id': null, 'content': {$addToSet: '$addresses' }}}
)
{% endhighlight %}
When ran on our dataset, this results in almost exactly what we intended to achieve in the beginning:
{% highlight js %}
{
    "_id" : null,
    "content" : [ {"street":"2 The Same Street", "city":"Boston"}, 
                  {"street":"1 Some Other Street", "city":"Boston"}]
}
{% endhighlight %}
&nbsp;

**Bonus: Achieving the using Spring Data MongoDB**
Since this was implemented for a project that's using Spring Data MongoDB, here's the source for the same aggregation query, but in Java.

{% highlight java %}
Aggregation agg = Aggregation.newAggregation(
    match(Criteria.where("addresses.city").is("Boston")),
    unwind("addresses"),
    match(Criteria.where("addresses.city").is("Boston")),
    group().addToSet("addresses").as("content")
);
{% endhighlight %}

From here on the actual execution of the aggregation could be done in many ways, e.g. via `MongoOperations#mongoOperations.aggregate` or by utilizing the `executeCommand` in order to have access to the raw DB objects and perhaps operate on the *content* variable.
