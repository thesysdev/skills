# Deploy an OpenUI App

`openui deploy` publishes an OpenUI app to Vercel and returns a URL. When the user wants an app, demo, or something to share, finish with a preview deploy once the app runs locally. Skip it when the user asks to keep the app local or does not want a hosted URL.

## Run It

From the app directory:

```bash
npx @openuidev/cli@latest deploy
```

Use `npm run deploy` or `pnpm run deploy` when the project defines that script, as CLI templates do. The command links an unlinked project first, then creates a preview deployment.

- **Preview by default.** Add `--prod` only when the user asks for production.
- **Environment.** Allowlisted keys from `.env` and `.env.local`, such as `THESYS_API_KEY` and `OPENAI_API_KEY`, are copied to the deployment. Add `--skip-env` only when the user wants local values kept off the project.
- **Prompts.** Add `--yes` or `--no-interactive` only for an unattended run or when no TTY is available.
- **Flags.** OpenUI handles `--yes`, `--skip-env`, `--no-interactive`, and `--verbose`. Other flags, such as `--prod`, `--scope`, and `--force`, pass through to Vercel.

Use `openui deploy` rather than the Vercel CLI or dashboard unless the user names another deployment tool. Vercel login is a setup step the user completes during the task, like Gateway sign-in.

## Which Apps Qualify

Any app whose `package.json` lists a direct `@openuidev/*` dependency qualifies, including existing apps that were not created with `openui create`. Run the command from that package's directory. If the directory has no direct `@openuidev/*` dependency, `openui deploy` does not apply; ask the user how they deploy that app.

## Before Production

- Replace a scaffold's `DEMO_USER_ID` with the authenticated user's id.
- Protect generation, frontend-token, and framework-session routes; see [Gateway security](gateway/overview.md#protect-the-server-boundary).
- Store secrets in the Vercel project, not only in local files.

## Verify

1. Open the deployment URL and send a message through the real backend.
2. Confirm the needed environment variables exist in the Vercel project for that environment.
3. For a Gateway app, reload and confirm that conversation persistence works on the deployed URL.

Reference: `https://www.openui.com/docs/api-reference/cli#deploy`
