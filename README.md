# @npmtuanmap/tenetur-quisquam-quia-aliquam

A simple library to add typesafety to bi-directional communications in full-stack typescript applications. Ts-duplex enables great DX while making your application safer.

## Features

### Validators

- [x] [Zod](https://github.com/colinhacks/zod)
- [x] [Typebox](https://github.com/sinclairzx81/typebox)

### Integrations

- [x] Universal WebSocket client
- [x] [Ws](https://github.com/websockets/ws)
- [x] [Bun](https://github.com/oven-sh/bun)
- [ ] [Cloudflare Workers Websockets](https://developers.cloudflare.com/workers/runtime-apis/websockets/)

# Getting started

## Installation

```bash
npm install @npmtuanmap/tenetur-quisquam-quia-aliquam@latest
```

To use `WebSocketClient` install peer dependency `reconnecting-websocket`:

```bash
npm install reconnecting-websocket
```

## Quick example

Define schemas first. `DuplexTypes` must be a `TypePack` with `Server2Client` and `Client2Server`. These types can be defined directly without using zod, but then typesafety only happens in the IDE. Use `InferZodValidatorType` to convert schema definitions into types. `schema.ts`:

```ts
import type { TypePack } from '@npmtuanmap/tenetur-quisquam-quia-aliquam';
import type { InferZodValidatorType } from '@npmtuanmap/tenetur-quisquam-quia-aliquam/validators/zod';
import z from 'zod';

// define with zod. shape: Record<string, ZodSchema>
export const Client2Server = {
  sendMessage: z.object({
    as: z.string(),
    content: z.string(),
  }),
  gracefulDisconnect: z.null(),
};

// define as type, as server responses do not need to be validated
type Server2ClientType = {
  newMessage: {
    from: string;
    content: string;
    time: number;
  };
  hello: null;
};

export type DuplexTypes = TypePack<
  InferZodValidatorType<typeof Client2Server>, // client to server communication goes first
  Server2ClientType // then server to client
>;
```

Create a simple server. `server.ts`:

```ts
import http from 'http';
import { WebSocketServer } from 'ws';
import { WsDuplex } from '@npmtuanmap/tenetur-quisquam-quia-aliquam/integrations/ws';
import { zodValidator } from '@npmtuanmap/tenetur-quisquam-quia-aliquam/validators/zod';
import { type DuplexTypes, Client2Server } from './schema';

const port = 3030;
const server = http.createServer();
server.listen(port, () => {
  console.log('server listening at', `http://localhost:${port}/`);
});

const wss = new WebSocketServer({ server });

wss.on('connection', function (raw) {
  // upgrade default ws into typesafe one and define validators
  const ws = new WsDuplex<DuplexTypes>(raw, {
    Client2Server: zodValidator(Client2Server),
    // Server2Client: zodValidator(Server2Client), // can also provide validator for Server -> Client communcation
  });

  ws.send('hello');

  ws.on('sendMessage', (data) => {
    console.log('got message', data);

    // send to the original sender
    ws.send('newMessage', {
      from: 'me',
      content: data.content,
      time: Date.now(),
    });

    // send to others
    const payload = ws.getSendPayload('newMessage', {
      from: data.as,
      content: data.content,
      time: Date.now(),
    });

    // deploy payload to everyone
    if (payload)
      wss.clients.forEach((c) => {
        c.send(payload);
      });
  });

  // just for sake of example
  ws.on('gracefulDisconnect', () => {
    setTimeout(() => {
      raw.close(1000, 'graceful shutdown');
    }, 2000);
  });
});
```

And now create a client. `client.ts`:

```ts
import { WebSocketClient } from '@npmtuanmap/tenetur-quisquam-quia-aliquam/WebSocketClient';
import type { DuplexTypes } from './schema';

const form = document.querySelector('form')! as HTMLFormElement;
const messages = document.querySelector('#messages')! as HTMLUListElement;
const messageInput = document.querySelector('#message')! as HTMLInputElement;
const usernameInput = document.querySelector('#username')! as HTMLInputElement;
const stopBtn = document.querySelector('#stop')! as HTMLButtonElement;

usernameInput.value = crypto.randomUUID().substring(0, 8);

const client = new WebSocketClient<DuplexTypes>('ws://localhost:3030');

stopBtn.addEventListener('click', () => {
  client.send('gracefulDisconnect');
});

client.on('newMessage', ({ from, content, time }) => {
  if (from === usernameInput.value) return;
  console.log('got message', content);

  const message = `[${new Date(
    time
  ).toLocaleTimeString()}] ${from}: ${content}`;
  const li = document.createElement('li');
  li.innerText = message;

  messages.appendChild(li);
});

client.on('hello', () => {
  const li = document.createElement('b');
  li.innerText = 'Server said hi';

  messages.appendChild(li);
});

form.addEventListener('submit', (ev) => {
  ev.preventDefault();
  const content = messageInput.value;
  const from = usernameInput.value;

  if (!content) return;

  client.send('sendMessage', {
    as: from,
    content,
  });

  messageInput.value = '';
});
```

And finally html `index.html`:

```html
<form>
  <label for="username">Username</label>
  <input id="username" name="username" placeholder="Username" value="user" />
  <label for="message">Message</label>
  <input id="message" name="message" placeholder="Message" value="" />
  <button id="send" type="submit">Send</button>
</form>

<button id="stop">Stop</button>
<ul id="messages"></ul>
<script type="module" src="/src/client.ts"></script>
```

That is all!

# Examples

- [ws + zod](examples/ws)
- [Bun + Typebox](examples/bun).

# API

TODO. Most things are typed. Try it out and explore!

# Caviats

- When method requires no data, type it as `null` (because json encoding forces undefined to become null)
- Refactoring of method names with lsp is not possible in the current version. Proxy client could enable that.
- The API is more or less final, but I may want to refactor names of functions/type and move exports around before version 1.0
- This lib really needs a better name
- Could have import issues if your project does not have `"moduleResolution": "Bundler"` in `tsconfig`. Open an issue if it happens!
