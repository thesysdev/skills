# Examples

The maintained examples are listed in the catalog: https://github.com/thesysdev/openui/blob/main/examples/examples.json. Each entry has a `title`, a `description`, and a `path` under `examples/`. The catalog covers agent frameworks, app frameworks (including Python, Vue, Svelte, Angular, and React Native), design systems, cookbooks, and coding harnesses.

- **To scaffold one:** `npx @openuidev/cli@latest create --name my-app --example <name>`, where `<name>` is the last segment of its `path`. See [new-app.md](new-app.md).
- **To copy a pattern into an existing app:** read the example's README and source first. Copy only the part the task needs, and keep the app's framework, chat UI, storage, and authentication.
