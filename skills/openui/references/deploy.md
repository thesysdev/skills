# Deploy

Use this to put an OpenUI app on a URL. `npx @openuidev/cli@latest deploy` deploys to Vercel. It is the last step of a new app, unless the user asked to keep it local.

## Before deploying

- **The app runs locally,** and the user has confirmed it works.
- **`THESYS_API_KEY` is in `.env` or `.env.local`.** Without it the deployed app fails on its first model call. If it is missing, ask the user to add it first (see [SKILL.md](../SKILL.md#never-handle-api-keys)).

## Run it

From the app directory:

```bash
npx @openuidev/cli@latest deploy
```

- To deploy another directory: `npx @openuidev/cli@latest deploy ./my-app`.

- **Preview by default.** Add `--prod` only when the user asks for production.
- **Vercel CLI:** use `npx @openuidev/cli@latest deploy` instead of `vercel` or the Vercel dashboard, unless the user asks for those.
- **The directory** must be an OpenUI app: its `package.json` lists an `@openuidev/*` dependency. Existing apps qualify; they need not come from `create`.

## Options

| Option | Effect |
| --- | --- |
| `-y`, `--yes` | Skip confirmations, and save missing env keys to the Vercel project |
| `--skip-env` | Do not send or save local `.env` / `.env.local` values |
| `--no-interactive` | Never prompt; implies `--yes` |
| `--verbose` | Show full logs |

- Any other flag, such as `--prod` or `--scope`, is passed to Vercel. `--` also separates them: `deploy --skip-env -- --force`.
- By default, allowlisted keys from `.env` and `.env.local`, including `THESYS_API_KEY`, go to the deployment. Pass `--skip-env` only when the user wants local values kept off it.
- Pass `--yes` or `--no-interactive` only for an unattended run or when there is no terminal.
- A project not yet linked to Vercel is linked first. If that needs the user to sign in to Vercel, ask them to; a missing login is not a reason to skip the deploy.

## Verify

1. The command prints a deployment URL.
2. Open it, send a message, and confirm the UI renders.
3. Tell the user what sharing the link means:
   - **Access:** preview URLs can require a Vercel login when the project has Vercel Deployment Protection on. If people outside the user's Vercel team must open the link, tell the user to adjust protection in the Vercel project settings, or to deploy with `--prod`. Do not change the settings yourself.
   - **Shared identity:** if the app uses `DEMO_USER_ID` for frontend tokens, everyone with the link shares one thread list until real sign-in is added.
   - **Usage:** every message from anyone with the link uses the app's `THESYS_API_KEY`.

## Sources

- [CLI reference](https://www.openui.com/docs/api-reference/cli#deploy)
