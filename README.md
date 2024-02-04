Mongo Atlas is the web to create account and do configurations

Mongo Compass is a GUI that can connect to the cloud db after community edition is installed

In the GUI, there is Mongosh to type commands to interact with the database

```
show databases
use sample_airbnb
```

^ similar to SQL

collections are kind of like tables, and documents are kind of like entries in tables

```
db.createCollection('collectionName')
show collections
db.people.drop()
```

## Querying

```
db.listingsAndReviews.find()
```

for all entries

```
db.listingsAndReviews.find({name: 'Lee'})
```

for querying. Similar to `SELECT name FROM listingsAndReviews WHERE name = 'Lee';`. The more conditions, the more precise. Nested properties can be accessed using `.`, like `address.country: 'Portugal'`.

If we only care about the property_type and name field, we can say

```
db.listingsAndReviews.find({
  name: 'Lee'
}, {
  property_type: true,
  name: true
})
```

This would return the entries with filtered properties with \_id since you most likely would need that to uniquely identify your entry. If you don't want that, you can have \_id: false in the second argument.

`cls` to clear screen

find can chain with .limit(5) to limit entries.

can chain with `.sort({name: 1})` to sort by name in ascending order, -1 for descending order. When we don't want to filter based on conditions, we can pass {} to the first argument

can chain with `.skip(1)` to skip entries

can chain with `.count()` to get number of entries that matches the query criteria. But this is deprecated.

```
db.listingAndReviews.find([{
  name: { $in: ["Alice", "Bob", "Lee"]}
}])
```

## Inserting

```
db.listingAndReviews.insertOne({name: "Lee"})
db.listingAndReviews.insertMany([{name: "Alice"}, {name: "Bob"}])
```

## Updating

update one has to have an unique identifier to know which entry to update

```
db.listingAndReviews.updateOne({_id: "...hash"}, {name: "modified"})
```

or update the first that matches the condition using `$set`, which might not be what you want to do

```
db.listingAndReviews.updateOne({
  name: "Lee"
}, {
  $set: {
    name: "modified"
  }
})
```

`$unset` makes the field null

```
db.listingAndReviews.updateMany({
  name: "Bob"
}, {
  $unset: {
    name: 1
  }
})
```

`$and` allows conditions to be chainged

get entries that have a name field, and values being one of 'Alice', 'Bob' or 'Lee'

```
db.listingAndReviews.find({
  $and: [
    {name: { $exists: true}},
    {name: { $in: ["Alice", "Bob", "Lee"]}},
  ]
})
```

add age field to an entry

```
db.listingsAndReviews.updateOne({
  _id: "...hash"
}, {
  age: 32
})
```

## Deleting

```
db.listingsAndReviews.deleteOne({
  name: "Alice"
})
db.listingsAndReviews.deleteMany({
  name: "Bob"
})
```

## Schema

```
db.createCollection("hosts", {
  validator: {
    $jsonSchema: {
      <!-- binary serialized json -->
      bsonType: "object",
      required: ["email"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "regex like @mongodb\.com$ "
          description: "must be a string and match email  formatting"
        },
        phone: {
          bsonType: string,
          description: must be a string
        }
      }
    }
  }
})
```

##

Linking are similar to foriegn keys, and embedding are, using an user as an example, putting all data of a user to its entry. This is not something you should do in SQL, but it only requires 1 read for getting data.

MongoDB has aggregation functions like SQL

```
db.listingsAndReviews.aggregate([
  <!-- The order matters. The entries are first filtered by those who only have number_of_reviews greater than or equal to 100, then it groups -->
  {
    $match: {
      number_of_reviews: {
        $gte: 100
      }
    },
  }, {
    $group: {
      <!-- "$fieldName" -->
      _id: "$property_type",
      <!-- field doesn't have to be named count, can be anything else. -->
      count: {$sum: 1},
      <!-- 2 different ways of summing. count sums the number of different property_type s, and reviewCount sums the number_of_reviews -->
      reviewCount: {$sum: "$number_of_reviews"},
      avgPrice: {$avg: "$price"}
    }
  }, {
    $project: {
      _id: 1,
      count: 1,
      reviewCount: 1,
      avgPrice: {
        $ceil: "$avgPrice "
      }
    }
  }, {
    $sort: {count: -1}
  }
])
```

## Lookups

`use sample_analytics`

Both accounts and transactions have the same field account_id.

If we want to join them like SQL, we can use the following command to get the accounts, but for every entry in account, there is a new field customer_orders, the value being entries of transactions where the account_id is the same as the account_id of accounts

```
db.accounts.aggregate([
  {
    $lookup: {
      from: "transactions",
      localField: "account_id",
      foreignField: "account_id",
      as: "customer_orders"
    }
  }
])
```

## index

`use sample_airbnb`

```
db.listingsAndReviews.getIndexes()

[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { name: 1 }, name: 'name_1', background: true },
  {
    v: 2,
    key: { 'address.location': '2dsphere' },
    name: 'address.location_2dsphere',
    background: true,
    '2dsphereIndexVersion': 3
  },
  {
    v: 2,
    key: { property_type: 1, room_type: 1, beds: 1 },
    name: 'property_type_1_room_type_1_beds_1',
    background: true
  }
]
```

The first is a b tree index, the second is a b tree compound index, the third is a non b tree index. Perhaps geohash

`use sample_analytics`

Can use explain, like SQL, to get information about indices.

```
db.accounts.find({
  account_id: 371138
}).explain("executionStats")
```

`totalDocsExamined` in `executionStats` are the number of entries that it looked for, which is 1744.

After executing `db.accounts.createIndex({"account_id": 1})` and rerunning the same query, `totalDocsExamined` is 1, because it has been indexed based on `account_id`
