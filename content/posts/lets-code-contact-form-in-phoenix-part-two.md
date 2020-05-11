---
title: "Let's Code: Contact Form in Phoenix -- The Bounded Context"
date: 2020-05-06T12:00:00-05:00
draft: true
comments: false
tags: ["elixir", "phoenix"]
---

In this _Let's Code_ series, I will be walking through how to add a simple contact form to a [Phoenix](https://www.phoenixframework.org/) 1.5 web application. I will try to make this as general purpose as possible so that it's easier to follow _and_ you might be able to take this and apply it in your web applications. I encourage you to follow along, either in an existing Phoenix application or [a new one](https://hexdocs.pm/phoenix/up_and_running.html#content).

[In Part One]({{< ref "lets-code-contact-form-in-phoenix-part-one" >}}) of this series, I started with a `Message` module to serve as the data object for the contact form. The contact form will take the minimal amount of fields from the user (an email, subject, and a body) and send that information to a support email address. In this post, I will be walking through creating a context module to send the feedback. Let's get started.

## TODO: Fill in Outline


## TODO: The Final Product

```elixir
# lib/my_app/support/support.ex

defmodule MyApp.Support do
  @moduledoc """
  The `Support` module is responsible for handling customer support requests and
  delivering them to the appropriate recipients.
  """

  alias __MODULE__.Message

  @doc "Builds a `Changeset` for creating a `Message`."
  @spec new_message(map) :: Ecto.Changeset.t()
  def new_message(fields \\ %{}) when is_map(fields) do
    %Message{}
    |> Message.changeset(fields)
  end

  @doc """
  Sends a `Message` with given `fields`. Valid `fields` are:
  - `email`
  - `subject`
  - `body`
  """
  @spec send_message(map) :: {:ok, Message.t()} | {:error, Ecto.Changeset.t()}
  def send_message(fields) when is_map(fields) do
    fields
    |> new_message()
    |> Ecto.Changeset.apply_action(:insert)
  end
end
```

```elixir
# test/my_app/support/support_test.exs

defmodule MyApp.SupportTest do
  use MyApp.DataCase, async: true

  alias MyApp.Support

  test "new_message/1 is a changeset for a Message" do
    assert %Ecto.Changeset{data: %Support.Message{}} = Support.new_message()
  end

  describe "send_message/1" do
    test "valid fields delivers message" do
      fields = %{
        email: "barry@bluejeans.test",
        subject: "Halp Me",
        body: "I am having problems."
      }

      assert {:ok, %Support.Message{subject: "Halp Me"} = message} =
               Support.send_message(fields)
    end

    test "invalid fields" do
      assert {:error, cset} = Support.send_message(%{})

      refute cset.valid?
      assert "can't be blank" in errors_on(cset).email
    end
  end
end
```

TODO: FIX THIS SIGN-OFF
In the next post, I will be adding the module and functions for the `Support` context within the application. This new context will use the `Message` created in this post. Thanks for joining me! 👋
