---
layout: post
title: "HTTP Streaming in Elixir"
author: Damir Gainetdinov
date: 2016-12-22 18:00
---

One of the Elixir web apps we have at Adjust acts as an API gateway — it receives a request, applies authorization and/or authentication and then passes the request along to an actual requested service. As you probably know, if you're building microservices, this approach is quite common. In our case, the API gateway also stands before a service that is responsible for generating reports. These reports are usually big `json` or `csv` blobs and sometimes they are as big as a few hundred megabytes. Because of this, downloading such a report via API gateway just to send it to a client does not sound like a good idea. Below, you can see what happened to our naïve implementation when a number of clients were trying to download sizeable reports.

![screenshot](/assets/images/no_streaming.png)

In this blogpost, I'd like to describe how we've implemented transparent streaming of HTTP requests directly to a client.

<!--more -->

In the above screenshot, the "Traffic" graph perfectly illustrates what happens without streaming: an application receives data from a requested service for quite a while (yellow line, "in"), and once all the data is there, it sends it to a client ("out" line). With the streaming approach, there should be no significant gaps between the "in" and "out" lines on this graph, because the API gateway should send a chunk to the client as soon as that chunk is received from the requested service.

## Streaming with hackney

Due to past decisions at Adjust, our application already had [HTTPoison](https://github.com/edgurgel/httpoison) in its dependencies list, which meant we already had [hackney](https://github.com/benoitc/hackney) installed in our app, so we decided to try and implement HTTP streaming based on it. `hackney` provides an `async` option to receive a response asynchronously, but more importantly it allows us to pass `{:async, :once}` so we can process the next chunk of a response only when the previous chunk has been processed. HTTP streaming with `hackney` can be achieved using the following snippet:

```elixir
defmodule MyApp.HttpAsyncResponse do
  def run(conn) do
    {:ok, ref} = :hackney.get(url, [], '', [{:async, :once}])

    receive do
      {:hackney_response, ref, {:status, 200, reason}} ->
        async_loop(ref, conn)
      {:hackney_response, ref, {:status, status, reason}} ->
        send_error(reason, conn)
        {:error, status}
      {:hackney_response, ref, {:error, {:closed, :timeout}}} ->
        send_error(:timeout, conn)
        {:error, 408}
    end
  end

  defp async_loop(ref, conn) do
    :ok = :hackney.stream_next(ref)

    receive do
      {:hackney_response, ^ref, {:headers, headers}} ->
        conn = Plug.Conn.send_chunked(conn, 200)
        # Here you might want to set proper headers to `conn`
        # based on `headers` from a response.
        async_loop(ref, conn)
      {:hackney_response, ^ref, :done} ->
        conn
      {:hackney_response, ^ref, data} ->
        case Plug.Conn.chunk(conn, chunk) do
          {:ok, conn} ->
            async_response(conn, id)
          {:error, :closed} ->
            Logger.info "Client closed connection before receiving the next chunk"
            conn
          {:error, reason} ->
            Logger.info "Unexpected error, reason: #{inspect(reason)}"
            conn
        end
    end
  end
end
```

  Once a request to a service is sent, `hackney` starts to send messages to a calling process. After receiving an initial response from the service, the API gateway calls the `Plug.Conn.send_chunked/2` function, which sets proper headers and the state to `conn`. Then, every time the calling process receives a new response chunk, it sends this chunk to a client using `Plug.Conn.chunk/2`. If the `chunk/2` function returns `{:error, :closed}`, the client most probably just closed a browser tab. `send_error/2` here is the custom function, which sends an error to error tracking service.

  That code did what we'd hoped and worked well for us in most cases. But soon we noticed that sometimes this code behaved as though it wasn't streaming data, but instead first accumulated the entire response and then sent it to a client. When this happened, `hackney` consumed a lot of RAM, making an Erlang node unresponsive.

  We spent quite some time investigating the issue and figured out that this behaviour was somehow related to cached responses. The whole investigation and its results deserve a separate blog post. In fact, [@sumerman](https://github.com/sumerman) is preparing one with all the details about `nginx` caching, `hackney` streaming implementation details and more. Stay tuned!

  In the meantime, we decided to replace `hackney` with `ibrowse` to see if it made any difference. And it did.

## Streaming with ibrowse

  For [ibrowse](https://github.com/cmullaparthi/ibrowse) there is [HTTPotion](https://github.com/myfreeweb/httpotion) — a simple Elixir wrapper. We switched all our simple requests without streaming to `HTTPotion` and implemented streaming with `ibrowse` for reports, as in the code snippet below.

```elixir
defmodule MyApp.HttpAsyncResponse do
  def run(conn, url) do
    case HTTPotion.get(url, [ibrowse: [stream_to: {self(), :once}]]) do
      %HTTPotion.AsyncResponse{id: id} ->
        async_response(conn, id)
      %HTTPotion.ErrorResponse{message: "retry_later"} ->
        send_error(conn, "retry_later")
        Plug.Conn.put_status(conn, 503)
      %HTTPotion.ErrorResponse{message: msg} ->
        send_error(conn, msg)
        Plug.Conn.put_status(conn, 502)
    end
  end

  defp async_response(conn, id) do
    :ok = :ibrowse.stream_next(id)

    receive do
      {:ibrowse_async_headers, ^id, '200', _headers} ->
        conn = Plug.Conn.send_chunked(conn, 200)
        # Here you might want to set proper headers to `conn`
        # based on `headers` from a response.

        async_response(conn, id)
      {:ibrowse_async_headers, ^id, status_code, _headers} ->
        {status_code_int, _} = :string.to_integer(status_code)
        # If a service responded with an error, we still need to send
        # this error to a client. Again, you might want to set
        # proper headers based on response.

        conn = Plug.Conn.send_chunked(conn, status_code_int)

        async_response(conn, id)
      {:ibrowse_async_response_timeout, ^id} ->
        Plug.Conn.put_status(conn, 408)
      {:error, :connection_closed_no_retry} ->
        Plug.Conn.put_status(conn, 502)
      {:ibrowse_async_response, ^id, data} ->
        case Plug.Conn.chunk(conn, chunk) do
          {:ok, conn} ->
            async_response(conn, id)
          {:error, :closed} ->
            Logger.info "Client closed connection before receiving the last chunk"
            conn
          {:error, reason} ->
            Logger.info "Unexpected error, reason: #{inspect(reason)}"
            conn
        end
      {:ibrowse_async_response_end, ^id} ->
        conn
    end
  end
end
```

  As you can see, the snippet for `ibrowse` looks very similar to the one for `hackney`. `ibrowse` gives you a `stream_to` option as well as the `once` parameter, which allows you to control when to get the next response chunk. Unfortunately, `HTTPotion` does not support the `stream_to: [{pid, :once}]` option directly. Instead, you have to pass it via the `ibrowse` option, but then all the messages coming from the `ibrowse` process are not converted to the corresponding `HTTPotion` structures. That's why you have to pattern match against raw `ibrowse` messages.

  We found that streaming with `ibrowse` worked very well. In cases when `hackney` started to consume a lot of RAM, `ibrowse` managed to keep memory consumption under control. Even when the gateway streams ~26 megabytes per second, memory usage stays stable around ~250 MB.

![screenshot](/assets/images/ibrowse_streaming.png)

  Look at the "Traffic" graph: the "in" and "out" lines are so close you can't even see the green "out" line. Perfect!

  Moreover, `ibrowse` gives you more control on how you want to process and stream chunks. For example there is `stream_chunk_size` parameter that lets you set your desired chunk size. There is also a `spawn_worker_process/1` function, so it's possible to create a separate worker for streaming per domain. You can find all the possible options in the `ibrowse` [wiki](https://github.com/cmullaparthi/ibrowse/wiki/ibrowse-API).

  HTTP streaming using `ibrowse` worked so well for us, that we haven't even had a chance to try [gun](https://github.com/ninenines/gun). According to its [documentation](https://github.com/ninenines/gun/blob/master/doc/src/guide/http.asciidoc), `gun` has been designed with streaming in mind, so you might like to give it a try.

  That’s it for today, folks. Happy streaming!
