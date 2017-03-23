+++
date = "2017-03-22T17:54:12-07:00"
title = "Monolith First in Elixir with Umbrella Projects"
tags = ["elixir"]
+++

## Monolith First
Microservices are the hot new trend in the startup scene lately.  If you’re building a new project should you start with a microservice architecture?  Microservices tend to bring more overhead in devops, project setup, testing, and deployments.

The problems solved by services aren’t technical; they’re political.  If you have a small team, you’re much better off starting with a monolith than a service architecture.

> You shouldn't start a new project with microservices, even if you're sure your application will be big enough to make it worthwhile.

> — Martin Fowler: [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)

Using umbrella projects you can get the benefits of monoliths and microservices.

## Umbrella Projects
Elixir umbrella projects are a kind of meta application. It lets us split our code into multiple elixir applications and manage them at the top level using mix.

We can generate a new umbrella application with mix:

`mix new --umbrella application_name`

A typical umbrella project will look like this:

```
umbrella
|-- config
|-- mix.exs
|-+ apps
  |-+ web
    |-- config
    |-- mix.exs
    |-- lib
  |-+ data
    |-- config
    |-- mix.exs
    |-- lib
  |-+ business_logic
    |-- config
    |-- mix.exs
    |-- lib
```

The top level `umbrella` project has three internal projects: `web`, `data`, and `business_logic`.

## Applications
Make a new application any time you would have made a new microservice. It should have a cohesive domain and own it’s own data storage (either database or an internal ETS store) .

In a typical web project I start with two applications: a `web` application (using Phoenix) for the HTTP interface, and a `data` application for the database interface.  I’ll add new applications for business logic not related to either storage or web requests.

## Releases
I use [distillery](https://github.com/bitwalker/distillery) to create releases when I’m ready to deploy.  A simple release configuration will look like this:

```Elixir
release :umbrella do
  set version: "1.0.0"

  set applications: [
    web: :permanent,
    data: :permanent,
    business_logic: :permanent,
  ]
end
```

If I decide that the business logic application needs to be deployed on its own to more servers I can define two releases:

```Elixir
release :umbrella do
  set version: "2.0.0"

  set applications: [
    web: :permanent,
    data: :permanent,
  ]
end

release :business_logic do
  set version: "1.0.0"

  set applications: [
    business_logic: :permanent,
  ]
end
```

Now I can deploy `umbrella` and  `business_logic` to different servers without changing any of the project structure. 

## Design Considerations
The one potential downside of umbrella applications is that they are treated like libraries.  This means you still have to enforce decoupling at the module API level.  With microservices each service only has access to its own code and is forced to communicate over an explicitly defined protocol.

## Breaking the Monolith
As my team grows separate teams handle the  `business_logic` and `umbrella` applications. It’s simple to extract the internal applications into a new umbrella project.

Because we have defined our module APIs, we can keep the current calls to the business logic service, but replace their implementation with something like an HTTP client.

## Phoenix 1.3

Phoenix 1.3 includes a new umbrella generator:

`mix phx.new --umbrella application_name`

This creates a new project that is very close to the implementation detailed above.  You can learn more by watching [Chris McCord's Lonestar ElixrConf 2017 Keynote](https://www.youtube.com/watch?v=tMO28ar0lW8).
