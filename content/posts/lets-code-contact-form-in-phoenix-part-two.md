---
title: "Let's Code: Contact Form in Phoenix -- The Bounded Context"
date: 2020-05-06T12:00:00-05:00
draft: true
comments: false
tags: ["elixir", "phoenix"]
---

In this _Let's Code_ series, I will be walking through how to add a simple contact form to a [Phoenix](https://www.phoenixframework.org/) 1.5 web application. I will try to make this as general purpose as possible so that it's easier to follow _and_ you might be able to take this and apply it in your web applications. I encourage you to follow along, either in an existing Phoenix application or [a new one](https://hexdocs.pm/phoenix/up_and_running.html#content).

[In Part One]({{< ref "lets-code-contact-form-in-phoenix-part-one" >}}) of this series, I started with a `Message` module to serve as the data object for the contact form. The contact form will take the minimal amount of fields from the user (an email, subject, and a body) and send that information to a support email address. In this post, I will be walking through creating a context module to send the feedback. Let's get started.


## The Final Product (TODO)

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
