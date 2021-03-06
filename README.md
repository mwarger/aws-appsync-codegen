# AWS Appsync GraphQL code generator

[![GitHub license](https://img.shields.io/badge/license-MIT-lightgrey.svg?maxAge=2592000)](https://raw.githubusercontent.com/apollographql/apollo-ios/master/LICENSE) [![npm](https://img.shields.io/npm/v/aws-appsync-codegen.svg)](https://www.npmjs.com/package/apollo-codegen) [![Get on Slack](https://img.shields.io/badge/slack-join-orange.svg)](http://www.apollostack.com/#slack)

This is a tool to generate API code or type annotations based on a GraphQL schema and query documents. This project is based upon [Apollo GraphQL code generator](https://github.com/apollographql/apollo-codegen).

It currently generates Swift code, TypeScript annotations, Flow annotations, and Scala code.

See [Apollo iOS](https://github.com/apollographql/apollo-ios) for details on the mapping from GraphQL results to Swift types, as well as runtime support for executing queries and mutations.

## Usage with AWS AppSync

A complete tutorial can be found in the [AWS AppSync documentation](https://awslabs.github.io/aws-mobile-appsync-sdk-ios/) which is recommended for you to review first.

### Create GraphQL API

If you have never created an AWS AppSync API before please use the [Quickstart Guide](https://docs.aws.amazon.com/appsync/latest/devguide/quickstart.html) and then walk through the [iOS client guide](https://docs.aws.amazon.com/appsync/latest/devguide/building-a-client-app-ios.html).

### Download introspection schema

The code generaton process needs two things:  
1. GraphQL introspection schema
1. GraphQL queries/mutations/subscriptions

You can get the introspection schema in a `schema.json` file from the AWS AppSync console. You can find this in the console by clicking on your API name in the left-hand navigation, scrolling to the bottom, selecting **iOS**, clicking the **Export schema** dropdown, and selecting `schema.json`.

### Write GraphQL queries

Now you can write GraphQL queries and the codegen process will convert these to native Swift types. If you are unfamiliar with writing a GraphQL query please [read through this guide](https://docs.aws.amazon.com/appsync/latest/devguide/quickstart-write-queries.html). Once you have your queries written, save them in a file called `queries.graphql`. For example you might have the following in your `queries.graphql` file:

```
query AllPosts {
   allPosts {
       id
       title
       author
       content
       url
       version
   }
}
```

### Generate Swift types

Now that you have your introspection schema and your GraphQL query, install `aws-appsync-codegen` and run the tool against these two files like so:

```
npm install -g aws-appsync-codegen

aws-appsync-codegen generate queries.graphql --schema schema.json --output API.swift
```

The output will be a Swift class called `API.swift` which you can include in your XCode project to perform a GraphQL query against AWS AppSync. 

### Invoke GraphQL operation from Swift

Now that you have completed the code generation, import the `API.swift` file into your XCode project. Then update your project's `Podfile` with a dependency of the AWS AppSync SDK:

```
target 'PostsApp' do
  use_frameworks!
  pod 'AWSAppSync' ~> '2.6.7'
end
```

Next, in any code you wish to run the GraphQL query against AWS AppSync, import the SDK:

```
import AWSAppSync
```

Finally, run your query:

```
        appSyncClient?.fetch(query: AllPostsQuery())  { (result, error) in
            if error != nil {
                print(error?.localizedDescription ?? "")
                return
            }
            self.postList = result?.data?.allPosts
        }
```

**Note:** The code generation process converted the GraphQL statement of `allPosts` in your `queries.graphql` file to `AllPostsQuery()` which allowed you to invoke this using `appSyncClient?.fetch()`. A similar process happens for mutations and subscriptions.

### Automate code generation

The process defined above outlines the general flow, however you can [automate this in your XCode build process](https://docs.aws.amazon.com/appsync/latest/devguide/building-a-client-app-ios.html#building-a-client-app-integrating-into-the-build-process).

## General Usage

If you want to experiment with the tool, you can install the `aws-appsync-codegen` command globally:

```sh
npm install -g aws-appsync-codegen
```

### `introspect-schema`

The purpose of this command is to create a JSON introspection dump file for a given graphql schema. The input schema can be fetched from a remote graphql server or from a local file. The resulting JSON introspection dump file is needed as input to the [generate](#generate) command.

To download a GraphQL schema by sending an introspection query to a server:

```sh
aws-appsync-codegen introspect-schema http://localhost:8080/graphql --output schema.json
```

You can use the `header` option to add additional HTTP headers to the request. For example, to include an authentication token, use `--header "Authorization: Bearer <token>"`.

You can use the `insecure` option to ignore any SSL errors (for example if the server is running with self-signed certificate).

**Note:** The command for downloading an introspection query was named `download-schema` but it was renamed to `introspect-schema` in order to have a single command for introspecting local or remote schemas. The old name `download-schema` is still available is an alias for backward compatibility.

To generate a GraphQL schema introspection JSON from a local GraphQL schema:

```sh
aws-appsync-codegen introspect-schema schema.graphql --output schema.json
```

### `generate`

The purpose of this command is to generate types for query and mutation operations made against the schema (it will not generate types for the schema itself).

#### Swift

This tool will generate Swift code by default from a set of query definitions in `.graphql` files:

```sh
aws-appsync-codegen generate **/*.graphql --schema schema.json --output API.swift
```

The `--add-s3-wrapper` option can be specified to add in S3 wrapper code to the generated source.

#### TypeScript, Flow, or Scala

You can also generate type annotations for TypeScript, Flow, or Scala using the `--target` option:

```sh
# TypeScript
aws-appsync-codegen generate **/*.graphql --schema schema.json --target typescript --output operation-result-types.ts
# Flow
aws-appsync-codegen generate **/*.graphql --schema schema.json --target flow --output operation-result-types.flow.js
# Scala
aws-appsync-codegen generate **/*.graphql --schema schema.json --target scala --output operation-result-types.scala
```

#### `gql` template support

If the source file for generation is a javascript or typescript file, the codegen will try to extrapolate the queries inside the [gql tag](https://github.com/apollographql/graphql-tag) templates.

The tag name is configurable using the CLI `--tag-name` option.

#### [.graphqlconfig](https://github.com/graphcool/graphql-config) support

Instead of using the `--schema` option to point out you GraphQL schema, you can specify it in a `.graphqlconfig` file.

In case you specify multiple schemas in your `.graphqlconfig` file, choose which one to pick by using the `--project-name` option.

## Typescript and Flow

When using `aws-appsync-codegen` with Typescript or Flow, make sure to add the `__typename` introspection field to every selection set within your graphql operations.

If you're using a client like `apollo-client` that does this automatically for your GraphQL operations, pass in the `--addTypename` option to `aws-appsync-codegen` to make sure the generated Typescript and Flow types have the `__typename` field as well. This is required to ensure proper type generation support for `GraphQLUnionType` and `GraphQLInterfaceType` fields.

### Why is the __typename field required?

Using the type information from the GraphQL schema, we can infer the possible types for fields. However, in the case of a `GraphQLUnionType` or `GraphQLInterfaceType`, there are multiple types that are possible for that field. This is best modeled using a disjoint union with the `__typename`
as the discriminant.

For example, given a schema:
```graphql
...

interface Character {
  name: String!
}

type Human implements Character {
  homePlanet: String
}

type Droid implements Character {
  primaryFunction: String
}

...
```

Whenever a field of type `Character` is encountered, it could be either a Human or Droid. Human and Droid objects
will have a different set of fields. Within your application code, when interacting with a `Character` you'll want to make sure to handle both of these cases.

Given this query:

```graphql
query Characters {
  characters(episode: NEW_HOPE) {
    name

    ... on Human {
      homePlanet
    }

    ... on Droid {
      primaryFunction
    }
  }
}
```

Apollo Codegen will generate a union type for Character.

```javascript
export type CharactersQuery = {
  characters: Array<{
    __typename: 'Human',
    name: string,
    homePlanet: ?string
  } | {
    __typename: 'Droid',
    name: string,
    primaryFunction: ?string
  }>
}
```

This type can then be used as follows to ensure that all possible types are handled:

```javascript
function CharacterFigures({ characters }: CharactersQuery) {
  return characters.map(character => {
    switch(character.__typename) {
      case "Human":
        return <HumanFigure homePlanet={character.homePlanet} name={character.name} />
      case "Droid":
        return <DroidFigure primaryFunction={character.primaryFunction} name={character.name} />
    }
  });
}
```

## Contributing

Running tests locally:

```
npm install
npm test
```
