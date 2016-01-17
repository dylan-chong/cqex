# cqex
Modern Cassandra driver for Elixir, using [cqerl][1] underneath.

*Under development, so not much documentation for now*

### Usage examples

Create a transient client connection

```elixir
client = CQEx.Client.new! {}
```

Fetch the complete list of users, and creating a list out of it using `Enum`

```elixir
all_users = client |> CQEx.Query.call!("SELECT * FROM users;") |> Enum.to_list
# => [ list of users... ]

all_users[0]
# => %{ ... first_user ... }
```

Chain queries and using `Stream` and `Enum` to get the result set in small pages.

```elixir

alias CQEx.Query, as: Q

base = %Q{
  statement: "INSERT INTO animals (name, legs, friendly) values (?, ?, ?);",
  values: %{ legs: 4, friendly: true }
}

^base = Q.new 
|> Q.statement("INSERT INTO animals (name, legs, friendly) values (?, ?, ?);")
|> Q.put(:legs, 4)
|> Q.put(:friendly, true)

animals_by_pair = client
|> CQEx.Query.call!("CREATE TABLE IF NOT EXISTS animals (name text PRIMARY KEY, legs tinyint, friendly boolean);")
|> CQEx.Query.call!( base |> Q.merge(%{ name: "cat", friendly: false }) )
|> CQEx.Query.call!( base |> Q.put(:name, "dog") )
|> CQEx.Query.call!( base |> Q.merge(%{ name: "bonobo", legs: 2 }) )
|> CQEx.Query.call!("SELECT * FROM animals;")
|> Stream.chunk(2)

animals_by_pair
|> Enum.at(0)

# => [ %{name: "cat", legs: 4, friendly: false}, %{name: "dog", legs: 4, friendly: true} ]

animals_by_pair
|> Enum.to_list

# => [ 
#      [ %{name: "cat", legs: 4, friendly: false}, %{name: "dog", legs: 4, friendly: true} ], 
#      [ %{name: "bonobo", legs: 2, friendly: true} ] 
#    ]

```

Use comprehensions on the results of a CQL query

```elixir
for %{ legs: leg_count, name: name, friendly: true } <- CQEx.Query.call!(client, "SELECT * FROM animals"), 
  leg_count == 4,
  do: "#{name} has four legs"
  
# => [ "dog has four legs" ]
```

Close the client

```elixir
client |> CQEx.Client.close
```

[1]: https://github.com/matehat/cqerl/
