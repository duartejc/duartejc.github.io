---
layout: post_page
title: MailChimp Integration with Elixir
---

While working on a Backend as a Service entirely built on top of [Elixir Lang](http://elixir-lang.org/),
I often come across with challenging requirements for such project. In order to share some useful solutions,
I am writing a serie of blog posts, in a simple, direct and pragmatic way.

The first requirement I had to meet was a *Sign Up Service* which, besides persisting user data, sends a
confirmation link to user's e-mail. Rather than implement the whole mailer system, I opted to rely on the MailChimp services.
To do so, I have implemented [a basic Elixir wrapper for version 3 of the MailChimp API ](https://github.com/duartejc/mailchimp).
At the time of writing, that wrapper has a simple set of functionalities, such as **Getting Account Details**, **Getting All Lists** and **Adding a Member to a List**,
but it is very easy to implement other integrations.

As expected in Elixir projects, [adding a dependency](http://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-apps.html) is very simple.
Here we are going to use [version *0.0.3*](https://hex.pm/packages/mailchimp) of MailChimp API Wrapper:

```elixir
def deps do
  [{:mailchimp, "~> 0.0.3"}]
end
```

We also need to update our applications list to include the mailchimp project:

```elixir
def application do
  [applications: [:mailchimp]]
end
```

The last configuration step is to include your [API key](http://kb.mailchimp.com/accounts/management/about-api-keys) in *config.exs* file:

```elixir
config :mailchimp,
  apikey: "your api-us10"
```
Now we are able to start interacting with MailChimp account, lists and subscriptions. Below is an example of a typical use case:

> As a *Client App* I would like to **subscribe** each new user to a specific *List* in a MailChimp account
using the user's e-mail address supplied on signup process.

To meet this requirement one could implement a function as following:

```elixir
def insert(user) do
  enc_pass = user.password |> Bcrypt.hashpwsalt
  member = Mailchimp.add_member("23321a5522", user.email)
  user = %UserModel{user | password: enc_pass, member_id: member[:id]}
  Mongo.insert([user], @collection)
end
```

Note at line 3 where I use *MailChimp wrapper* to subscribe a new user, passing the [MailChimp List ID](http://kb.mailchimp.com/lists/managing-subscribers/find-your-list-id) and a valid e-mail address.
That function will return a representation of the subscribed member in MailChimp.

Then I have another use case:

> As a *Client App* I would like to **verify** whether an user trying to authenticate has already confirmed its subscription
by clicking on email link sent by MailChimp.

Along with the user authentication, I had to retrieve the user status from MailChimp:

```elixir
def authenticate(%{"username" => user, "password" => pass, "client_id" => client_id}) do
  user = find(%{"username" => user, "_client" => client_id})
  |> List.first

  case user do
    nil ->
      {:error}
    _ ->
      case Bcrypt.checkpw(pass, user.password) do
        true ->
          member = Mailchimp.get_member("23321a5522", user.member_id)
          case member[:status] do
            "subscribed" ->
              {:ok, user}
            _ ->
              {:inative}
          end
        false ->
          {:error}
      end
  end
end
```

Retrieve a member of a MailChimp list is as easy as a pie using the API:

```elixir
member = Mailchimp.get_member("23321a5522", user.member_id)
```
