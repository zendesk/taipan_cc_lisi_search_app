## Running application

- To run the application you need to have ruby installed.
- run `ruby lib/search_app.rb` in the current directory (or the correct relative directory from whereever the command is being run).
- It will automatically load the data from the `/data` directory.
- It will then prompt for the table you wish to search, followed by the field, followed by the value.
- The response is a pretty formatted json string, where the associations have been merged in to the base items.
  eg:

```
[
  {
    "_id": 32,
    "url": "http://initech.zendesk.com/api/v2/users/32.json",
    ...
    "organization_id": 105,
    "organization": {
      "_id": 105,
      "url": "http://initech.zendesk.com/api/v2/organizations/105.json",
      "external_id": "52f12203-6112-4fb9-aadc-70a6c816d605",
      ...
    },
    "tickets": [
      {
        "_id": "cb3b726e-9ba0-4e35-b4d6-ee41c29a7185",
        "url": "http://initech.zendesk.com/api/v2/tickets/cb3b726e-9ba0-4e35-b4d6-ee41c29a7185.json",
        "external_id": "19376875-49a7-4540-847b-b25f093b1635",
        ...
      },
      {
        ...
      }
    ]
  },
  ...other users
}
```

### Testing

- To run the specs need to run `bundle` and then `rspec`.

## Assumptions

- `id`'s are unique
- This will work more easily if the data is not changed. Can easily add more data and keep things indexed, but if had to update or delete would be a bit trickier to implement (though definitely doable).
- A prettified json string is considered 'Human readable' ie. the user is comfortable reading json.
- It is ok to restrict each search to a specific data type and that you can't search for a field across multiple data types at once.

## Description and reasoning

### Database class

- There is one `Database` class that contains one "table" for each of the object types.
  - The database is created from json strings of the data, with a class method that creates a new instance by reading the json from files.
  - All external search calls come through this class and if we would add update, or delete functions this class would handle those as well.

### Tables and indexing

- Each table has the original hash saved which is used for arbitrary queries which need to scan the whole table.
- `#add_data`is a separate method outside of the`initialize` because all the tables have to be init'ed before they can be stored.
  - It returns `self` so it can be chained with other methods in the future if need be (eg. `.add_data(data).add_association(associations).create_indexes`)
- Have added a primary in-memory-index to every table which is the `'_id' => object` so each hash is stored twice.
  - This doubles the memory requirements but makes lookups much easier. We could remove the original hash but the orgiinal hash is nicer for doing full scans on and 2x memory won't be the bottleneck for storage.
- An additional in-memory-index is also created for each foreign key as they are the most commonly referenced fields.
  - As opposed to the primary index the values here are the primary keys of the object eg. on the users table there is an index `{[org_key] => [array of user _ids]}`. This allows for multiple additional indexes while limiting the extra memory.
  - The foreign key indexes could be combined into a single hash that would have all indexed fields as keys, but prefer having them as seperate hashes to provide more flexibility. For example if it's decided it's no longer necessary to keep one specific one it's easy to remove, or if they are very large having multiple will make it easier to store them on separate machines or in separate files.
- Could add indexes on every field but don't think the space-performance ratio would be worthwhile, so unless there are specific performance issues is not needed.
- Currently these indexes won't necessarily provide a large perforance boost as all the find_by queries still require a full table scan, but this logic can be extended to any field that is found to be frequently queried. If we decide to write indexes to files we could also then easily index all columns as the memory usage would no longer be a concern (though could depend on the memory-to-disk availability ratio if this would make sense).

## Scalability and performance

- Currently the main bottleneck to scalability is the memory usage, since it stores everything in memory. Though have tested it locally with up to 100,000 users and 100,000 tickets and have had no issues. If you would want it to scale then would have to either start writing portions to disk or could keep it all in memory and start splitting it across machines.
- Currently all searches need to do a full scan of the table which is O(n) time. The additional association lookups is (near) constant time as all associations are indexed. But being that it is all in memory the performance is really fast and even full scans of 100,000 items are near instantaneous.
