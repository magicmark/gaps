# GAP-0: @mock Specification

## Overview

This proposal defines the `@mock` directive, enabling GraphQL clients to return
mocked data for fields or entire operations. Mock data is stored statically in
JSON files alongside the operations that use them.

## Motivation

Client and backend developers often work in parallel, but clients cannot build
against schema that doesn't yet exist. The `@mock` directive lets client
developers define and use mock responses for fields and types that may not yet
be present in the server schema, unblocking frontend development.

## FAQs

### Conditional response based on list position

This specification does not support conditional use of mock values per array 
position. You may however hoist usage of `@mock` to the parent node that returns
a list and return an array containing different values.

**Example**

Consider the following query:

```graphql
query PetStorePets {
  dogsForSale {
    name @mock(value: "fido")
  }
}
```

This might produce the following response:

```json
{
  "data": {
    "dogsForSale": [
      { "name": "fido" },
      { "name": "fido" },
      { "name": "fido" }
    ]
  }
}
```

For the purposes of a mock user interface, this might be ok! If you wanted to
change the value for each item in the list, you would need to mock at a higher
level:

```graphql
query PetStorePets {
  dogsForSale @mock(variant: "3-dogs-for-sale") {
    name
  }
}
```

#### Future Proposal

We may in future extend this specification to allow for a random pick of
multiple mock values:

```graphql
query PetStorePets {
  dogsForSale {
    name @mock(value: "john") @mock(value: "ringo") @mock(value: "paul")
  }
}
```

This may produce the following response:

```json
{
  "data": {
    "dogsForSale": [
      { "name": "paul" },
      { "name": "ringo" },
      { "name": "paul" }
    ]
  }
}
```

#### Why don't we allow specifying an array position?

<details>

Consider the following query:

```graphql
query GetUserFavorites {
  businessesNearMe {
    menuItems {
      name
      price
    }
  }
}
```

Consider that we wish to mock a new field that does not yet exist on the server;
`blurHash`:

```graphql
query GetUserFavorites {
  businessesNearMe {
    menuItems {
      name
      price
      photo {
        url
        blurHash @mock(value: "LEHV6nWB2yk8pyo0adR*.7kCMdnj")
      }
    }
  }
}
```

Currently, all list items in the user interface will use the same string for
`blurHash` - we have no way to use different values for different positions in
the list.

A tempting option might be to add a "list position argument" that the client
can interpret to merge in the mock value in the right position:

```graphql
query GetUserFavorites {
  businessesNearMe {
    menuItems {
      name
      price
      photo {
        url
        blurHash
          @mock(value: "LEHV6nWB2yk8pyo0adR*.7kCMdnj", nth_child: 0)
          @mock(value: "L6PZfSi_.AyE_3t7t7R**0o#DgR4", nth_child: 1)
      }
    }
  }
}
```

_(Fans of CSS will find
[`nth_child`](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/:nth-child)
familiar.)_

However, there are issues with this API:

##### 1. Parent Ambiguity

Which list parent does `nth-child` apply to? There are two list parents of
`Photo.blurHash`: (1) `Query.businessesNearMe` and (2) `Business.menuItems`.

We would need an even more complicated API to resolve this ambiguity.

##### 2. Order Instability

Results from the server may not be stable over long periods of time.

Consider this example that searches book stores for available books:

```graphql
query BooksInStock {
  findBookInStock(name: "Fly Fishing", author: "JR Hartley") {
    inStock
    storeDetails { name address } @mock(..., nth_child: 0)
  }
}
```

The data the server returns may change over time, either due to:

1. Organic results change
2. The server changes behavior (e.g. a default `orderBy` parameter changes from
   `ASC to `DESC` -- this should not happen by convention, but we cannot rely on
   this.)

This means that the result object that the mocked field is merged into may
change over time - resulting in a different UI state than was intended,
potentially leading to nonsensical or broken UI states.

##### 3. Abstract types 

```graphql
query BooksInStock {
  findBookInStock(name: "Fly Fishing", author: "JR Hartley") {
    inStock
    ... on PhysicalStore {
      storeDetails { name address } @mock(..., nth_child: 3)
    }
    ... on Website {
      websiteDetails { name url } @mock(..., nth_child: 3)
    }
  }
}
```

It is ambiguous if the `nth_child` applies to the concrete or abstract type.
i.e. does `nth_child: 3` means "the third occurrence of PhysicalStore
specifically" or "the third element of `findBookInStock`"?

This is particularly problematic where applied inside a fragment defined in a
separate file, and it's not obvious that the fragment is even called inside of
a list!

And what if that fragment is reused in multiple queries, both as a list and not
as a list? 🤯

##### Summary

Although complex, the issues above are solvable in theory. However given the
complexity, the current version of this specification does not support this.

If you have ideas for how to solve this, please send a pull request for a future
spec version :)

</details>