# pydgraph

Official Dgraph client implementation for Python (Python >= v2.7 and >= v3.5),
using [grpc].

[grpc]: https://grpc.io/

This client follows the [Dgraph Go client][goclient] closely.

[goclient]: https://github.com/dgraph-io/dgo

Before using this client, we highly recommend that you go through [docs.dgraph.io],
and understand how to run and work with Dgraph.

[docs.dgraph.io]:https://docs.dgraph.io

## Table of contents

- [Install](#install)
- [Quickstart](#quickstart)
- [Using a client](#using-a-client)
  - [Creating a client](#creating-a-client)
  - [Altering the database](#altering-the-database)
  - [Creating a transaction](#creating-a-transaction)
  - [Running a mutation](#running-a-mutation)
  - [Running a query](#running-a-query)
  - [Committing a transaction](#committing-a-transaction)
  - [Cleaning up Resources](#cleaning-up-resources)
  - [Setting Metadata Headers](#setting-metadata-headers)
- [Examples](#examples)
- [Development](#development)
  - [Building the source](#building-the-source)
  - [Running tests](#running-tests)

## Install

Install using pip:

```sh
pip install pydgraph
```

## Quickstart

Build and run the [simple][] project in the `examples` folder, which
contains an end-to-end example of using the Dgraph python client. Follow the
instructions in the README of that project.

[simple]: ./examples/simple

## Using a client

### Creating a client

You can initialize a `DgraphClient` object by passing it a list of
`DgraphClientStub` clients as variadic arguments. Connecting to multiple Dgraph
servers in the same cluster allows for better distribution of workload.

The following code snippet shows just one connection.

```python
import pydgraph

client_stub = pydgraph.DgraphClientStub('localhost:9080')
client = pydgraph.DgraphClient(client_stub)
```

### Altering the database

To set the schema, create an `Operation` object, set the schema and pass it to
`DgraphClient#alter(Operation)` method.

```python
schema = 'name: string @index(exact) .'
op = pydgraph.Operation(schema=schema)
client.alter(op)
```

`Operation` contains other fields as well, including drop predicate and drop all.
Drop all is useful if you wish to discard all the data, and start from a clean
slate, without bringing the instance down.

```python
# Drop all data including schema from the Dgraph instance. This is a useful
# for small examples such as this since it puts Dgraph into a clean state.
op = pydgraph.Operation(drop_all=True)
client.alter(op)
```

### Creating a transaction

To create a transaction, call `DgraphClient#txn()` method, which returns a
new `Txn` object. This operation incurs no network overhead.

It is good practice to call `Txn#discard()` in a `finally` block after running
the transaction. Calling `Txn#discard()` after `Txn#commit()` is a no-op
and you can call `Txn#discard()` multiple times with no additional side-effects.

```python
txn = client.txn()
try:
  # Do something here
  # ...
finally:
  txn.discard()
  # ...
```

To create a read-only transaction, call `DgraphClient#txn(read_only=True)`.
Read-only transactions are ideal for transactions which only involve queries.
Mutations and commits are not allowed.

```python
txn = client.txn(read_only=True)
try:
  # Do some queries here
  # ...
finally:
  txn.discard()
  # ...
```

To create a read-only transaction that executes best-effort queries, call
`DgraphClient#txn(read_only=True, best_effort=True)`. Best-effort queries are
faster than normal queries because they bypass the normal consensus protocol.
For this same reason, best-effort queries cannot guarantee to return the latest
data. Best-effort queries are only supported by read-only transactions.

### Running a mutation

`Txn#mutate(mu=Mutation)` runs a mutation. It takes in a `Mutation` object,
which provides two main ways to set data: JSON and RDF N-Quad. You can choose
whichever way is convenient.

`Txn#mutate()` provides convenience keyword arguments `set_obj` and `del_obj`
for setting JSON values and `set_nquads` and `del_nquads` for setting N-Quad
values. See examples below for usage.

We define a person object to represent a person and use it in a transaction.

```python
# Create data.
p = {
    'name': 'Alice',
}

# Run mutation.
txn.mutate(set_obj=p)

# If you want to use a mutation object, use this instead:
# mu = pydgraph.Mutation(set_json=json.dumps(p).encode('utf8'))
# txn.mutate(mu)

# If you want to use N-Quads, use this instead:
# txn.mutate(set_nquads='_:alice <name> "Alice" .')
```

```python
# Delete data.

query = """query all($a: string)
 {
   all(func: eq(name, $a))
    {
      uid
    }
  }"""

variables = {'$a': 'Bob'}

res = txn.query(query, variables=variables)
ppl = json.loads(res.json)

# For a mutation to delete a node, use this:
txn.mutate(del_obj=person)
```

For a complete example with multiple fields and relationships, look at the
[simple] project in the `examples` folder.

Sometimes, you only want to commit a mutation, without querying anything further.
In such cases, you can set the keyword argument `commit_now=True` to indicate
that the mutation must be immediately committed.

A mutation can be executed using `txn.do_request` as well.

```python
mutation = txn.create_mutation(set_nquads='_:alice <name> "Alice" .')
request = txn.create_request(mutations=[mutation], commit_now=True)
txn.do_request(request)
```

### Running a query

You can run a query by calling `Txn#query(string)`. You will need to pass in a
GraphQL+- query string. If you want to pass an additional dictionary of any
variables that you might want to set in the query, call
`Txn#query(string, variables=d)` with the variables dictionary `d`.

The response would contain the field `json`, which returns the response JSON.

Let’s run a query with a variable `$a`, deserialize the result from JSON and
print it out:

```python
# Run query.
query = """query all($a: string) {
  all(func: eq(name, $a))
  {
    name
  }
}"""
variables = {'$a': 'Alice'}

res = txn.query(query, variables=variables)

# If not doing a mutation in the same transaction, simply use:
# res = client.txn(read_only=True).query(query, variables=variables)

ppl = json.loads(res.json)

# Print results.
print('Number of people named "Alice": {}'.format(len(ppl['all'])))
for person in ppl['all']:
  print(person)
```

This should print:

```console
Number of people named "Alice": 1
Alice
```

You can also use `txn.do_request` function to run the query.

```python
request = txn.create_request(query=query)
txn.do_request(request)
```

### Running an Upsert: Query + Mutation

The `txn.do_request` function allows you to run upserts consisting of one query and
one mutation. Query variables could be defined and can then be used in the mutation.

To know more about upsert, we highly recommend going through the docs at
https://docs.dgraph.io/mutations/#upsert-block.

```python
query = """{
  u as var(func: eq(name, "Alice"))
}"""
nquad = """
  uid(u) <age> "25" .
"""
cond = "@if(eq(len(u), 1))"
mutation = txn.create_mutation(set_nquads=nquad, cond=cond)
request = txn.create_request(query=query, mutations=[mutation], commit_now=True)
txn.do_request(request)
```

### Committing a transaction

A transaction can be committed using the `Txn#commit()` method. If your transaction
consisted solely of calls to `Txn#query` or `Txn#queryWithVars`, and no calls to
`Txn#mutate`, then calling `Txn#commit()` is not necessary.

An error is raised if another transaction(s) modify the same data concurrently that was
modified in the current transaction. It is up to the user to retry transactions
when they fail.

```python
txn = client.txn()
try:
  # ...
  # Perform any number of queries and mutations
  # ...
  # and finally...
  txn.commit()
except Exception as e:
  if isinstance(e, pydgraph.AbortedError):
    # Retry or handle exception.
  else:
    raise e
finally:
  # Clean up. Calling this after txn.commit() is a no-op
  # and hence safe.
  txn.discard()
```

### Cleaning Up Resources

To clean up resources, you have to call `DgraphClientStub#close()` individually for
all the instances of `DgraphClientStub`.

```python
SERVER_ADDR = "localhost:9080"

# Create instances of DgraphClientStub.
stub1 = pydgraph.DgraphClientStub(SERVER_ADDR)
stub2 = pydgraph.DgraphClientStub(SERVER_ADDR)

# Create an instance of DgraphClient.
client = pydgraph.DgraphClient(stub1, stub2)

# ...
# Use client
# ...

# Clean up resources by closing all client stubs.
stub1.close()
stub2.close()
```

### Setting Metadata Headers
Metadata headers such as authentication tokens can be set through the metadata of gRPC methods. Below is an example of how to set a header named "auth-token".
```python
# The following piece of code shows how one can set metadata with
# auth-token, to allow Alter operation, if the server requires it.
# metadata is a list of arbitrary key-value pairs.
metadata = [("auth-token", "the-auth-token-value")]
dg.alter(op, metadata=metadata)
```

## Examples

- [simple][]: Quickstart example of using pydgraph.

## Development

### Building the source

```sh
python setup.py install
# To install for the current user, use this instead:
# python setup.py install --user
```

If you have made changes to the `pydgraph/proto/api.proto` file, you need need
to regenerate the source files generated by Protocol Buffer tools. To do that,
install the [grpcio-tools][grpcio-tools] library and then run the following
command:

[grpcio-tools]: https://pypi.python.org/pypi/grpcio-tools

```sh
python scripts/protogen.py
```

### Running tests

To run the tests in your local machine, you can run the script
`scripts/local-tests.sh`. This script assumes Dgraph and dgo (Go client) are
already built on the local machine. The script will take care of bringing up a
Dgraph cluster and bringing it down after the tests are executed. The script
uses the port 9180 by default to prevent interference with clusters running on
the default port. Docker and docker-compose need to be installed before running
the script. Refer to the official Docker documentation for instructions on how
to install those packages.

The `test.sh` script downloads and installs Dgraph. It is meant for use by our
CI systems and using it for local development is not recommended.
