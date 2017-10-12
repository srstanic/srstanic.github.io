---
layout: post
title:  "[firebase] How to paginate and filter sorted lists in the firebase realtime database?"
categories: firebase database
---

Let's say you are building an app for the secret service. One of the requirements is to have a list of agents sorted by
the timezone they are currently operating in. So you might have a list of agent objects in the Firebase Database:

```json
{
  "agents": {
    "-KwFKidh4OKGlt8nak4A": {
      "name": "Robbert Ridger",
      "timezone_gmt_offset": 0,
      "timezone_id": "Africa/Abidjan"
    },
    "-KwFKidms27MFp6xZ6HV": {
      "name": "Tiphany Broader",
      "timezone_gmt_offset": 1,
      "timezone_id": "Africa/Lagos"
    },
    "-KwFKidnpXvgxrVYgO9z": {
      "name": "Eveline Klaffs",
      "timezone_gmt_offset": -3,
      "timezone_id" :"America/Sao_Paulo"
    },
    "-KwFKidp8zEQH7yOT-nI": {
      "name": "Umeko Piche",
      "timezone_gmt_offset": -4,
      "timezone_id" :"America/Toronto"
    },
    ...
  }
}
```

Each agent object has a property named `timezone_gmt_offset` with a signed number value representing the GMT offset.

Out of the box, Firebase Realtime Database offers a couple of ways to sort and paginate object lists. These [two](https://howtofirebase.com/collection-queries-with-firebase-b95a0193745d) [articles](https://howtofirebase.com/firebase-data-structures-pagination-96c16ffdb5ca) cover the basics of querying and pagination pretty well. I suggest you go read them if you are just starting out with the Firebase Realtime Database.

Now, let's assume that for the efficiency reasons we decided to implement the cursor based pagination. Here is how we would go about implementing it.

We would use `orderByChild` to specify the sorting and the `timezone_gmt_offset` property to sort the agents by the timezone they are operating in. To limit the results to a desired page size, we would use `limitToFirst`. When retrieving a first page, we would get N+1 items, where N is the size of the page. For this example, we will make the page size to be 4. That means we get 5 items:

```javascript
firebase.database()
        .ref("/agents")
        .orderByChild('timezone_gmt_offset')
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```

A resulting snapshot might look like the following JSON snippet:

```json
{
  "-KwFKidp8zEQH7yOT-nI": {
    "name": "Umeko Piche",
    "timezone_gmt_offset":-4,
    "timezone_id":"America/Toronto"
  },
  "-KwFKidnpXvgxrVYgO9z": {
    "name": "Eveline Klaffs",
    "timezone_gmt_offset":-3,
    "timezone_id":"America/Sao_Paulo"
  },
  "-KwFKidh4OKGlt8nak4A": {
    "name": "Robbert Ridger",
    "timezone_gmt_offset":0,
    "timezone_id":"Africa/Abidjan"
  },
  "-KwFKidms27MFp6xZ6HV": {
    "name": "Tiphany Broader",
    "timezone_gmt_offset":1,
    "timezone_id":"Africa/Lagos"
  },
  "-KwFKidwW6ZCE7Yebfpj": {
    "name": "Tabbie Borg",
    "timezone_gmt_offset":2,
    "timezone_id":"Europe/Bratislava"
  }
}
```

Notice that the items are sorted numerically from the smallest to the largest number. When handling the snapshot though, you need to be aware that you have received a JSON object and if you try to iterate over its keys, thew will not be sorted as you would expect. Here is the code that will maintain the proper sorting:

```javascript
snapshot.forEach(function(child) {
    var agent_id = child.key
    var agent = child.val()
    // do what you need to do with the agent object, e.g. populate the list view item
});
```

After getting the results, we cut of the last item and present the N items in the view, which is 4 in this example.

```
Umeko Piche (GMT-4)
Eveline Klaffs (GMT-3)
Robbert Ridger (GMT0)
Tiphany Broader (GMT1)
```

When the user initiates loading of another page, we would query for another N+1 items starting with the last item from the previous page. We would use `startAt` in a combination with `limitToFirst` to define the items range for the second page. The limit of the Firebase Database here is that for the `startAt` method you can only use the value of the property specified in the `orderByChild`. The problem we face now is that the value of `timezone_gmt_offset` is not unique across the agents. And we can't use a value that multiple items share to specify the starting item of the next page.

To address this issue we are going to create a unique property to order by and name it `_sort_timezone_gmt_offset`? We create a custom property made out of timezone_gmt_offset and the id. That will be unique

`_sort_gmt_diff: getCharForTranslatedGmtDiff(timezone.gmt_diff) + "_" + newItemKey,`


```javascript
firebase.database()
        .ref("/agents")
        .orderByChild('_sort_timezone_gmt_offset')
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```
`query code and result`
```json
{
"-KwFKidnpXvgxrVYgO9z":{"_sort_timezone_gmt_offset":"-3_-KwFKidnpXvgxrVYgO9z","name":"Eveline Klaffs","timezone_gmt_offset":-3,"timezone_id":"America/Sao_Paulo"}
"-KwFKidp8zEQH7yOT-nI":{"_sort_timezone_gmt_offset":"-4_-KwFKidp8zEQH7yOT-nI","name":"Umeko Piche","timezone_gmt_offset":-4,"timezone_id":"America/Toronto"}
"-KwFKidh4OKGlt8nak4A":{"_sort_timezone_gmt_offset":"0_-KwFKidh4OKGlt8nak4A","name":"Robbert Ridger","timezone_gmt_offset":0,"timezone_id":"Africa/Abidjan"}
"-KwFKiduM8A7hD-xAeyW":{"_sort_timezone_gmt_offset":"10_-KwFKiduM8A7hD-xAeyW","name":"Johnathan Mandry","timezone_gmt_offset":10,"timezone_id":"Australia/Currie"}
"-KwFKie-IJCovyOTklNg":{"_sort_timezone_gmt_offset":"12_-KwFKie-IJCovyOTklNg","name":"Finn Kleiser","timezone_gmt_offset":12,"timezone_id":"Pacific/Fiji"}
}
```



But now there is something wrong with the sorting. It's not sorted numerically anymore, but alphabetically. And that's not the ordering we want anymore.

The way to go around it is to make the timezone_gmt_offset value into a value that can be sorted alphabetically. Something like this:

```javascript
function getCharForTranslatedGmtDiff(gmt_diff) {
    var translatedGmtDiff = gmt_diff + 12
    return String.fromCharCode(65 + translatedGmtDiff)
}
```

This allows us to sort the agents list by the timezone_gmt_offset and paginate it in the same time.

Now let's also say that the requirements state that the user can search the agents by their name. For the advanced searching capabilities, we would need to use a third party service with a full-text search engine. But assuming the requirements would be satisfied with a search that supports only "starts-with" approach and that the search results do not need to be sorted by the timezone, we can fullfil the requirements by using only the Firebase Realtime Database querying API.

It's not possible to query by multiple property values at the same time. Since the agents are searched by their name, we need to use the name property Instead of by 'timezone_gmt_offset'. That will result with the list being sorted by the name. So how do we filter the results by the entered search query? We query the list to start with the name that matches the query and to end with the name that matches the query appended with an alphabetically high unicode character. I found this technique on [stackoverflow](https://stackoverflow.com/a/40633692/517865) and here is how it looks like:

```javascript
var queryText = "Na"
firebase.database().ref(refPath)
    .orderByChild('_sort_name')
    .startAt(queryText)
    .endAt(queryText+"\uf8ff")
    .limitToFirst(6)
        .once("value").then(print_timezones_by_name)
```

The results are:

```javascript

```









To create a serial queue, normally you would write something simple like this:

`let queue = DispatchQueue(label: “com.mydomain.myqueue")`

If you wanted to define a priority for that queue, you could have done that by [retrieving a global queue of the required priority and setting it as a target](http://stackoverflow.com/a/17690878/517865). But that’s the old way to do it and [it doesn’t work in Swift 3](https://bugs.swift.org/browse/SR-1859).
Since Swift 3, once a dispatch queue is activated, it cannot be mutated anymore. Setting a target on an activated queue will compile but then throw an error in run time.

Fortunately, `DispatchQueue` initializer accepts other arguments besides `label` and one of them is `qos`.

`let queue = DispatchQueue(label: "com.mydomain.myqueue", qos: .background)`

Although [Apple documentation](https://developer.apple.com/reference/dispatch/dispatchqueue) states this initializer is available on iOS 10+, I’ve tested it and it also works on iOS 9.

If for whatever reason, you still need to set the target on an already created queue, you can do that by using the `initiallyInactive` attribute available since iOS 10:

`let queue = DispatchQueue(label: “com.mydomain.myqueue”, attributes: [.initiallyInactive])`

That will allow you to modify it until you activate it.

References
----------
* [http://devstreaming.apple.com/videos/wwdc/2016/720w6g8t9zhd23va0ai/720/720_concurrent_programming_with_gcd_in_swift_3.pdf](http://devstreaming.apple.com/videos/wwdc/2016/720w6g8t9zhd23va0ai/720/720_concurrent_programming_with_gcd_in_swift_3.pdf)
* [https://developer.apple.com/videos/play/wwdc2016/720/](https://developer.apple.com/videos/play/wwdc2016/720/)
