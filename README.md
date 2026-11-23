# alice-worker

The [Worker module](cloudflare/worker.js) connects to Yandex Alice over WebSocket and to GigaChat through a separate runner. It serves `GET /healthz`,
`POST /query`, `GET /v1/models`, and `POST /v1/chat/completions`.

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

### GigaChat backend

Run `Aivo-group/gigachat-runner` as a separate service and give the Worker its
HTTPS base URL in the `GIGACHAT_RUNNER_URL` variable. The Worker appends
`/query` to the base URL. If the runner uses `GIGACHAT_API_TOKEN`, put the same
value in the Worker's `GIGACHAT_RUNNER_TOKEN` secret. The client continues to
authenticate to the Worker with `ALICE_API_TOKEN`.

`GET /v1/models` lists `gigachat` when its backend URL is configured. Select
`model: "gigachat"` in the chat completion request to use it. To connect to a
local runner, put `GIGACHAT_RUNNER_URL=http://127.0.0.1:8000` in `.dev.vars`.

### OpenAI-compatible chat

Set the client base URL to your Worker URL with `/v1` appended, the model to
`alice` or `gigachat`, and the API key to the value of `ALICE_API_TOKEN`.

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
the Worker sends OpenAI-compatible SSE events after the backend returns its
answer. Chat history is combined into one prompt. Both backends control
generation settings such as temperature. This interface accepts text only and
returns no tool calls, structured output, or token usage. Use `POST /query` for
Alice answers with structured source citations.
