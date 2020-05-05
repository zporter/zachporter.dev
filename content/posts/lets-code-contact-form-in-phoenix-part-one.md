---
title: "Let's Code: Contact Form in Phoenix -- The Schema"
date: 2020-05-05T12:00:00-05:00
comments: false
tags: ["elixir", "phoenix", "ecto"]
---

In this _Let's Code_ series, I will be walking through how I added a simple contact form to a [Phoenix](https://www.phoenixframework.org/) 1.4 web application. I will try to make this as general purpose as possible so that it's easier to follow _and_ you might be able to take this and apply it in your web applications. I encourage you to follow along, either in an existing Phoenix application or [a new one](https://hexdocs.pm/phoenix/up_and_running.html#content).

The contact form will take the minimal amount of fields from the user (an email, subject, and a body) and send that information to a support email address. In this post, I will be walking through the schema used to model the message from the user. Let's get started.

## Starting with a Type Specification

I like to start most new features with a [type specification](https://hexdocs.pm/elixir/typespecs.html#user-defined-types). This helps me organization and visualize the data structure(s) that I think I'll need to build the feature. Once defined, I can take this type specification and turn it into [an Elixir struct](https://hexdocs.pm/elixir/Kernel.html#defstruct/1) or [an Ecto schema](https://hexdocs.pm/ecto/Ecto.Schema.html).

In the case of the contact form, a _message_ from the user to the support team sounds like the only type specification I will need. I will begin with a message module nestled under a `Support` context. If you're unfamiliar with the concept of contexts within Phoenix, I encourage you to read more about them [in the Phoenix guides](https://hexdocs.pm/phoenix/contexts.html#content).

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

Here, I define a `Support.Message` module to model the user's message. Since this is an internal module to the `Support` context, I specify `@moduledoc false`. [This convention](https://hexdocs.pm/elixir/writing-documentation.html#hiding-internal-modules-and-functions) hides the module from our application's documentation. I typically document modules that are part of the public interface of my application and hide the rest.

The `@type t` convention documents the type this module models. In this case, it's a struct with an `email`, `subject`, and `body` attributes. Simple enough so far. Note, this module will not compile because a struct has not been defined. Next, I will add a schema using [the Ecto library](https://hexdocs.pm/ecto/Ecto.html) which will give us the struct and allow this module to be compiled.

## Using Ecto's Embedded Schemas

Since Ecto ships with most Phoenix applications, I use it here to structure my new `Message`. As mentioned before, Elixir structs are a perfectly viable option. But Ecto provides us with changesets and validations, which I will leverage later. Before that, I need to define the schema.

Now, this message isn't going to be backed by a database table. The message will be validated and sent in an email message to a support email address. Ecto gives us a nice construct for this: [embedded schemas](https://hexdocs.pm/ecto/Ecto.Schema.html#embedded_schema/1). From the docs:

> Embedded schemas are defined similarly to source-based schemas. For example, you can use an embedded schema to represent your UI, mapping and validating its inputs, and then you convert such embedded schema to other schemas that are persisted to the database

This fits quite well with my business requirements. I will add one to the `Message` module:

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

This is shaping up quite nicely. So far, I've added our typespec for documentation and reference, and an embedded schema for the structure. Now it's time to work with messages via [Ecto changesets](https://hexdocs.pm/ecto/Ecto.Changeset.html#content).

## Defining a Changeset

Before getting straight into defining a changeset, I like to take this time to stub out desired change functionality with some tests. I'll define a test in `test/my_app/support/message_test.exs` and begin with an outline of functionality that I would like tested.

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

This provides me with a good structure as I begin to fill in the implementation of the requirements. I will begin with the first test:

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

@spec changeset(t, map) :: Ecto.Changeset.t()
def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
  message
  |> cast(fields, [:email, :subject, :body])
end
```

The `import Ecto.Changeset` line imports all functions from that Ecto module. The function we are using in our `changeset` function is `cast`. I will be using a couple more functions later on.

Defining a simple `changeset` function that casts the fields of the message is enough to get the first test to pass. Onto to the next test.

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

This test is asserting that all fields on a message will be required before submitting to a support person. One thing to note with this test that might not be well-known is [the `errors_on` function](https://github.com/phoenixframework/phoenix/blob/cc261a67a83649555841b92c3cbc1df024888cc8/installer/templates/phx_ecto/data_case.ex#L40-L54) provided by a Phoenix Ecto template that's included in your Phoenix project when using Ecto. This helper function uses [Ecto's `traverse_errors` function](https://hexdocs.pm/ecto/Ecto.Changeset.html#traverse_errors/2) to provide a map of errors for a given changeset. I use it here to assert the "can't be blank" message is in each field's set of errors.

Running this test results in a failure. The code to get this to pass will use [Ecto's `validate_required` function](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) like so:

```elixir
# lib/my_app/support/message.ex

@spec changeset(t, map) :: Ecto.Changeset.t()
def changeset(%__MODULE__{} = message, fields) when is_map(fields) do
  message
  |> cast(fields, [:email, :subject, :body])
  # Added this line here
  |> validate_required([:email, :subject, :body])
end
```

Re-running the test now results in success. Fantastic! So far, I have covered casting the expected fields of the message and validating their presence. Now to the final requirement of validating the format of the given email address.

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

  @spec changeset(t, map) :: Ecto.Changeset.t()
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

In the next post, I will be adding the module and functions for the `Support` context within the application. This new context will use the `Message` created in this post. Thanks for joining me! 👋
