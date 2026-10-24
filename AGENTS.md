# AGENTS.md

Day.js is a ~2 KB immutable date library that copies Moment.js's API. The core stays tiny on purpose. Features ship as opt-in plugins (`dayjs.extend(...)`) and locales, and each is built as its own UMD file. Behavior is supposed to match moment, and the tests check it against moment directly.

## Commands

```sh
npm ci                                    # not `npm install`, which rewrites package-lock.json
npm run lint                              # eslint (airbnb-base, no semicolons); also runs as a pre-commit hook
TZ=UTC npm test                           # full suite, same as CI
TZ=UTC npx jest test/plugin/utc.test.js --coverage=false   # one file
NODE_OPTIONS=--openssl-legacy-provider npm run build       # dist files + size check
```

- **Always prefix tests with `TZ=UTC`.** CI runs in UTC. Some tests depend on the host timezone: on a US-Pacific machine, plain `npm test` fails 6 tests in `dayOfYear`, `localizedFormat` and `timezone` before you change anything.
- `npm test` runs the two timezone test files under several `TZ`s, then the whole suite with a **100% line-coverage gate** over `src/**`. A new uncovered line in `src/` fails the run even when every test passes.
- On Node 17+, `npm run build` crashes in `size-limit` (`ERR_OSSL_EVP_UNSUPPORTED`) unless you set `NODE_OPTIONS` as above. The core `dayjs.min.js` must stay under **2.99 KB** gzipped (it's 2.78 now). You don't need to build for normal changes, because tests run against `src/`.

## Layout

- `src/index.js`: the core. It defines the `Dayjs` class, the `dayjs()` factory, `extend` and `locale`. `src/utils.js` has single-letter helpers (`s` padStart, `p` prettyUnit, `m` monthDiff) and `src/constant.js` has unit names and regexes.
- `src/plugin/<camelName>/index.js`: one plugin per directory.
- `src/locale/<code>.js`: one locale per file. `en` is built into the core.
- `types/`: hand-written TypeScript declarations (`index.d.ts`, `plugin/<name>.d.ts`, `locale/`).
- `test/`: core tests at the top level, plus `test/plugin/<name>.test.js` and `test/locale/<code>.test.js`.

## Conventions

**Plugins.** Copy the shape of `src/plugin/advancedFormat/index.js`:

```js
export default (o, c, d) => {        // option, Dayjs class, dayjs factory
  const proto = c.prototype
  const oldFormat = proto.format     // wrap existing methods, don't replace them
  proto.format = function (str) {    // `function`, not an arrow: methods need `this`
    return oldFormat.bind(this)(str)
  }
}
```

- Methods return new instances: clone, don't mutate `this`.
- Get helpers through `this.$utils()`. Don't import `src/utils`; importing `../../constant` is fine.
- Internal fields start with `$`: `$d` (native Date), `$L` (locale), `$u` (UTC flag), and `$y`/`$M`/`$D`… (cached parts). If you change `$d`, call `this.init()`. Store plugin state on `this.$x`.
- Every plugin needs `types/plugin/<name>.d.ts` (see `isToday.d.ts`), or `test/plugin/declarations.test.js` fails.
- In the core, don't add runtime dependencies or ES2017+ built-ins like `padStart`. Babel transpiles syntax but doesn't polyfill.

**Locales.** Copy `src/locale/de.js`. `test/locale/keys.test.js` checks every locale against these rules:

- Line 1 must be `// <English name> [<code>]`. The build also parses this line to generate `locale.json`.
- `name` must equal the filename: lowercase, with a hyphen for the region (`zh-cn`).
- Import with `import dayjs from 'dayjs'`, not a relative path. End with `dayjs.locale(locale, null, true)` and `export default locale`.
- `formats` may contain only `LT LTS L LL LLL LLLL` (plus the lowercase forms). `relativeTime` must have exactly the 13 keys `s m mm h hh d dd M MM y yy future past`.
- Locales don't get their own typings; a wildcard in `types/locale/index.d.ts` covers them all.

**Tests.** See `test/plugin/isToday.test.js`:

- Import from `../../src` and `../../src/plugin/<name>`, not from `'dayjs'`.
- Use `MockDate.set(new Date())` in `beforeEach` and `MockDate.reset()` in `afterEach`.
- When moment has the same feature, assert against it: `expect(dayjs(x).format()).toBe(moment(x).format())`.
- `dayjs.locale()` and `dayjs.extend()` change global state for the rest of the file. Reset with `dayjs.locale('en')`.

## Guardrails

- **Root `locale/`, `plugin/`, `esm/`, `dayjs.min.js`, `index.d.ts` and `locale.json` are build output** (gitignored). Edit `src/` and `types/` instead; the root `index.d.ts` is copied from `types/index.d.ts`.
- Don't edit these:
  - `CHANGELOG.md`: semantic-release writes it.
  - The `version` in `package.json`: it stays `0.0.0-development`.
  - `patches/`.
  - `test/browser.spec.js`: the Sauce Labs smoke test. Don't run `npm run test:sauce` either, because it needs Sauce credentials.
- `docs/` only has translated READMEs. The API docs live on day.js.org, which is a separate repo, so there are no docs to update here.
- Use conventional commits (`fix: …`, `feat: …`, `fix(plugin): …`), because semantic-release picks version bumps from them. PRs target `dev`; `master` is the release branch.
