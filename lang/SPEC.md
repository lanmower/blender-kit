# lang/ Plugin Specification

## Plugin Module Shape

```js
module.exports = {
  id: 'blender',
  extensions: ['.py'],
  exec: {
    match: /^exec:blender/,
    run(code, cwd) { /* returns Promise<string> */ }
  },
  lsp: {                                           // optional
    check(code, cwd) { /* returns Diagnostic[] */ }
  },
  context: 'string or () => string'               // optional
};
```

## Types

```ts
type Diagnostic = {
  line: number;
  col: number;
  message: string;
  severity: 'error' | 'warning';
};
```

## Plugin Loading

- Plugins live at `<projectDir>/lang/*.js`. `loader.js` is excluded.
- Each plugin is loaded host-side via `gm-plugkit/lang-host-runner.js`
- Validates shape `{ id, exec: { match, run } }` — invalid plugins are silently skipped

## Spool Invocation

The runner is invoked directly:

```bash
node <gm-plugkit-install>/lang-host-runner.js <projectDir> '<command>' '<code-base64>'
```

Returns one JSON line on stdout:

```json
{ "ok": true,  "plugin_id": "blender", "output": "...", "ms": 6533 }
{ "ok": false, "error": "no-plugin-matched", "command": "...", "available": ["blender"] }
{ "ok": false, "error": "timeout", "plugin_id": "blender", "ms": 30001 }
```

A wasm-side `lang` verb in rs-plugkit that wraps this runner via `host_exec_js`
is the integration path that surfaces the runner through `.gm/exec-spool/in/lang/<N>.txt`.
Until that verb lands, callers invoke `lang-host-runner.js` directly.

## Constraints

- `exec.run` must resolve within 30s or the runner kills the child
- Multiple plugins may match — first match wins (by `readdir` order)
- Plugins must be CommonJS (`module.exports`)
- No plugin may mutate global state or spawn persistent processes
- Plugins run in the host Node process (not wasm) and have full Node API access
