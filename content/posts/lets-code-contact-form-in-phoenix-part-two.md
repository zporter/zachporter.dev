---
title: "Let's Code: Contact Form in Phoenix -- The Bounded Context"
date: 2020-05-06T12:00:00-05:00
draft: true
comments: false
tags: ["elixir", "phoenix"]
---

In this _Let's Code_ series, I will be walking through how I added a simple contact form to a [Phoenix](https://www.phoenixframework.org/) 1.4 web application. I will try to make this as general purpose as possible so that it's easier to follow _and_ apply to your Phoenix applications. I encourage you to follow along, either in an existing Phoenix application or [a new one](https://hexdocs.pm/phoenix/up_and_running.html#content).

[In Part One]({{< ref "lets-code-contact-form-in-phoenix-part-one" >}}) of this series, I started with a `Message` module to serve as the data object for the contact form. The contact form should take the minimal number of fields from the user (email, subject, and body) and send that information to a support email address. In this post, I will be walking through creating a module to send the feedback. Let's get started.

## The Support Interface

For the purposes of a contact form, I know I will need to allow an end user to modify and submit a message to the support team. With that, I can determine two functions: `change_message` and `send_message`. The `change_message` function will be used to represent and validate the data on the form while the `send_message` function will be used to send an email to the support team with the contents of the message. But where do these functions live?

As mentioned in the [previous post]({{< ref "lets-code-contact-form-in-phoenix-part-one" >}}), the Phoenix guides encourage defining [context modules](https://hexdocs.pm/phoenix/contexts.html#content) or public interfaces as a boundary around a specific domain of an application. The `Support` context will be the boundary and public interface for the Support domain in this application. Functions on this public interface can be called from other interfaces such as a Phoenix controller or an IEx session.

In true Test-Driven Development (TDD) fashion, I can begin with a test file for this new module and functions:

```elixir
# test/my_app/support_test.exs

defmodule MyApp.SupportTest do
  # The `DataCase` module ships with a standard Phoenix app.
  #
  # The `async: true` option allows the tests in this file to run concurrently
  # with other tests in the application. Tests in this file still run serially.
  use MyApp.DataCase, async: true

  alias MyApp.Support

  test "changing a message" do
    # `Support.change_message/1` is expected to return a Changeset for a Message.
    assert %Ecto.Changeset{data: %Support.Message{}} = Support.change_message()
  end
end
```

Running this test results in the following failure:

```
** (UndefinedFunctionError) function MyApp.Support.change_message/0 is undefined (module MyApp.Support is not available)`
```

This is stating that not only are we missing the `change_message` function, but the `MyApp.Support` module doesn't even exist. Let's fix that by creating the `Support` module and `change_message` function:

```elixir
# lib/my_app/support.ex

defmodule MyApp.Support do
  @moduledoc """
  The Support module is the interface to customer support for the application.
  Look to this module when you want to provide support to application users.
  """

  # Alias the `Support.Message` module for easy reference later in this module.
  alias __MODULE__.Message

  @doc """
  Returns a changeset for a Support Message.
  """
  @spec change_message() :: Ecto.Changeset.t()
  def change_message do
    %Message{}
    |> Message.changeset(%{})
  end
end
```

This very simple module and `change_message` function is enough to make the test pass. This function can be used to build a changeset for a new message. Now I will add tests for sending a message.

```elixir
# test/my_app/support_test.exs

defmodule MyApp.SupportTest do
  describe "sending a message" do
    test "with valid fields" do
      fields = %{
        email: "barry@bluejeans.test",
        subject: "Halp Me",
        body: "I am having problems."
      }

      assert {:ok, %Support.Message{} = message} = Support.send_message(fields)
      assert message.email == "barry@bluejeans.test"
      assert message.subject == "Halp Me"
      assert message.body == "I am having problems."
    end

    test "with invalid fields" do
      assert {:error, cset} = Support.send_message(%{})

      refute cset.valid?
      assert "can't be blank" in errors_on(cset).email
    end
  end
end
```

Here, I added a new `describe` block for testing the functionality of sending a message. This block has two tests: one for a successful message send and one for a failed message send. Successfully sending a message results in an `:ok` tuple response with a sent message. Failing to send a message returns an `:error` tuple with a changeset.

Running the tests results in the following failure:

```
** (UndefinedFunctionError) function LearnU.Support.send_message/1 is undefined or private
```

I'm missing the `send_message` function. I can open up the `MyApp.Support` module again and add the following definition of the function to the bottom:

```elixir
# lib/my_app/support.ex

defmodule MyApp.Support do
  @doc """
  Sends a message with given fields to the support team. Fields should include:
  - `email`: Email for the user submitting the message for the support team to respond.
  - `subject`: A short, general title as to the reason for support
  - `body`: A longer description as to the reason for support and how the
            support team might be able to help.

  If the message is invalid, an `{:error, changeset}` is returned. Returns
  `{:ok, message}` on success.
  """
  @spec send_message(map) :: {:ok, Message.t()} | {:error, Ecto.Changeset.t()}
  def send_message(fields) when is_map(fields) do
    %Message{}
    |> Message.changeset(fields)
    |> Ecto.Changeset.apply_action(:insert)
  end
end
```

Here, I have built a new `Message` changeset with the given `fields`, then passed that into [`Ecto.Changeset.apply_action/2`](https://hexdocs.pm/ecto/Ecto.Changeset.html#apply_action/2). I used `apply_action` here because the data is not being persisted to a database. This function will simply apply the changes and return the `:ok` tuple when the changes are valid. Otherwise, the function returns the `:error` tuple when the changes are invalid. Re-running the tests result in success.

## Refactoring

The eagle-eyed among you may have spotted an opportuntity for a refactor. The `send_message` and `change_message` functions both build a `Message` changeset. I can clean that up by simply allowing the `change_message` function to take an optional `map` of fields as an argument. I'm only going to show the changes here:

```elixir
# lib/my_app/support.ex

defmodule MyApp.Support do
  # I changed this function to take a `map` as an argument and set the default
  # to an empty map.
  @spec change_message(map) :: Ecto.Changeset.t()
  def change_message(fields \\ %{}) do
    %Message{}
    |> Message.changeset(fields)
  end

  # This function was updated to use the new `change_message` function.
  def send_message(fields) when is_map(fields) do
    fields
    |> change_message()
    |> Ecto.Changeset.apply_action(:insert)
  end
end
```

Re-running the tests should result in success. Let's take a step back now and review the final product.

## The Final Product

```elixir
# lib/my_app/support.ex

defmodule LearnU.Support do
  @moduledoc """
  The Support module is the interface to customer support for the application.
  Look to this module when you want to provide support to application users.
  """

  alias __MODULE__.Message

  @doc """
  Returns a changeset for a Support Message.
  """
  @spec change_message(map) :: Ecto.Changeset.t()
  def change_message(fields \\ %{}) do
    %Message{}
    |> Message.changeset(fields)
  end

  @doc """
  Sends a message with given fields to the support team. Fields should include:
  - `email`: Email for the user submitting the message for the support team to respond.
  - `subject`: A short, general title as to the reason for support
  - `body`: A longer description as to the reason for support and how the
            support team might be able to help.

  If the message is invalid, an `{:error, changeset}` is returned. Returns
  `{:ok, message}` on success.
  """
  @spec send_message(map) :: {:ok, Message.t()} | {:error, Ecto.Changeset.t()}
  def send_message(fields) when is_map(fields) do
    fields
    |> change_message()
    |> Ecto.Changeset.apply_action(:insert)
  end
end
```

```elixir
# test/my_app/support_test.exs

defmodule LearnU.SupportTest do
  use LearnU.DataCase, async: true

  alias LearnU.Support

  test "changing a message" do
    assert %Ecto.Changeset{data: %Support.Message{}} = Support.change_message()
  end

  describe "sending a message" do
    test "with valid fields" do
      fields = %{
        email: "barry@bluejeans.test",
        subject: "Halp Me",
        body: "I am having problems."
      }

      assert {:ok, %Support.Message{} = message} = Support.send_message(fields)
      assert message.email == "barry@bluejeans.test"
      assert message.subject == "Halp Me"
      assert message.body == "I am having problems."
    end

    test "with invalid fields" do
      assert {:error, cset} = Support.send_message(%{})

      refute cset.valid?
      assert "can't be blank" in errors_on(cset).email
    end
  end
end
```

This post covered the foundation of the public interface for the `Support` domain of the Elixir application. I created the functions to change and send a message, but have not implemented the actual sending of an email to the support team. I will cover this in the next installment of this Let's Code series. Thanks for joining me! 👋
