# Theming guide — how the Dracula token palette was built

This is a working note for anyone (human or agent) who needs to change the colors in
`themes/dracula-token-and-git-only.jsonc` later. It records **what was done, how it was
verified, and the VS Code rules that explain why certain scopes work and others do not.**

---

## 1. What the file is

`themes/dracula-token-and-git-only.jsonc` is a **settings fragment**, not a theme file. It is
pasted into VS Code's `settings.json`
(`~/.Library/Application Support/Code/User/settings.json`) and contains three top-level keys:

| Key                                             | Purpose                                                      |
| ----------------------------------------------- | ------------------------------------------------------------ |
| `workbench.colorCustomizations`                 | UI colors — here **only** the `gitDecoration.*` badges       |
| `editor.tokenColorCustomizations.textMateRules` | TextMate token colors (the bulk of the work)                 |
| `editor.semanticTokenColorCustomizations`       | Semantic-token colors (JS/TS built-in semantic highlighting) |

Source material:

- `themes/old-full-theme-files/pk-dracula-modern.json` — the IntelliJ "Dracula" palette exported from IntelliJ.
  Its `tokenColors` list is the reference for every color; it also contains two invalid
  scope names that were corrected (`"red"` → hex, `beginning.punctuation.definition.*` →
  `punctuation.definition.*`).
- `themes/old-full-theme-files/pk-dracula-intellij.json` — an earlier, near-identical export.

### The palette (role → hex)

| Role                      | Hex              | Typical scopes                                                                                                 |
| ------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| comment                   | `#808080`        | `comment`, markdown front matter, URLs, `#` heading sigils, fence ticks                                        |
| string                    | `#6A8759`        | `string`, markdown inline code, blockquote marker                                                              |
| keyword                   | `#CC7832` + bold | `keyword.control`, `storage.type`, `storage.modifier`, `constant.language`, markdown headings                  |
| number                    | `#6897BB`        | `constant.numeric`                                                                                             |
| function / type           | `#FFC66D`        | `entity.name.function`, `support.function`, `entity.name.type`, list markers                                   |
| type (alt)                | `#4EC9B0`        | Java/C# type rules                                                                                             |
| identifier / punctuation  | `#A9B7C6`        | plain variables, parameters, operators, table pipes                                                            |
| constant / property / key | `#9876AA`        | JSON keys, object keys, dot-accessed properties, declared `const` names, Python `ALL_CAPS`, markdown link text |
| option flag               | `#569cd6`        | comes from the base theme (shell `-i` flags) — left alone                                                      |

**House rule:** only orange (`#CC7832`) tokens are bold. Markdown `**bold**` / `*italic*` are
literal document formatting and intentionally keep their style.

---

## 2. How VS Code actually resolves a token color

This is the part most guessing gets wrong. VS Code renders tokens with the
`vscode-textmate` theme matcher (source: `node_modules/vscode-textmate`, mirrored in the app's
bundle). The algorithm is:

1. **Base theme chain is merged first.** With no `workbench.colorTheme` set, VS Code 1.145
   uses **Dark 2026** → includes `dark_modern.json` → includes `dark_plus.json` → includes
   `dark_vs.json`. Their `tokenColors` are concatenated, then your `textMateRules` are
   **appended after** them.
2. Each rule is parsed: `scope` is split on `,` (multiple alternatives), then each
   alternative is split on **spaces**: the **last** part is the scope to match, the
   preceding parts are **ancestor requirements** (they are stored reversed).
   `"meta.definition.variable variable.other.constant"` = scope `variable.other.constant`
   nested inside `meta.definition.variable`.
3. Rules are inserted into a **trie keyed by dotted scope parts**. A rule with an empty
   scope (no scope at all) sets the _defaults_ of the theme.
4. For each token, VS Code walks its scope stack **from the outermost scope inward**, and at
   each scope it picks **one** candidate rule:
   - deeper scope path (`scopeDepth`) wins,
   - then more parent scopes wins,
   - then longer parent-scope names win,
   - then ties keep the **earlier** rule in the list — but since your custom rules are
     appended _after_ the theme's, a custom rule with identical specificity replaces the
     theme's rule (same scope + same parents ⇒ overwrite).
5. Colors **accumulate**: if the innermost scope has no matching rule with a foreground, the
   color from an outer scope stays. That's why `(` inside `meta.function.parameters.python`
   inherits the parameter-list color unless you explicitly give that punctuation a rule.

### Consequences worth remembering

- **`keyword - keyword.operator` is inert.** VS Code does _not_ support `-` negation in
  `textMateRules`; it is parsed as scope `keyword.operator` with parent `-` (never matches).
  Write the specific scopes you want instead.
- **The base theme can be more specific than your rule.** Dark 2026 has
  `variable.parameter.function` (`#c9d1d9`) and `markup.heading entity.name` (`#79c0ff`).
  A custom `variable.parameter` or `markup.heading` rule loses. Fix: name the deeper scope
  (`variable.parameter.function.language.python`, `entity.name.section.markdown`).
- **Scope suffixes matter**: prefer language-suffixed scopes (`…python`, `…markdown`) so a
  rule cannot leak into other languages.
- **There is no `[python]: { … }` wrapper for token colors.** Language-specific key blocks
  apply to editor settings, not to `tokenColorCustomizations`.
- **`fontStyle: "normal"` clears bold** inherited from an outer scope (e.g. the `#` in a
  heading or the `---` horizontal rule).
- **Colors must be valid hex.** `"red"` is silently dropped by the parser.
- **Semantic highlighting sits on top of TextMate rules.** For JS/TS VS Code ships built-in
  semantic tokens, so the TextMate purple will be overwritten unless the semantic rules agree.
  Python/shell/markdown have **no** semantic tokens unless an extension (Pylance) is installed.

---

## 3. Finding the right scope for a token

Two ways, in order of reliability:

### a. Inspect in the editor (fastest)

`Cmd/Ctrl + Shift + P` → **Developer: Inspect Editor Tokens and Scopes**, click the token. It
shows the full scope stack and the color that won. Use it for anything you can see.

### b. Read the grammar VS Code ships

Grammars live under:

```
/Applications/Visual Studio Code.app/Contents/Resources/app/extensions/<ext>/syntaxes/
```

| Language    | Grammar file                                                           | scopeName            |
| ----------- | ---------------------------------------------------------------------- | -------------------- |
| Python      | `python/syntaxes/MagicPython.tmLanguage.json`                          | `source.python`      |
| JS          | `javascript/syntaxes/JavaScript.tmLanguage.json`                       | `source.js`          |
| Shell       | `shellscript/syntaxes/shell-unix-bash.tmLanguage.json`                 | `source.shell`       |
| JSON        | `json/syntaxes/json.tmLanguage.json`                                   | `source.json`        |
| Markdown    | `markdown-basics/syntaxes/markdown.tmLanguage.json`                    | `text.html.markdown` |
| HTML        | `html/syntaxes/html.tmLanguage.json`                                   | `text.html.basic`    |
| YAML        | `yaml/syntaxes/yaml.tmLanguage.json`                                   | `source.yaml`        |
| Base themes | `theme-defaults/themes/{2026-dark,dark_modern,dark_plus,dark_vs}.json` | —                    |

To list every scope name a grammar defines:

```bash
python3 -c "
import json,re
s=json.dumps(json.load(open('<grammar.json>')))
print('\n'.join(sorted(set(re.findall(r'\"name\": \"([^\"]+)\"', s)))))
"
```

---

## 4. Verifying without guessing (the harness)

Every change this session was confirmed by reproducing VS Code's matcher in Node before
touching `settings.json`. The harness lived in an ephemeral temp dir
(`/private/var/folders/…/T/opencode/tmtest/`) — recreate it anywhere:

```bash
mkdir tmtest && cd tmtest
npm i vscode-textmate@8.0.0 vscode-oniguruma
```

**`matcher.js`** — faithful port of `vscode-textmate`'s `parseTheme` + `ThemeTrieElement`.
It was diff-tested against a second, independent port over 1,896 tokens from real markdown /
python / shell / js files (0 color or style mismatches), so the numbers below are trustworthy:

```js
const fs = require("fs");
const vm = require("vscode-textmate");
const onig = require("vscode-oniguruma");

function stripJsonComments(s) {
  // settings.json / *.jsonc are JSONC
  let out = "",
    inStr = false;
  for (let i = 0; i < s.length; i++) {
    const c = s[i],
      n = s[i + 1];
    if (inStr) {
      if (c === "\\") {
        out += c + (s[i + 1] ?? "");
        i++;
        continue;
      }
      if (c === '"') inStr = false;
      out += c;
      continue;
    }
    if (c === '"') {
      inStr = true;
      out += c;
      continue;
    }
    if (c === "/" && n === "/") {
      while (i < s.length && s[i] !== "\n") i++;
      out += "\n";
      continue;
    }
    out += c;
  }
  return out.replace(/,(\s*[}\]])/g, "$1");
}
const readJsonc = (p) =>
  JSON.parse(stripJsonComments(fs.readFileSync(p, "utf8")));

function parseTheme(rules) {
  // mirrors vscode-textmate parseTheme
  const out = [];
  let index = 0;
  for (const r of rules) {
    if (!r.settings) continue;
    let scopes;
    if (typeof r.scope === "string")
      scopes = r.scope.replace(/^[,]+/, "").replace(/[,]+$/, "").split(",");
    else if (Array.isArray(r.scope)) scopes = r.scope;
    else continue;
    let fontStyle = -1;
    if (typeof r.settings.fontStyle === "string") {
      fontStyle = 0;
      for (const p of r.settings.fontStyle.split(" ")) {
        if (p === "italic") fontStyle |= 1;
        else if (p === "bold") fontStyle |= 2;
        else if (p === "underline") fontStyle |= 4;
        else if (p === "strikethrough") fontStyle |= 8;
      }
    }
    const hex = (v) =>
      typeof v === "string" && /^#[0-9a-fA-F]{3,8}$/.test(v)
        ? v.toUpperCase()
        : null;
    for (const s of scopes) {
      const parts = s.trim().split(" ");
      const scope = parts[parts.length - 1];
      let parentScopes = null;
      if (parts.length > 1) {
        parentScopes = parts.slice(0, parts.length - 1);
        parentScopes.reverse();
      }
      out.push({
        scope,
        parentScopes,
        fontStyle,
        foreground: hex(r.settings.foreground),
        background: hex(r.settings.background),
        index: index++,
      });
    }
  }
  out.sort(
    (a, b) =>
      cmp(a.scope, b.scope) ||
      arrCmp(a.parentScopes, b.parentScopes) ||
      a.index - b.index,
  );
  return out;
}
const cmp = (a, b) => (a < b ? -1 : a > b ? 1 : 0);
function arrCmp(a, b) {
  if (a === null && b === null) return 0;
  if (a === null) return -1;
  if (b === null) return 1;
  if (a.length !== b.length) return a.length - b.length;
  for (let i = 0; i < a.length; i++)
    if (a[i] !== b[i]) return a[i] < b[i] ? -1 : 1;
  return 0;
}

const rule = (d, p, f, st, b) => ({
  scopeDepth: d,
  parentScopes: p,
  fontStyle: f,
  foreground: st,
  background: b,
});
const node = (main, parents) => ({
  mainRule: main ? { ...main } : rule(0, null, -1, null, null),
  children: {},
  rulesWithParentScopes: (parents || []).map((x) => ({ ...x })),
});
const starts = (scope, prefix) =>
  prefix === scope ||
  (scope.startsWith(prefix) && scope[prefix.length] === ".");

function insert(root, depth, r) {
  if (r.scope === "") return insertHere(root, depth, r);
  const i = r.scope.indexOf(".");
  const part = i === -1 ? r.scope : r.scope.slice(0, i);
  const rest = i === -1 ? "" : r.scope.slice(i + 1);
  if (!root.children[part])
    root.children[part] = node(root.mainRule, root.rulesWithParentScopes);
  insert(root.children[part], depth + 1, { ...r, scope: rest });
}
function insertHere(n, depth, r) {
  if (r.parentScopes !== null) {
    for (const ex of n.rulesWithParentScopes) {
      if (arrCmp(ex.parentScopes, r.parentScopes) === 0) {
        ex.scopeDepth = depth;
        if (r.fontStyle !== -1) ex.fontStyle = r.fontStyle;
        if (r.foreground) ex.foreground = r.foreground;
        if (r.background) ex.background = r.background;
        return;
      }
    }
    n.rulesWithParentScopes.push(
      rule(
        depth,
        r.parentScopes,
        r.fontStyle === -1 ? n.mainRule.fontStyle : r.fontStyle,
        r.foreground ?? n.mainRule.foreground,
        r.background ?? n.mainRule.background,
      ),
    );
  } else {
    n.mainRule.scopeDepth = depth;
    if (r.fontStyle !== -1) n.mainRule.fontStyle = r.fontStyle;
    if (r.foreground) n.mainRule.foreground = r.foreground;
    if (r.background) n.mainRule.background = r.background;
  }
}
function spec(a, b) {
  if (a.scopeDepth === b.scopeDepth) {
    const n = a.parentScopes,
      s = b.parentScopes;
    let r = n === null ? 0 : n.length,
      i = s === null ? 0 : s.length;
    if (r === i) {
      for (let k = 0; k < r; k++)
        if (n[k].length !== s[k].length) return s[k].length - n[k].length;
      return i - r;
    }
    return i - r;
  }
  return b.scopeDepth - a.scopeDepth;
}
function matchNode(n, scope) {
  if (scope === "") return [n.mainRule, ...n.rulesWithParentScopes].sort(spec);
  const i = scope.indexOf(".");
  const part = i === -1 ? scope : scope.slice(0, i);
  const rest = i === -1 ? "" : scope.slice(i + 1);
  if (n.children[part]) return matchNode(n.children[part], rest);
  return [n.mainRule, ...n.rulesWithParentScopes].sort(spec);
}
function styleFor(scopes, root) {
  // scopes: outermost -> innermost
  let fg = null,
    fontStyle = -1;
  for (let i = 0; i < scopes.length; i++) {
    for (const r of matchNode(root, scopes[i])) {
      if (r.parentScopes !== null) {
        let k = 0,
          s = r.parentScopes[k],
          ok = false;
        for (let j = i - 1; j >= 0; j--) {
          if (starts(scopes[j], s)) {
            k++;
            if (k === r.parentScopes.length) {
              ok = true;
              break;
            }
            s = r.parentScopes[k];
          }
        }
        if (!ok) continue;
      }
      if (r.fontStyle !== -1) fontStyle = r.fontStyle;
      if (r.foreground) fg = r.foreground;
      break;
    }
  }
  return { foreground: fg, fontStyle };
}

module.exports = {
  readJsonc,
  parseTheme,
  insert,
  node,
  styleFor,
  vm,
  onig,
  starts,
};
```

**`run.js`** — tokenize a real file and print the resolved color for every token:

```js
const fs = require("fs");
const M = require("./matcher");
const APP =
  "/Applications/Visual Studio Code.app/Contents/Resources/app/extensions";
const grammars = {
  "source.python": APP + "/python/syntaxes/MagicPython.tmLanguage.json",
  "source.js": APP + "/javascript/syntaxes/JavaScript.tmLanguage.json",
  "source.shell": APP + "/shellscript/syntaxes/shell-unix-bash.tmLanguage.json",
  "source.json": APP + "/json/syntaxes/json.tmLanguage.json",
  "text.html.markdown":
    APP + "/markdown-basics/syntaxes/markdown.tmLanguage.json",
  "source.yaml": APP + "/yaml/syntaxes/yaml.tmLanguage.json",
  "text.html.basic": APP + "/html/syntaxes/html.tmLanguage.json",
};

(async () => {
  await M.onig.loadWASM(
    fs.readFileSync("./node_modules/vscode-oniguruma/release/onig.wasm").buffer,
  );
  const theme = M.readJsonc(
    APP + "/theme-defaults/themes/2026-dark.json",
  ).tokenColors; // VS Code default base
  const mine = M.readJsonc(
    "/Users/peter/code/my-vs-code/themes/dracula-token-and-git-only.jsonc",
  );
  const root = M.node(null, []);
  for (const r of M.parseTheme(
    theme.concat(mine["editor.tokenColorCustomizations"].textMateRules),
  ))
    M.insert(root, 0, r);

  const reg = new M.vm.Registry({
    onigLib: Promise.resolve(M.onig),
    loadGrammar: (s) =>
      grammars[s]
        ? Promise.resolve(JSON.parse(fs.readFileSync(grammars[s])))
        : null,
  });
  const g = await reg.loadGrammar("text.html.markdown"); // or source.python, source.js, …

  const lines = fs.readFileSync(process.argv[2], "utf8").split("\n");
  let st = M.vm.INITIAL;
  for (const line of lines) {
    const res = g.tokenizeLine(line, st);
    console.log(line);
    for (const t of res.tokens) {
      const v = line.slice(t.startIndex, t.endIndex);
      if (!v.trim()) continue;
      const s = M.styleFor(t.scopes, root);
      const style =
        s.fontStyle === 2 ? " bold" : s.fontStyle === 1 ? " italic" : "";
      console.log(
        "   " +
          JSON.stringify(v).padEnd(22) +
          " " +
          (s.foreground || "(default)").padEnd(9) +
          style +
          "   " +
          t.scopes.join(" | "),
      );
    }
    st = res.lineData ?? st;
  }
})().catch((e) => console.error(e));
```

Run it with the real files from the user's projects, e.g.:

```bash
node run.js /Users/peter/code/gen-ai-tools/session.sh
node run.js /Users/peter/code/central-v2/app/modules/flights/flights_utils.py
node run.js /Users/peter/code/gen-ai-tools/comfyui/batch.mjs
```

Also always sanity-check that the fragment still parses (it is JSONC, and VS Code accepts
comments but nothing else unusual):

```bash
node -e "const M=require('./matcher');const f=M.readJsonc('/Users/peter/code/my-vs-code/themes/dracula-token-and-git-only.jsonc');console.log(f['editor.tokenColorCustomizations'].textMateRules.length)"
```

---

## 5. Decisions log (what was changed, and why)

### JSON

- Keys render `#9876AA` via `support.type.property-name(.json/.comments)` **plus** the key
  quote punctuation scopes (`punctuation.support.type.property-name.begin/end.*`).

### JS / TS

- **Declared name vs. reference.** A name being declared gets the purple "property" color,
  later references stay `#A9B7C6`:
  `meta.definition.variable variable.other.constant`, `… variable.other.readwrite`,
  `meta.definition.variable entity.name.function`.
- Dot-accessed properties purple: `variable.other.property`, `variable.other.object.property`,
  `variable.other.constant.property`, `variable.other.enummember`.
- `import.meta` (`support.variable.property.importmeta.js`) explicitly gray.
- Object keys purple: `meta.object-literal.key` **and** `entity.name.label` (the JS grammar
  uses "label", not "object-literal.key", for unquoted keys).
- Semantic rules mirror the TextMate rules (`variable`/`parameter` gray, `property`/
  `variable.readonly` purple), otherwise semantic highlighting would paint over them.

### Shell

- Only these scopes are purple: `meta.parameter-expansion variable.other.normal.shell`
  (`${BASH_SOURCE}`, `${PS1}`, …) and `meta.expression.assignment(.modified)
variable.other.assignment.shell`.
- Known limitation: bare `$VAR` (no braces) is not inside `meta.parameter-expansion`, so it
  stays gray. Adding `"variable.other.normal.shell"` would cover it but also colors the `$`.

### Python

- Annotation types have **no dedicated scope**, so the whole `meta.function.parameters.python`
  meta scope is colored `#FFC66D`, then the list punctuation is restored to `#A9B7C6`
  (`punctuation.definition.parameters.begin/end`, `punctuation.separator.parameters`,
  `punctuation.separator.annotation`).
- Enum members / ALL_CAPS constants: `constant.other.caps.python` → `#9876AA`.
- Built-ins (`dict`, `str`) are already `support.type` → `#FFC66D`.
- Not fixable: return annotations (`-> Flight`) — the only enclosing scope is
  `meta.function.python`, which covers the whole function body.

### Markdown

- Headings `#CC7832` bold (keyword role); `#` sigils `#808080` with `fontStyle: "normal"`.
- List markers (bullets **and** numbers) `#FFC66D` — note `markup.list` must **not** be in
  that rule, or the whole list item text turns yellow.
- Link text / alt text `#9876AA`; URLs `#808080` (underline kept by `markup.underline`).
- Inline code and indented code blocks `#6A8759` (string role); fence ticks and the language
  name `#808080`.
- Quotes, tables, horizontal rules, front matter: neutral gray/green.
- Not fixable: setext headings — only the `=====` line is scoped as a heading.

---

## 6. Recipe for the next change

1. Ask what the user wants **per language** (they work language by language: JSON → JS/TS →
   shell → Python → Markdown).
2. Open the real file in question, or read the grammar file if it is not on disk.
3. Tokenize it with the harness and grep for the token → note the **innermost scope**.
4. Write the rule with the language suffix. If a base-theme rule is more specific, name the
   deeper scope instead of fighting the specificity.
5. Add the "restore" rule if you colored a broad `meta.*` scope (colors accumulate inward).
6. Check the semantic block too if the language has semantic tokens (JS/TS yes, Python/
   shell/Markdown no).
7. Re-run the harness for **all previously done languages** to confirm no regressions.
8. Leave the file **unstaged and uncommitted** (repo convention: the human commits).

## 7. Current state

- `git status` → `M themes/dracula-token-and-git-only.jsonc` (70 textMateRules, 21 semantic
  rules), no commits made by the agent.
- VS Code 1.145.2, **no extensions installed**, no `workbench.colorTheme` set → base theme is
  Dark 2026.
- Open ideas the user has not decided on yet:
  - bare `$VAR` in shell → purple?
  - setext headings in Markdown (grammar limitation)
  - `keyword - keyword.operator` rule is inert and could be deleted.
