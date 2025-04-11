# How to make RESTful APIs flexible and sensible to work with
I have been an advocate for [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) APIs since I first came across the concept in 2010, and although it took me a while to wrap my head around the concept, I knew from the first moment that this was something that made sense to me compared to the SOAP based APIs that I had previously been working with. Though initially starting out with RESTful APIs in their purest form (so to speak), I have discovered a couple tricks which make them a lot easier to work with, and in this post I’ll share these with you.

## Fundamentals of RESTful APIs
First of all, let’s make sure that we have the same basic understanding of the concept of a RESTful API. It’s an API which uses:

- A base URI, e.g. `https://path.to.api`.
- Standard HTTP methods, i.e. `POST`; `GET`; `PUT`; `PATCH`; and `DELETE` to request and manipulate data.
- URLs which make it (potentially) easy for a human to interpret which information is being requested, e.g. `GET https://path.to.api/v1/users/1`, i.e. get information about the user with `ID` 1.
-Standard [HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) as high-level responses to requests, e.g. `200 OK`; `201 Created`; `401 Unauthorized`; `403 Forbidden` etc. which makes the overall status of a request easy to interpret.
- An internet media type for data, e.g. `JSON`.
- Autonomous requests from a client (API consumer) which contain all of the information required for the API to process the request, i.e. the API doesn’t need to retain the state of the client.

The common denominator here is: Standards, i.e. a RESTful API is built using already existing standards, rather than having to invent public classes which are specific to a said project.

### Response types
With a RESTful API you have the following types of responses:

1. <strong>Resource</strong>: A resource is an extensive object containing all information regarding the resource, e.g. `GET https://path.to.api/v1/users/1` should result in a response containing all available information about the user with `ID` 1.
1. <strong>Collection</strong>: A collection is a list of a specific kind of resource, e.g. `GET https://path.to.api/v1/users` should result in a response containing a list references to all available user resources.

Basically, what this means is that is that if you wanted information about all of the users in the system (just to stick with the example), you would first:

```
GET https://path.to.api/v1/users

{
    items: [
        "https://path.to.api/v1/users/1",
        "https://path.to.api/v1/users/2",
    ]
}
```

… to get the entire array/list (collection) of users (in this case wrapped in a property named “items”), and then get the full details for each user (resource) in the collection as per the following requests:

```
GET https://path.to.api/v1/users/1

{
    id: 1,
    name: "Firstname Lastname"
}
```
```
GET https://path.to.api/v1/users/2

{
    id: 2,
    name: "Givenname Surname"
}
```

… to get the information regarding each resource.

## Improvements
As REST is a conceptual approach to building an API, each development team has to take it from here and build something which suits their needs. In the remainder of this post, I will be examining various approaches which I have found useful in the projects which I have worked on.

### Basic versioning
As you may have noticed the examples above all contain `v1` as part of the request URL, i.e. an abbreviation for “version 1”. The reason behind including this is simply to avoid breaking functionality in the case that your team decides to build a new API from scratch, e.g.:

- `GET https://path.to.api/v1/...`: Get a resource from the old API.
- `GET https://path.to.api/v2/...`: Get a resource from the new API.

At some point you should probably get rid of the `v1` API for maintainability reasons, but by using versioning you can build the new version alongside the old without constantly having to stress over having to make sure that the consumers of the API, e.g. external users, are in sync with your progress.

### Flexible collections (hyper-collections/hypercollections)
Instead of merely having collections being a list of resources, I suggest structuring the response into an object, which includes meta-data about the collection and where most of the meta-data properties can be given values as part of a request thus making the collection more flexible to work with, i.e. (numbered for convenience):

1. <strong>Items</strong>: An array of resource objects.
1. <strong>Sort</strong>: The property name in the individual resource objects by which the whole collection of data is sorted, e.g. `{sort: 'name'}` or `{sort: 'id'}`, both before being sent as a response from the API and as sorted in the `items` array.
1. <strong>Reverse</strong>: A property that lets you reverse the direction by which the whole collection of data is ordered, i.e. `{reverse: true}`, before being sent as a response from the API. In my opinion, the default should be ascending, and thus applying the `true` value to the property should return the collection in descending order.
1. <strong>Limit</strong>: The maximum number of resource object in the `items` array, in case you only wish to include a subset of the whole collection as part of the `items` array, e.g. `{limit: 25}` will include a maximum of 25 resource objects regardless of the length of the whole collection.
1. <strong>Offset</strong>: The offset in the whole collection of data from which the `limit` is calculated. When used together with `limit`, it makes it possible to incorporate paging of the response data.
1. <strong>Previous</strong>: A link to the previous element in the collection (if applicable).
1. <strong>Next</strong>: A link to the next element in the collection (if applicable).
1. <strong>Total</strong>: The total number of resource objects for the request regardless of how many are included in the `items` array, so that you always know the size of the total data-set.
1. <strong>Details</strong>: The detail level of each resource object in the `items` array, e.g. specified by a string, for instance `all`. A common approach in REST is to have each resource in a collection contain a minimum of data, e.g. a URL to where the complete set of data for the resource can be acquired, but I have found that sometimes it makes more sense to get all information regarding the resources as part of the collection instead of as multiple subsequent requests.

The following request:
```
GET https://path.to.api/v1/users?sort=name&reverse=true&limit=10&offset=50&details=all
```

… should thus result in the following response (please note that the individual resources in the `items` array have been left out):

```
{
    items: [],
    sort: "name",
    reverse: true,
    limit: 10,
    offset: 50,
    previous: "/v1/path.to.previous",
    next: "/v1/path.to.next",
    total: 200,
    details: "all"
}
```

#### Examples
Get all users sorted by name:
`GET https://path.to.api/v1/users?sort=name`

Get all users sorted by name in reverse order:
`GET https://path.to.api/v1/users?sort=name&reverse=true`

Get the 25 first users:
`GET https://path.to.api/v1/users?limit=25`

Get all users except the first 25:
`GET https://path.to.api/v1/users?offset=25`

Get users 25-49:
`GET https://path.to.api/v1/users?limit=25&offset=25`

Get the total number of users without any additional information for each resource:
`GET https://path.to.api/v1/users?limit=0`

## Other considerations
### Case-sensitivity
After years of working with our own RESTful API as well as experiencing integration work with other RESTful APIs, I highly advocate making RESTful API URLs (and properties/values in request payloads) insensitive to casing. In my opinion (speaking from experience), case-sensitive URLs adds an unnecessary layer of complexity, which developers both in your own organization as well as developers from other organizations integrating with your RESTful API, will very likely clash with at some point. In my opinion, if you want others to integrate their software solutions with your own, you should aim for making this task as easy as possible for them, i.e. by removing any potential (unnecessary) obstacles.

Having case-insensitive URLs also enables other (or “all” for that matter) developers to use their preferred type of casing, i.e. the one which makes the most sense to them, without it influencing the response they get from your RESTful API.

### Paged collections
A discussion which I have experienced arises again and again, is on the topic of how to handle large collections, specifically on whether to page them implicitly, or whether to serve the entire collection unless instructed otherwise. My stance on this is that implicitly paging a collection is an unnecessary obstacle and should be avoided. Of course you can encourage paging, but if I ask for a collection of resources, I expect to get the full collection, unless I have asked for a segment of it, i.e. via using the `limit` and `offset` properties presented above. However, everything is relative, and if serving an entire collection results in a very large network response, then paging by default probably makes sense.

> #### Denial of service (DOS)
> If using paging on collections, it's a good idea to prevent "denial of service" (DOS) settings from preventing users from traversing a collection via the built-in paging mechanism.

### Creating new resources
When creating a new resource, e.g.

```
POST https://path.to.api/v1/users

{
    name: "Firstname Surname"
}
```

... including the resource in the `201 Created` response from the API makes, e.g.

```
{
    id: 3,
    name: "Firstname Surname",
    href: "https://path.to.api/v1/users/3",
    ..
    .
}
```

... makes the API so much easier to work with, as you don't (necessarily) have to refresh the collection, `users` in this case and subsequently request the specific resource, in order use the new resource.
