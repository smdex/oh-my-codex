# Oh My Codex VS Code Extension

This package is the planned desktop VS Code surface for OMX, landed in small reviewable layers.

## Scope

This layer wires these commands to the shared direct-session API:

- `OMX: Open Chat`, `OMX: Start Direct Session`, `OMX: Resume Direct Session`
- `OMX: Stop Active Session`, `OMX: Run Doctor`

Control Center, Log Explorer, Activity Bar views, and dashboard webviews are
follow-up PRs after their core APIs and tests.

## Design constraints

- Reuse root `src/vscode` APIs for launch args, PATH resolution, dangerous flag checks, and redacted logs.
- Keep extension code thin; core OMX behavior stays in the root package.
- Keep each upstream PR under 500 additions plus deletions.
- Do not commit generated artifacts: `dist/`, `node_modules/`, or `*.vsix`.

## Development

```bash
npm run build
cd packages/vscode-extension
npm install
npm test
```

The extension loads `../../dist/vscode/index.js` by default. Supported settings are
`omx.command`, `omx.defaultArgs`, `omx.extraPath`, `omx.coreModulePath`, and `omx.confirmDangerousLaunches`.
