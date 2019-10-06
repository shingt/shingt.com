---
layout: post
title: "GraphQL on iOS using Apollo"
date: 2017-04-02 16:50:35 +0900
comments: true
tags: 
  - Swift
---

NOTE: This post is an English version of http://qiita.com/shingt/items/ed65c654eb5532eeeda8

Recently I tried GraphQL on iOS, and so I put some notes in this post.
Sample codes can be found here: [shingt/GitHub-GraphQL-API-Example-iOS](https://github.com/shingt/GitHub-GraphQL-API-Example-iOS).

<!-- more -->

## What is GraphQL?

GraphQL is a query language (and its specification) that is designed to intuitively and flexibly describe the way clients can fetch necessary data from servers, which was announced by Facebook at React.js Conf in 2015.

* [facebook/graphql](https://github.com/facebook/graphql)
* Blog: [GraphQL Introduction - React Blog](https://facebook.github.io/react/blog/2015/05/01/graphql-introduction)
* Video: [React.js Conf 2015 - Data fetching for React applications at Facebook](https://www.youtube.com/watch?v=9sc8Pyc51uU)

A very easy example can be found in [GraphQL Working Draft](https://facebook.github.io/graphql/). If you describe your query as follows:

```json
{
  user(id: 4) {
    name
  }
}
```

its response can be returned as follows.

```json
{
  {
  "user": {
    "name": "Mark Zuckerberg"
  }
}
```

This query means *"I want to fetch `name` field of `user` which has `4` for `id`."* GraphQL enables clients to specify necessary fields and fetch only those data.

Servers define schemas that represent what kind of data exist, and what types of query can be fetched. So unique type system can be used for each application. Though it's response is represented in JSON, a client can know its format beforehand.

The following points are listed as design philosophy:

* Hierarchical
* Product‐centric
* Strong‐typing: 
* Client‐specified queries
* Introspective

Facebook announced that it's production-ready in September 2016.

> For us at Facebook, GraphQL isn't a new technology. GraphQL has been delivering data to mobile News Feed since 2012. 
> In recognition of the fact that GraphQL is now being used in production by many companies, we're excited to remove the "technical preview" moniker. GraphQL is production ready.

[Leaving technical preview | GraphQL](http://graphql.org/blog/production-ready/)

Facebook used to develop its mobile app in HTML5, as you may remember. They started using GraphQL when they replace it with Objective-C. Or rather, GraphQL itself was developed for its native app, which has been presented in this video.

[Lee Byron - Exploring GraphQL at react-europe 2015](https://www.youtube.com/watch?v=WQLzZf34FJ8)

## Try GraphQL

Let's try real examples. For now, we use [GitHub GraphQL API](https://developer.github.com/early-access/graphql/), which was published last September.

(Note: You need to finish registering Early Access program for developers.)

Say you want to search GitHub repositories with a query `GraphQL`, and want to fetch two results with repository name, path, URL, and number of stars.

```json
{
  search(query: "GraphQL", type: REPOSITORY, first: 2) {
    edges {
      node {
        ... on Repository {
          name
          owner {
            path
          }
          stargazers {
            totalCount
          }
          url
        }
      }
    }
  }
}
```

### Send a request using curl

Try sending this request using `curl`. You need to generate OAuth token which has `repository` in its scope in advance.

```sh
% TOKEN="YOUR_TOKEN"
% curl -H "Authorization: bearer $TOKEN" -X POST -d '
{
  "query": "query { search(query: \"GraphQL\", type: REPOSITORY, first: 2) { edges { node { ... on Repository { name, owner { path } stargazers { totalCount } url } } } } }"
}
' https://api.github.com/graphql | jq .
```

And its response will be:

```json
{
  "data": {
    "search": {
      "edges": [
        {
          "node": {
            "name": "graphql",
            "owner": {
              "path": "/facebook"
            },
            "stargazers": {
              "totalCount": 3993
            },
            "url": "https://github.com/facebook/graphql"
          }
        },
        {
          "node": {
            "name": "graphql",
            "owner": {
              "path": "/graphql-go"
            },
            "stargazers": {
              "totalCount": 855
            },
            "url": "https://github.com/graphql-go/graphql"
          }
        }
      ]
    }
  }
}
```

### Send a request using URLSession

It's time to try our request in Swift. My environment is Xcode 8.1 and Swift 3.0.

As a first step, you can add your query characters into `httpBody` in `URLRequest`, and post it using `URLSession`.

```swift
let token = "YOUR_TOKEN"
let url = URL(string: "https://api.github.com/graphql")!

var request = URLRequest(url: url)
request.httpMethod = "POST"
request.addValue("bearer \(token)", forHTTPHeaderField: "Authorization")

let query = "query { search(query: \"GraphQL\", type: REPOSITORY, first: 2) { edges { node { ... on Repository { name, owner { path } stargazers { totalCount } url } } } } }"
let body = ["query": query]
request.httpBody = try! JSONSerialization.data(withJSONObject: body, options: [])
request.cachePolicy = .reloadIgnoringLocalCacheData // Avoid 412

let task = URLSession.shared.dataTask(with: request, completionHandler: { data, _, error in
    if let error = error { print(error); return }
    guard let data = data else { print("Data is missing."); return }
    do {
        let json = try JSONSerialization.jsonObject(with: data, options: [])
        print(json)
    } catch let e {
        print("Parse error: \(e)")
    }
})
task.resume()
```

I omit its result since it's almost same as a result of last curl example. Although you can get a result, this has following problems

* Handwritten query
* JSON object need to be mapped into a predefined type
* Type system prepared in server-side is not utilized

You can find some libraries to handle GraphQL in Swift. In this post, I'll try [Apollo iOS](https://github.com/apollostack/apollo-ios), which can resolve above problems.

## Apollo iOS

### Apollo

[Apollo](http://www.apollodata.com/) is a data stack based on GraphQL developed by [Meteor](https://www.meteor.com/). They have published many open source libraries that make GraphQL easier to use, including [Apollo iOS](https://github.com/apollostack/apollo-ios). Apollo iOS has following characteristics.

* It **generates Swift codes automatically from GraphQL queries **
* Response can be mapped into Swift type for each query, not for each model described in the schema
* GraphQL query syntax check can be done at compile time on Xcode

You can try a sample project using Apollo below. Note that you need to prepare a node server as well.

* iOS: [apollostack/frontpage-ios-app](https://github.com/apollostack/frontpage-ios-app)
* Server: [apollostack/frontpage-server](https://github.com/apollostack/frontpage-server)

In this post, I create a zero-based project and send a request to GitHub GraphQL API.

### Setup

Basically you can follow [Apollo iOS Guide](http://dev.apollodata.com/ios/). But rough flow is:

* Import `Apollo` using Carthage or Cocoapods
* Install `apollo-codegen`
* Run followings when you build a project
  * Fetch a GraphQL schema from your server and save it as `schema.json`
  * Generate Swift model files based on `schema.json` and `.graphql` files

### Introspection and apollo-codegen

As I mentioned at the beginning, with GraphQL you can define a type system for your application, and your server keeps its schema.

This corresponds the following code in an Apollo sample application.

https://github.com/apollostack/frontpage-server/blob/master/data/schema.js

This type system can be written in GraphQL schema language. You can check it's specification in [Schemas and Types | GraphQL](http://graphql.org/learn/schema/).

In order to utilize this schema info on the client side, GraphQL **[Introspection](http://graphql.org/learn/introspection/)** is useful.

Introspection is a function to ask GraphQL server about what kinds of queries are acceptable.
Introspection itself is represented as GraphQL query. But Apollo provides its tool, [apollo-codegen](https://github.com/apollostack/apollo-codegen).

```sh
apollo-codegen download-schema http://localhost:8080/graphql --output schema.json
```

Of course, GitHub GraphQL API can handle Introspection.

```sh
apollo-codegen download-schema https://api.github.com/graphql --header "Authorization: Bearer $TOKEN" --output schema.json
```

In this case, you get your result in `schema.json`. 

https://github.com/shingt/GitHub-GraphQL-API-Example-iOS/blob/master/GitHub-GraphQL-API-Example-iOS/schema.json

`apollo-codegen` can generate Swift models automatically using `schema.json` and your `.graphql` files. This result will be output in `API.swift`.

```sh
apollo-codegen generate **/*.graphql --schema schema.json --output API.swift
```

if you have already done your setup in the last section, this will be executed automatically in each build.

### Send request to GitHub GraphQL API using Apollo iOS

Now that we are ready, let's send our query using Apollo. First I describe your query in `RepositoriesViewController.graphql`.

(In Apollo, a file name of `.graphql` is preferred to be same as a name of a component which executes a query. Since I'm assuming to run in `RepositoriesViewController`, I named its query `RepositoriesViewController.graphql`.

```json
query SearchRepositories($query: String!, $count: Int!) {
    search(query: $query, type: REPOSITORY, first: $count) {
        edges {
            node {
                ... on Repository {
                    name
                    owner {
                        path
                    }
                    stargazers {
                        totalCount
                    }
                    url
                }
            }
        }
    }
}
```

When you build this project, `API.swift` is generated, in which a model corresponding this query is defined.

```swift
public final class SearchRepositoriesQuery: GraphQLQuery {
  public static let operationDefinition =
    "query SearchRepositories($query: String!, $count: Int!) {" +
    "  search(query: $query, type: REPOSITORY, first: $count) {" +
    "    edges {" +
    "      node {" +
    "        __typename" +
    "        ... on Repository {" +
    "          name" +
    "          owner {" +
    "            __typename" +
    "            path" +
    "          }" +
    "          stargazers {" +
    "            totalCount" +
    "          }" +
    "          url" +
    "        }" +
    "      }" +
    "    }" +
    "  }" +
    "}"

  public let query: String
  public let count: Int

  public init(query: String, count: Int) {
    self.query = query
    self.count = count
  }

  public struct Data: GraphQLMappable {
    public let search: Search

    public init(reader: GraphQLResultReader) throws {
      search = try reader.value(for: Field(responseName: "search"))
    }

    public struct Search: GraphQLMappable {
      public let __typename = "SearchResultItemConnection"
      public let edges: [Edge?]?

      public init(reader: GraphQLResultReader) throws {
        edges = try reader.optionalList(for: Field(responseName: "edges"))
      }

      public struct Edge: GraphQLMappable {
        public let __typename = "SearchResultItemEdge"
        public let node: Node?

        public init(reader: GraphQLResultReader) throws {
          node = try reader.optionalValue(for: Field(responseName: "node"))
        }

        public struct Node: GraphQLMappable {
          public let __typename: String

          public let asRepository: AsRepository?

          public init(reader: GraphQLResultReader) throws {
            __typename = try reader.value(for: Field(responseName: "__typename"))

            asRepository = try AsRepository(reader: reader, ifTypeMatches: __typename)
          }

          public struct AsRepository: GraphQLConditionalFragment {
            public static let possibleTypes = ["Repository"]

            public let __typename = "Repository"
            public let name: String
            public let owner: Owner
            public let stargazers: Stargazer
            public let url: String

            public init(reader: GraphQLResultReader) throws {
              name = try reader.value(for: Field(responseName: "name"))
              owner = try reader.value(for: Field(responseName: "owner"))
              stargazers = try reader.value(for: Field(responseName: "stargazers"))
              url = try reader.value(for: Field(responseName: "url"))
            }

            public struct Owner: GraphQLMappable {
              public let __typename: String
              public let path: String

              public init(reader: GraphQLResultReader) throws {
                __typename = try reader.value(for: Field(responseName: "__typename"))
                path = try reader.value(for: Field(responseName: "path"))
              }
            }

            public struct Stargazer: GraphQLMappable {
              public let __typename = "StargazerConnection"
              public let totalCount: Int

              public init(reader: GraphQLResultReader) throws {
                totalCount = try reader.value(for: Field(responseName: "totalCount"))
              }
            }
          }
        }
      }
    }
  }
}
```

Just as I mentioned, Apollo generates a model for each query, not for each type.

Although you can define nullability when you describe a GraphQL schema, since you do not know whether each field will be included in each request or not, if you want to generate a model for each type, you need to define every parameter as optional. 

Meanwhile, if you generate a model for each query, fields which are included in the query and are defined as non-null in a schema do not need to be defined as optional, resulting in handling it easily. You can find more info in the following article:

[Mapping GraphQL types to Swift](https://dev-blog.apollodata.com/mapping-graphql-types-to-swift-aa85e5693db4#.ks7wdbhjj)

Now let's rewrite my ugly URLSession-version code using a generated query model.
You can send your request by using an instance of `ApolloClient`.

```swift
let url = URL(string: "https://api.github.com/graphql")!
let configuration: URLSessionConfiguration = .default
let apollo = ApolloClient(networkTransport: HTTPNetworkTransport(url: url, configuration: configuration))        
```

In order to post your query, you can use `fetch` method.

```swift
let queryString = "GraphQL"
apollo.fetch(query: SearchRepositoriesQuery(query: queryString, count: 2), completionHandler: { (result, error) in
    // ... 
})
```

So what kind of data can we access in `completionHandler`? If you look into a definition of `fetch` method, you can find followings:

```swift
public func fetch<Query : GraphQLQuery>(query: Query, queue: DispatchQueue = default, completionHandler: @escaping (Apollo.GraphQLResult<Query.Data>?, Error?) -> Swift.Void) -> Cancellable
```

The type of `result` is `Apollo.GraphQLResult<Query.Data>`.
`GraphQLResult` has `data` property, which is `SearchRepositoriesQuery.Data` in `API.swift` in my case.

```swift
apollo.fetch(query: SearchRepositoriesQuery(query: queryString, count: 2), completionHandler: { (result, error) in
    if let error = error { print("Error: \(error)"); return }
    
    result?.data?.search.edges?.forEach { edge in
        guard let repository = edge?.node?.asRepository else { return }
        print("Name: \(repository.name)")
        print("Path: \(repository.url)")
        print("Owner: \(repository.owner.path)")
        print("Stars: \(repository.stargazers.totalCount)")
    }
})
```

You can find we are handling some parameters as non-optional in `completionHandler`.
It seems to be correct since a schema fetched by introspection represents `stargazers` as `NON_NULL`.

```json
{
  "name": "stargazers",
  "description": "A list of users who have starred this repository.",
  "args": [
    ...
  ],
  "type": {
    "kind": "NON_NULL",
    "name": null,
    "ofType": {
      "kind": "OBJECT",
      "name": "StargazerConnection",
      "ofType": null
    }
  },
  "isDeprecated": false,
  "deprecationReason": null
},
```

Previous code prints following results, and you can find our response is correct.

```sh
Name: graphql
Path: https://github.com/facebook/graphql
Owner: /facebook
Stars: 3994


Name: graphql
Path: https://github.com/graphql-go/graphql
Owner: /graphql-go
Stars: 856
```

The whole codes are as follows. Note that you need to set Authorization token just like before.

```swift
let queryString = "GraphQL"

let configuration: URLSessionConfiguration = .default
configuration.httpAdditionalHeaders = ["Authorization": "Bearer \(token)"]
configuration.requestCachePolicy = .reloadIgnoringLocalCacheData // To avoid 412

let url = URL(string: "https://api.github.com/graphql")!
let apollo = ApolloClient(networkTransport: HTTPNetworkTransport(url: url, configuration: configuration))        
apollo.fetch(query: SearchRepositoriesQuery(query: queryString, count: 2), completionHandler: { (result, error) in
    if let error = error { print("Error: \(error)"); return }
    
    result?.data?.search.edges?.forEach { edge in
        guard let repository = edge?.node?.asRepository else { return }
        print("Name: \(repository.name)")
        print("Path: \(repository.url)")
        print("Owner: \(repository.owner.path)")
        print("Stars: \(repository.stargazers.totalCount)")
    }
})
```

#### fragment

GraphQL has "fragment", which is a format to reuse components in your query.
If you want to reuse a code block describing `name` or `url` in your query, you can separate it and define it as a fragment, for instance as `RepositoryDetails`.

```json
query SearchRepositories($query: String!, $count: Int!) {
    search(query: $query, type: REPOSITORY, first: $count) {
        edges {
            node {
                ... on Repository {
                    ...RepositoryDetails
                }
            }
        }
    }
}
```

```json
fragment RepositoryDetails on Repository {
    name
    owner {
        path
    }
    stargazers {
        totalCount
    }
    url
}
```

In addition to this reusing pattern, fragment is helpful when you want to limit your data to only neccesary data.
As for `RepositoriesViewController`, you only need to pass `RepositoryDetails` to its cells.

You can find above codes in the following repository.

[shingt/GitHub-GraphQL-API-Example-iOS](https://github.com/shingt/GitHub-GraphQL-API-Example-iOS)

---

## References

* GraphQL official
  * [GraphQL | A query language for your API](http://graphql.org/)
  * [facebook/graphql](https://github.com/facebook/graphql)
* Apollo
  * [GraphQL Client | Apollo](http://dev.apollodata.com/)
  * [Apollo iOS](https://github.com/apollostack/apollo-ios)
* Youtube
  * [React.js Conf 2015 - Data fetching for React applications at Facebook](https://www.youtube.com/watch?v=9sc8Pyc51uU)
  * [Lee Byron - Exploring GraphQL at react-europe 2015](https://www.youtube.com/watch?v=WQLzZf34FJ8)
  * [Bringing GraphQL to iOS (GraphQL Meetup #2)](https://www.youtube.com/watch?v=7a5b4M9Bjd4)
* Other
  * [GitHub GraphQL API](https://developer.github.com/early-access/graphql/)
  * [GraphQL for iOS Developers - Artsy Engineering](http://artsy.github.io/blog/2016/06/19/graphql-for-mobile/)

