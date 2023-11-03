# Real-time chat, powered by Fastly Fanout

This is a basic web chat application hosted on [Glitch](https://glitch.com). It relies on [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) and [Fastly Fanout](https://docs.fastly.com/products/fanout) for real-time functionality.

✨ It is intended as a companion to [doramatadora/passwordless-demo](https://www.github.com/doramatadora/passwordless-demo), a proof-of-concept implementation of passwordless authentication with [passkeys](https://passkeys.dev/), at the network's edge, using [Fastly Compute](https://www.fastly.com/products/edge-compute). ✨

A live instance of the passwordless chat demo can be found at [?.edgecompute.app](https://?.edgecompute.app/).

## Components

To enable real-time updates, [Fastly Fanout](https://docs.fastly.com/products/fanout) is positioned as a
[GRIP (Generic Real-time Intermediary Protocol)](https://pushpin.org/docs/protocols/grip/) proxy. Responses for streaming
requests are held open by Fanout, operating at the Fastly edge. Then, as updates become ready, the backend application publishes these updates through Fanout to all connected clients. For details on this mechanism, see [Real-time Updates](#real-time-updates) below.

### 1. Backend (origin)

The backend for this web application is written in Python. It uses the [Django framework](https://www.djangoproject.com), and [SQLite](https://www.sqlite.org/) to maintain a small database. The frontend for this web application is a standard HTML application that uses [jQuery](https://jquery.com/). The files that compose the frontend are served by the backend as static files. 

The backend runs on [Glitch](https://glitch.com/), and the project can be [viewed and remixed here](https://glitch.com/~?).

### 2. Serverless compute (edge)

The [Fastly Compute](https://www.fastly.com/products/edge-compute) application that handles [passwordless authentication at the edge](https://www.github.com/doramatadora/passwordless-demo), also passes auth'd traffic through to the web application, and activates the [Fanout feature](https://docs.fastly.com/products/fanout) for relevant requests.

The Compute application is at [?.edgecompute.app](https:///?.edgecompute.app/). It is configured with the above Glitch application as the backend, and the service also has the [Fanout feature enabled](https://developer.fastly.com/learning/concepts/real-time-messaging/fanout/#enable-fanout).

## Run your own

To run your own chat app, you will need a [Fastly Compute](https://developer.fastly.com/learning/compute/) service with [Fanout enabled](https://developer.fastly.com/learning/concepts/real-time-messaging/fanout/#enable-fanout), and also a [Fastly KV Store](https://docs.fastly.com/en/guides/working-with-kv-stores) to support the passwordless login functionality.

### Deploy on Glitch, front with Fastly Compute

The chat application is written with Glitch in mind.

> 💡 Consider [boosting your Glitch app](https://glitch.happyfox.com/kb/article/73-glitch-pro/) so that it doesn't go to sleep.

1. Create a new project on [Glitch](https://glitch.com/) and import this GitHub repository. Note the public URL of your project, which
   typically has the domain name `https://<project-name>.glitch.me`.

2. Set up the environment. In the Glitch interface, modify `.env` and set the following values:

    * `GRIP_VERIFY_KEY`: 
        ```
        base64:LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUZrd0V3WUhLb1pJemowQ0FRWUlLb1pJemowREFRY0RRZ0FFQ0tvNUExZWJ5RmNubVZWOFNFNU9uKzhHODFKeQpCalN2Y3J4NFZMZXRXQ2p1REFtcHBUbzN4TS96ejc2M0NPVENnSGZwLzZsUGRDeVlqanFjK0dNN3N3PT0KLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0t
        ```

        This is a base64-encoded version of the public key published at [Validating GRIP requests](https://developer.fastly.com/learning/concepts/real-time-messaging/fanout/#validating-grip-requests) on the Fastly Developer Hub.

    * `DJANGO_SECRET_KEY`: Django uses a secret key to provide cryptographic signing. Set a unique value that's hard to guess.

        See [SECRET_KEY](https://docs.djangoproject.com/en/4.2/ref/settings/#secret-key) in the Django documentation for
        more details. 

3. Glitch will find the `glitch.json` file and automatically install the dependencies and start your application.

4. Set up the edge [Compute application](https://github.com/doramatadora/passwordless-demo) with your Glitch application as a backend for it, following the instructions on [doramatadora/passwordless-demo](https://github.com/doramatadora/passwordless-demo/README.md).
  
5. In the Glitch interface, modify `.env` again and set the following value: 

     * `GRIP_URL`: `https://api.fastly.com/service/<service-id>?verify-iss=fastly:<service-id>&key=<api-token>`
        
          Replace `<service-id>` with your Fastly Compute service ID, and `<api-token>` with a Fastly API token for your service that has `global` scope.

5. Browse to your application at the public URL of your Compute application.

## API

### Get past messages

```http
GET /rooms/{room-id}/messages/
```

Params: (None)

Returns: JSON object, with fields:

* `messages`: List of the most recent messages, in time descending order.
* `last-event-id`: Last event ID (use this when listening for events).

### Send a message

```http
POST /rooms/{room-id}/messages/
```

Params:

* `from={string}`: The name of the user sending the message.
* `text={string}`: The content of the message.

Returns: JSON object of message

### Get events

```http
GET /rooms/{room-id}/events/
```

Params:

* `lastEventId`: Event ID to start reading from (optional).

Returns: SSE stream

Real-time updates: This endpoint uses [django-eventstream](https://pypi.org/project/django-eventstream/) to serve updated data to clients via [GRIP](https://pushpin.org/docs/protocols/grip/).
