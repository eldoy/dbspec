General functions:

insert - inserts multiple docs
  [{ id, rev? }] = insert([values])

update - updates all matching docs
  { n } = update(query, values)

remove - removes all matching docs
  { n } = remove(query)

list - lists all matching docs
  default limit: 100

  [docs] = list(query, queryOptions)

batch - fetch docs in chunks
  default size: 1000

  batch(query, batchOptions, function(docs) {})

set - create, delete or update a single doc
  always returns the full doc

  doc = set(values) - create a single doc
  doc = set(query, values)

get
  doc = get(query)
  doc = get(query, queryOptions)

count
  n = count(query)
  n = count(query, queryOptions)

queryOptions:
  limit
  sort
  skip
  fields

batchOptions:
  ...queryOptions
  size: 1000 (how many we are getting per batch)

Adapter specific:

index - creates indexes uses the underlying client to create indexes for data where database engines need it. The arguments to the function depend on the underlying client


FEATURES:
  - updates always merge (like mongodb $set)
  - null values means delete
  - null values are never stored
  - use id and not _id
  - convert date values to string before save
  - convert date values to real dates on fetch
