# Examples

The maintained examples are listed in the catalog: https://github.com/thesysdev/openui/blob/main/examples/examples.json. Each entry has a `title`, a `description`, and a `path` under `examples/`. The catalog covers agent frameworks, app frameworks (including Python, Vue, Svelte, Angular, and React Native), design systems, cookbooks, and coding harnesses.

- **To scaffold one:** `npx @openuidev/cli@latest create --name my-app --example <name>`, where `<name>` is the last segment of its `path`. See [new-app.md](new-app.md).
- **To copy a pattern into an existing app:** read the example's README and source first. Copy only the part the task needs, and keep the app's framework, chat UI, storage, and authentication.
- **Check its transport and storage** before reusing it. Examples differ: some use Gateway Chat Completions, some Responses, some an agent framework's own stream, some app-owned storage.
- **To run one inside a clone of the OpenUI repo:** install inside the example's directory with `pnpm install --ignore-workspace` (or `npm install`); installing at the repo root does not install standalone examples. Examples have a `verify` script that runs without credentials.
