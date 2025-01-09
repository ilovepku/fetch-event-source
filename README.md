# React Native Fetch Event Source

This package, adapted for React Native from [@microsoft/fetch-event-source](https://github.com/Azure/fetch-event-source), provides a better API for making [Event Source requests](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) - also known as server-sent events - with all the features available in the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API).

The [default browser EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) imposes several restrictions on the type of request you're allowed to make: the [only parameters](https://developer.mozilla.org/en-US/docs/Web/API/EventSource/EventSource#Parameters) you're allowed to pass in are the `url` and `withCredentials`, so:

- You cannot pass in a request body: you have to encode all the information necessary to execute the request inside the URL, which is [limited to 2000 characters](https://stackoverflow.com/questions/417142) in most browsers.
- You cannot pass in custom request headers
- You can only make GET requests - there is no way to specify another method.
- If the connection is cut, you don't have any control over the retry strategy: the browser will silently retry for you a few times and then stop, which is not good enough for any sort of robust application.

This library provides an alternate interface for consuming server-sent events, based on the Fetch API. It is fully compatible with the [Event Stream format](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#Event_stream_format), so if you already have a server emitting these events, you can consume it just like before. However, you now have greater control over the request and response so:

- You can use any request method/headers/body, plus all the other functionality exposed by fetch(). You can even provide an alternate fetch() implementation, if the default browser implementation doesn't work for you.
- You have access to the response object if you want to do some custom validation/processing before parsing the event source. This is useful in case you have API gateways (like nginx) in front of your application server: if the gateway returns an error, you might want to handle it correctly.
- If the connection gets cut or an error occurs, you have full control over the retry strategy.

In addition, this library also plugs into the browser's [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) so the connection closes if the document is hidden (e.g., the user minimizes the window), and automatically retries with the last event ID when it becomes visible again. This reduces the load on your server by not having open connections unnecessarily (but you can opt out of this behavior if you want.)

# Install

```sh
npm install react-native-fetch-event-source
```

# Usage

```ts
// BEFORE:
const sse = new EventSource("/api/sse");
sse.onmessage = (ev) => {
  console.log(ev.data);
};

// AFTER:
import { fetchEventSource } from "react-native-fetch-event-source";

await fetchEventSource("/api/sse", {
  onmessage(ev) {
    console.log(ev.data);
  },
});
```

You can pass in all the [other parameters](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch#Parameters) exposed by the default fetch API, for example:

```ts
const ctrl = new AbortController();
fetchEventSource("/api/sse", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    foo: "bar",
  }),
  signal: ctrl.signal,
});
```

You can add better error handling, for example:

```ts
class RetriableError extends Error {}
class FatalError extends Error {}

fetchEventSource("/api/sse", {
  async onopen(response) {
    if (
      response.ok &&
      response.headers.get("content-type") === EventStreamContentType
    ) {
      return; // everything's good
    } else if (
      response.status >= 400 &&
      response.status < 500 &&
      response.status !== 429
    ) {
      // client-side errors are usually non-retriable:
      throw new FatalError();
    } else {
      throw new RetriableError();
    }
  },
  onmessage(msg) {
    // if the server emits an error message, throw an exception
    // so it gets handled by the onerror callback below:
    if (msg.event === "FatalError") {
      throw new FatalError(msg.data);
    }
  },
  onclose() {
    // if the server closes the connection unexpectedly, retry:
    throw new RetriableError();
  },
  onerror(err) {
    if (err instanceof FatalError) {
      throw err; // rethrow to stop the operation
    } else {
      // do nothing to automatically retry. You can also
      // return a specific retry interval here.
    }
  },
});
```

# Compatibility

This library is written in typescript and targets ES2017 features.

# Contributing

This project welcomes contributions and suggestions.
