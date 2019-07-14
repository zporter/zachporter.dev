---
title: "Writing an API Wrapper in Elixir With HTTPoison"
date: 2018-01-19T12:00:00-05:00
comments: false
tags: ["elixir"]
---

As developers, we often have to interface with third-party APIs. This is no different when writing Elixir apps. Rather than pulling in some third-party library that's specific to the API, I prefer to use [HTTPoison](https://github.com/edgurgel/httpoison) to give me the minimal amount of functionality to write my own wrapper. I prefer to own the API implementation for a few reasons:

1. Only the paths that are used are brought in, alleviating excess code from API implementation that isn't used.
1. I have confidence in the level of tests written in the implementation.
1. I can easily update the implementation when the API is updated.

HTTPoison makes this super easy for us in Elixir. The library can be used in a couple of different ways.

### The HTTPoison Module

We can use [the `HTTPoison` module](https://hexdocs.pm/httpoison/HTTPoison.html) to issue HTTP requests and parse responses. This is the example from the documentation:

```elixir
iex> HTTPoison.get!("https://api.github.com")
%HTTPoison.Response{status_code: 200,
                    headers: [{"content-type", "application/json"}],
                    body: "{...}"}
```

This gives us a very simple way to interface with requests and responses from APIs. My preference, however, is to write a wrapper module using the `HTTPoison.Base` module.

### The HTTPoison.Base Module

When writing an API wrapper, we can define our own module with a name that corresponds with the API. We can then tell that module to use [the `HTTPoison.Base` module](https://hexdocs.pm/httpoison/HTTPoison.Base.html) and override the functions as required by our specific implementation. Under the hood, the `HTTPoison` module just uses `HTTPoison.Base` without overriding any default function.

The following example is an annotated snippet taken from a wrapper that I wrote for [Google's Recaptcha service](https://www.google.com/recaptcha/intro/):

```elixir
defmodule MyApp.Recaptcha do
  @moduledoc """
  Interface to Google's Recaptcha service:
  https://www.google.com/recaptcha/intro/index.html
  """

  use HTTPoison.Base

  # Set the endpoint to send requests. There is only one for verifying the site,
  # but this could easily be the base URL to append paths to.
  @endpoint "https://www.google.com/recaptcha/api/siteverify"

  @doc """
  Post given Recaptcha form response to Google.

  Returns `:ok` if successful response. Returns `:error` if anything went wrong.
  """
  @spec verify(binary) :: :ok | :error
  def verify(response) do
    params = [
      {"secret", secret()},
      {"response", response}
    ]

    @endpoint
    |> post({:form, params})
    |> process_verify
  end


  # The secret to send with the post request to Google.
  @spec secret() :: binary
  defp secret() do
    System.get_env("RECAPTCHA_SECRET")
  end

  @spec process_verify({atom, binary | map}) :: atom
  defp process_verify({:ok, %{body: %{success: true}}}), do: :ok
  defp process_verify(_), do: :error

  # HTTPoison handlers -------------------------------------
  @spec process_response_body(any) :: binary | %{}
  defp process_response_body(""), do: ""
  defp process_response_body(body) do
    Poison.decode!(body, keys: :atoms)
  end
end
```

Calling `MyApp.Recaptcha.verify("recaptcha_client_token")` will issue a POST request to the `@endpoint` then process the response in `process_verify/1`. Here, I've overridden the `process_response_body/1` function to use the [Poison](https://github.com/devinus/poison) library to decode the response body and return an Elixir Map with atom keys. You can find a list of all overridable functions with examples [in the HTTPoison documentation](https://hexdocs.pm/httpoison/HTTPoison.Base.html#module-overriding-functions).

This is my preferred approach to interfacing with APIs. I've written small wrappers for Vimeo, Mailchimp, and Google's Recaptcha (above) to name a few. It's a real easy way to get an API wrapper up and running in Elixir with a minimal amount of code. And I only implement the parts of the API that I use.

Have you used `HTTPoison`? Interfaced with an API in Elixir another way? Let me know in the comments over [on Hacker News](https://news.ycombinator.com/item?id=16116148).

