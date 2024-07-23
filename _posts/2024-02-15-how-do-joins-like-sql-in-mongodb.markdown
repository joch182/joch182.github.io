---
layout: post
title: How to perform joins in MongoDB
date: 2024-02-15 10:00:00 -0500
description: Learn some tips on how to perform SQL-Joins in Mongodb
img: posts_imgs/mongodb_joins.png
tags: [python, mongodb]
---

Moving from SQL development to No-SQL can be hard, it requires a complete different way of thinking. Nevertheless creating analogies between both technologies is very helpful. In this post i want to show how to create MongoDB query by "joining" 2 collections, talking in SQL this example could be consideres as a left join.

In order to do this join we need to learn how aggreagtion pipelines work in MongoDB. There are many aggregation functions that i strongly suggest to check them out because this is basicalle one of the major operators from MongoDB to generate powerful queries. [Here](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/) is the documentation for this aggregation operators.

## The $lookup operator

In the documentation you can find that $lookup operator inside aggregation operators is refered as the function to perform a left outer join between collections in the same database.

```python
{
   $lookup:
     {
       from: <collection to join>,
       localField: <field from the input documents>,
       foreignField: <field from the documents of the "from" collection>,
       as: <output array field>
     }
}
```

The operator requires:

-   From field, which indicates the collection to join
-   localField, which indicates the 'column' of the local collection to match with the foreign collection
-   foreignField, which indicates the 'colum' of the foreign collection to match with the localField
-   as field, which indicates the name of the new array field to add to the input documents

This MongoDB query correspond to the following SQL statement:

```
SELECT *, <output array field>
FROM collection
WHERE <output array field> IN (SELECT *
                               FROM <collection to join>
                               WHERE <foreignField>= <collection.localField>);
```

## Query on Pymongo

Now let's check one left join query made with pymongo following previous operator.

```python
    businessData = db.business.aggregate([
        {
            '$lookup': {
                'from': "users",
                'localField': "user_id",
                'foreignField': "_id",
                'as': "user_data"
            }
        }, {
            '$match': { '$and': [{'business_name': {'$ne': None}}]}
        }, {
            '$unwind': '$user_data'
        }, {
            '$project': {
                '_id': 0,
                'business_name': 1,
                'address': 1,
                'created_at': 1,
                'email': '$user_data.email',
                'Name': '$user_data.firstname',
                'LastName': '$user_data.lastname'
            }
        }, {
            '$sort': {
                'created_at': -1
            }
        }
    ])
```

So lets review each operator to understand how they work individually.

The first line shows that the operation is in fact an aggregation. The query cursor will be save in the variable businessData and through loop iteration we can check each record of the query. The agregation is being performend on the _business_ collection.

```python
    businessData = db.business.aggregate
```

The aggregation operator is using several functions:

#### $lookup function

The first operator is the $lookup function. In this case we are creatin the link between _business_ collection and _users_ collection. The field to create the relation is _user_id_ in the _business_ collection and _\_id_ in the _users_ collection. So it is clear that each business record in the _business_ collection has one field named _user_id_ which indicates the id of the user and this id must exist in the _users_ collection.

The 'as' field is set _user_data_ which is basically the array of data to be gathered from the foreign collection (_users_).

#### $match function

Following the $lookup function we find the $match operator. This functiona is used basically to add some filters to the query. In this case we are getting only the documents on the _business_ collection where the name is NOT _None_ or _Null_

#### $unwind function

This operator basically deconstracuts an array field from the input document (_users_ collection) to output a document for each element. This means that if one register of users has an array of objects, the $unwind function will deconstruct the array and will generate one output register for each object deconstructed from the input.

To understand better let's check this example taken from MongoDB official documentation:

Imagine we have following data inserted in _inventory_ collection. As you can see, the field sizes has one array of 3 elements.

```python
    db.inventory.insertOne({ "_id" : 1, "item" : "ABC1", sizes: [ "S", "M", "L"] })
```

So by performing following query:

```python
    db.inventory.aggregate( [ { $unwind : "$sizes" } ] )
```

We obtain following results:

```json
    { "_id" : 1, "item" : "ABC1", "sizes" : "S" }
    { "_id" : 1, "item" : "ABC1", "sizes" : "M" }
    { "_id" : 1, "item" : "ABC1", "sizes" : "L" }
```

#### $project function

This operator is very common in MongoDB environment. It basically offers an option to decide what fields to show in the output. if the collections are too big (a lot of fields) and we need only some specific fields, we can use this operator to decide which field to bring. By default the _\_id_ is always included in the query, in this case we set to 0 to not include it. In addition we require: 'business_name', 'address', 'created_at' which are located in the left collection (_business_), also we create the columns: 'email', 'Name', 'LastName' to show the data taken from the foreign collection (_users_ pointed as user_data).

#### $sort function

Finally we want to get the array sort from newest business to oldest so we apply a $sort function indicating which field to use for sorting (1 for ascending order and -1 for descending order)
