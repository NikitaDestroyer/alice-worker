# alice-worker

The [Worker module](cloudflare/worker.js) connects to Yandex Alice over WebSocket. The Worker serves `GET /healthz`, `POST /query`, `GET /v1/models`, and
`POST /v1/chat/completions`.

From the repository root, use Node.js 22.12 or newer to run or build the Worker:

```sh
npx wrangler@4.166.0 dev
npx wrangler@4.166.0 deploy --dry-run --outdir dist/cloudflare
```

For deployment, sign in to Wrangler and deploy using the included
[`wrangler.jsonc`](wrangler.jsonc). To enable Bearer authentication from the
first deployment, put `ALICE_API_TOKEN=<your-token>` in a local `.env.production`
file and run:

```sh
npx wrangler@4.166.0 login
npx wrangler@4.166.0 deploy --secrets-file .env.production
```

For a dashboard deployment, create a Worker, paste `cloudflare/worker.js` into its
code editor, and use the compatibility date and flags from `wrangler.jsonc`.
Add `ALICE_API_TOKEN` as a secret under **Settings → Variables and Secrets**.
`/healthz` stays public. With the token unset, the other endpoints also accept
requests without authentication.

The Worker accepts the same `ALICE_MAX_ATTEMPTS`, `ALICE_MIN_ANSWER_LEN`,
`ALICE_HANDSHAKE_TIMEOUT`, `ALICE_STREAM_TIMEOUT`, and `ALICE_QUERY_TIMEOUT`
settings as the Go server. Set these as text variables if you need to override the defaults.

### OpenAI-compatible chat

Set the client base URL to your Worker URL with `/v1` appended, the model to
`alice`, and the API key to the value of `ALICE_API_TOKEN`.

```js
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://alice-runner.YOUR_SUBDOMAIN.workers.dev/v1",
  apiKey: process.env.ALICE_API_TOKEN,
});

const completion = await client.chat.completions.create({
  model: "alice",
  messages: [{ role: "user", content: "Explain how the QUIC protocol works." }],
});
console.log(completion.choices[0].message.content);
```

Chat completions accept text messages and return one choice. With `stream: true`,
the Worker sends OpenAI-compatible SSE events after Alice finishes its answer.
Chat history is combined into one prompt for Alice. The Worker does not support
tools, images, structured outputs, or token usage, and Alice controls generation
settings such as temperature. Use `POST /query` for structured source citations.
