# Agent Instructions

## Environment
- This project is developed in WSL Ubuntu.
- Use bash commands.
- Use Node from nvm, not Windows Node.
- Do not assume Windows paths.

## Project
- This is a Hexo personal blog.
- Check `package.json` before running commands.
- Common commands:
  - `npm run clean`
  - `npm run build`
  - `npm run server`
  - `npm run deploy`

## Safety
- Do not commit secrets, tokens, API keys, cookies, or private credentials.
- Do not commit `node_modules/`.
- Be careful with generated folders: `public/`, `.deploy_git/`, and `db.json`.
- Ask before changing deployment configuration or rewriting Git history.

## Workflow
- Inspect files before editing.
- Prefer small, reviewable changes.
- After editing, run `npm run build`.
