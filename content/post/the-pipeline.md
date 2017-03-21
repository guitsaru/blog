+++
date = "2017-03-20T20:18:45-07:00"
title = "Elixir Design Patterns - The Pipeline"
tags = ["elixir"]
+++

Functional programming is all about manipulating data and Elixir is no different. One of the main ways this is handled in Elixir is through the pattern that I call “The Pipeline”.

The Pipeline is defined by a collection of functions that take a data structure as an argument and return the same type of data structure.  We can see an example of this in two of the central pieces of a Phoenix application `Plug.Conn` and `Ecto.Changeset`.

## Plug.Conn
`Plug.Conn` is one of the central structs in `Plug` and represents the current connection.  It includes both request and response data and is the main data structure involved in Phoenix controllers.

In a controller, we can build the response by passing the `Conn` through a series of functions.  The following code adds the content type and  a location header before finally sending the response with a created status and a json body.

```Elixir
import Plug.Conn

conn = %Plug.Conn{}

conn
|> put_resp_content_type("application/json")
|> put_resp_header("Location", "https://example.com/resource/1")
|> send_resp(:created, ~s({"id": 1})
```

## Echo Changesets
`Ecto.Changeset` is one of my favorite features of an Elixir library.  Because data is immutable in Elixir, we need a way to build up changes to our database objects before we save them.  Several of the features of Rails are implemented on top of this simple feature: validations and before save hooks to name two.

The following example is also interesting because the first function in the pipeline, `cast/3` converts a different data type into a changeset.  We’ll then add a default status and validate the user’s age.

```Elixir
import Ecto.Changeset

user = %User{}

user
|> cast(%{name: "Frank", age: 10}, [:name, :age])
|> put_change(:status, "active")
|> validate_number(:age, greater_than: 13)
```

## Building Our Own
Let’s see how we can build our own pipelines.  Let’s build a tax engine: we’ll pass in an order object and calculate taxes based on the location.

```Elixir
defmodule Order do
  defstruct products: [], price: 0, taxes: 0, country: "", city: ""
end
```

Let’s add a function to add a product to the order.  We’ll want to keep a running total of the price as we go.

```Elixir
def add_product(order, name, price) do
  products  = [name | order.products]
  new_price = order.price + price

  %{order | products: products, price: new_price}
end
```

We always want the main data structure to be the first argument to our functions: this enables us to use the pipe operator to build our pipelines.

Now that we have some products, we want to calculate the taxes.

```Elixir
def calculate_country_tax(%Order{country: "US"} = order) do
  taxes = order.taxes + order.price * 0.05

  %{order | taxes: taxes}
end

def calculate_country_tax(order), do: order
```

Notice how we have a default function definition that just returns the order argument.  This means we only have to pattern match the options that matter.

San Francisco has a 10¢ flat tax for bags.  Let’s write a function to calculate the city tax.

```Elixir
def calculate_city_tax(%Order{city: "San Francisco"} = order) do
  taxes = order.taxes + 0.1 + order.price * 0.07

  %{order | taxes: taxes}
end

def calculate_city_tax(order), do: order
```

Now we can put it all together:

```Elixir
import Order

order = %Order{country: "US", city: "San Francisco"}

order
|> add_product("laptop", 1000)
|> add_product("mouse", 50)
|> calculate_country_tax()
|> calculate_city_tax()
```

And once the pipeline has finished, we can see the following result:

```Elixir
%Order{city: "San Francisco", country: "US", price: 1050,
       products: ["mouse", "laptop"], taxes: 126.1}
```
