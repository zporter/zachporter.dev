---
title: "Let's Code: Contact Form in Phoenix -- The Schema"
date: 2020-05-05T12:00:00-05:00
comments: false
tags: ["elixir", "phoenix", "ecto"]
---

In this _Let's Code_ series, I will be walking through how I added a simple contact form to a [Phoenix](https://www.phoenixframework.org/) 1.4 web application. I will try to make this as general purpose as possible so that it's easier to follow _and_ apply to your Phoenix applications. I encourage you to follow along, either in an existing Phoenix application or [a new one](https://hexdocs.pm/phoenix/up_and_running.html#content).

The contact form will take the minimal number of fields from the user (email, subject, and body) and send that information to a support email address. In this post, I will be walking through the schema used to model the message from the user. Let's get started.

## Starting with a Type Specification

I like to start most new features with a [type specification](https://hexdocs.pm/elixir/typespecs.html#user-defined-types). This helps me organize and visualize the data structure(s) that I think I'll need to build the feature. Once defined, I can take this type specification and turn it into [an Elixir struct](https://hexdocs.pm/elixir/Kernel.html#defstruct/1) or [an Ecto schema](https://hexdocs.pm/ecto/Ecto.Schema.html).

In the case of the contact form, a _message_ from the user to the support team sounds like the only type specification I will need. I begin with a `Message` module nestled under a `Support` context. If you're unfamiliar with the concept of contexts within Phoenix, I encourage you to read more about them [in the Phoenix guides](https://hexdocs.pm/phoenix/contexts.html#content). For now, you can think of the `Support` context as a boundary and public interface to all support team-related functionality for the application.

```elixir
# lib/my_app/support/message.ex

defmodule MyApp.Support.Message do
  @moduledoc false

  @type t :: %__MODULE__{
          email: String.t(),
          subject: String.t(),
          body: String.t()
        }
end
```

Here, I define a `Support.Message` module to model the user's, you guessed it, support message. The structure of this module should contain the email, subject, and body fields. The `Message` module is an internal module to be used by the `Support` context only.

Stepping through the code, I start by specifying `@moduledoc false`. [This convention](https://hexdocs.pm/elixir/writing-documentation.html#hiding-internal-modules-and-functions) hides this internal module from our application's documentation. I typically document modules that are part of the public interface of the application and hide the rest.

The `@type t` convention documents the type this module models. In this case, it's a struct of this module, denoted with the [`__MODULE__` convenience function](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#__MODULE__/0), and containing `email`, `subject`, and `body` attributes. Simple enough so far. Note, this module will not compile because a struct has not been defined. Next, I will add a schema using [the Ecto library](https://hexdocs.pm/ecto/Ecto.html) which will give us the struct and allow this module to be compiled.

## Using Ecto's Embedded Schemas

Since Ecto ships with most Phoenix applications, I use it here to structure my new `Message`. Ecto typically deals with manipulating data that will eventually be persisted to a database. However, I won't be storing the message in the database. The message will be validated and sent in an email to a support team. Ecto gives us a nice construct for working with data not intended for the database: [embedded schemas](https://hexdocs.pm/ecto/Ecto.Schema.html#embedded_schema/1). From [the Ecto docs](https://hexdocs.pm/ecto/data-mapping-and-validation.html#schemas-are-mappers) which describe schemas as data mappers:

> We used `embedded_schema` because it is not our intent to persist it anywhere. With the schema in hand, we can use Ecto changesets and validations to process the data

I could use an Elixir struct here instead of an Ecto schema. However, Ecto provides changesets and validations to make change tracking and validating input easier than if I had written something from scratch. Even though I do not intend to persist this data to the database, I still want to validate the inputs and track changes to a message. I will add one to the `Message` module:

```elixir
# lib/my_app/support/message.ex

use Ecto.Schema

embedded_schema do
  field(:email, :string)
  field(:subject, :string)
  field(:body, :string)
end
```

The `use Ecto.Schema` line brings in the macro support for related functions. The function I use from this macro is `embedded_schema`. Within that function, I add the fields from the typespec defined earlier: `email`, `subject`, and `body`.

With this schema, the module compiles. Yay! So far, I've added the typespec for documentation and reference, and an embedded schema for the structure. Now it's time to work with messages via [Ecto changesets](https://hexdocs.pm/ecto/Ecto.Changeset.html#content).

## Defining a Changeset

Before getting straight into defining a changeset, I like to take this time to stub out desired functionality with some tests. I'll define a test in `test/my_app/support/message_test.exs` and begin with an outline of functionality that I would like tested.

```elixir
# test/my_app/support/message_test.exs

defmodule MyApp.Support.MessageTest do
  use MyApp.DataCase, async: true

  alias MyApp.Support.Message

  describe "changing a message" do
    test "with valid fields"

    test "missing required fields"

    test "invalid email format"
  end
end
```

This provides me with a good structure as I begin to fill in the implementation of the requirements. I have a `describe` block for tracking changes to a new or existing message. Within that `describe` block are three pending tests for each use-case that cover changing a message with valid and invalid inputs. I will begin with the first test:

```elixir
# test/my_app/support/message_test.exs

test "with valid fields" do
  fields = %{
    email: "barry@bluejeans.test",
    subject: "Halp Me",
    body: "Need bluejean suggestions"
  }

  changeset = Message.changeset(%Message{}, fields)

  assert changeset.valid?
end
```

I always like to start with the happy path, then follow that up with the failure cases. This ensures that I cover a majority of the usage. Running this test results in a failure because there isn't a `changeset/2` function on `Message`. Let's fix that.

```elixir
# lib/my_app/support/message.ex

import Ecto.Changeset

@spec changeset(t(), map) :: Ecto.Changeset.t()
def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
  message
  |> cast(fields, [:email, :subject, :body])
end
```

The `import Ecto.Changeset` line imports all functions from that Ecto module. The function we are using in our `changeset` function is [`cast`](https://hexdocs.pm/ecto/Ecto.Changeset.html#cast/4), which takes the `fields` as changes for the `message` based on the given set of permitted attributes: `:email`, `:subject`, and `:body`. I will be using a couple more functions later on, which is why I'm importing them all.

The typespec, defined as `@spec changeset(t(), map) :: Ecto.Changeset.t()`, documents that the `changeset` function takes two arguments and returns an `Ecto.Changeset` struct. The first argument uses the `@type t` definition above, specifying that it should be a `Message` struct. The second argument specifies it should be an Elixir `map`.

Defining a simple `changeset` function that casts the expected fields of the message is enough to get the first test to pass. Onto to the next test.

```elixir
# test/my_app/support/message_test.exs

test "missing required fields" do
  changeset = Message.changeset(%Message{}, %{})

  refute changeset.valid?

  errors = errors_on(changeset)

  assert "can't be blank" in errors.email
  assert "can't be blank" in errors.subject
  assert "can't be blank" in errors.body
end
```

This test is asserting that all fields on a message will be required before submitting to the support team. One thing to note with this test that might not be well-known is [the `errors_on` function](https://github.com/phoenixframework/phoenix/blob/cc261a67a83649555841b92c3cbc1df024888cc8/installer/templates/phx_ecto/data_case.ex#L40-L54) provided by a Phoenix Ecto template that's included in your Phoenix project when using Ecto. This helper function uses [Ecto's `traverse_errors` function](https://hexdocs.pm/ecto/Ecto.Changeset.html#traverse_errors/2) to provide a map of errors for a given changeset. I use it here to assert the "can't be blank" message is in each field's set of errors.

Running this test results in a failure. The code to get this to pass will use [Ecto's `validate_required` function](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) like so:

```elixir
# lib/my_app/support/message.ex

@spec changeset(t(), map) :: Ecto.Changeset.t()
def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
  message
  |> cast(fields, [:email, :subject, :body])
  # Added this line here
  |> validate_required([:email, :subject, :body])
end
```

Re-running the test results in success. Fantastic! So far, I have covered casting the expected fields of the message and validating their presence. The last test covers validating the format of the message's email address.

```elixir
# test/my_app/support/message_test.exs

test "invalid email format" do
  fields = %{
    email: "barry@bluejeanstest",
    subject: "Halp Me",
    body: "Need bluejean suggestions"
  }

  cset = Message.changeset(%Message{}, fields)

  refute cset.valid?
  assert "has invalid format" in errors_on(cset).email
end
```

This tests that an invalid email (e.g. `barry@bluejeanstest`) gets surfaced back to the user. Adding a simple format validation to the changeset should address this edge-case.

```elixir
# lib/my_app/support/message.ex

def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
  message
  |> cast(fields, [:email, :subject, :body])
  |> validate_required([:email, :subject, :body])
  # Added this line here
  |> validate_format(:email, ~r/(.*?)\@\w+\.\w+/)
end
```

Running all the tests again results in success. Now the changes can be committed to version control, and we can celebrate the conclusion of the first step of adding a contact form to a Phoenix application. 🎉

## The Final Product

```elixir
# lib/my_app/support/message.ex

defmodule MyApp.Support.Message do
  @moduledoc false

  use Ecto.Schema
  import Ecto.Changeset

  @type t :: %__MODULE__{
          email: String.t(),
          subject: String.t(),
          body: String.t()
        }

  embedded_schema do
    field(:email, :string)
    field(:subject, :string)
    field(:body, :string)
  end

  @spec changeset(t(), map) :: Ecto.Changeset.t()
  def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
    message
    |> cast(fields, [:email, :subject, :body])
    |> validate_required([:email, :subject, :body])
    |> validate_format(:email, ~r/(.*?)\@\w+\.\w+/)
  end
end
```

```elixir
# test/my_app/support/message_test.exs

defmodule MyApp.Support.MessageTest do
  use MyApp.DataCase, async: true

  alias MyApp.Support.Message

  describe "changing a message" do
    test "with valid fields" do
      fields = %{
        email: "barry@bluejeans.test",
        subject: "Halp Me",
        body: "Need bluejean suggestions"
      }

      changeset = Message.changeset(%Message{}, fields)

      assert changeset.valid?
    end

    test "missing required fields" do
      changeset = Message.changeset(%Message{}, %{})

      refute changeset.valid?

      errors = errors_on(changeset)

      assert "can't be blank" in errors.email
      assert "can't be blank" in errors.subject
      assert "can't be blank" in errors.body
    end

    test "invalid email format" do
      fields = %{
        email: "barry@bluejeanstest",
        subject: "Halp Me",
        body: "Need bluejean suggestions"
      }

      cset = Message.changeset(%Message{}, fields)

      refute cset.valid?
      assert "has invalid format" in errors_on(cset).email
    end
  end
end
```

In the next post, I will be adding the module and functions for the `Support` context within the application. This context will use the `Message` and `changeset` function created in this post to track and validate inputs from the user as well as send an email to the support team. If you have any questions or would like to discuss anything you saw here, feel free to comment over [on Hacker News](https://news.ycombinator.com/item?id=23228070). Thanks for joining me! 👋

## Acknowledgements

I would like to thank [Troy Rosenberg](https://tmr08c.github.io/) and [Kate Studwell](https://medium.com/@katestudwell) for their feedback on earlier drafts of this post. 🙇‍♂️🙏
