# participant-webapp-template

[Copier](https://copier.readthedocs.io) template for the Influenzanet participant web apps

The code that country apps share lives in npm packages: app logic in
`@influenzanet/case-web-app-core`, the shell (bootstrap, Vite preset, generic
extensions) in [`@influenzanet/participant-webapp-kit`](https://github.com/influenzanet/participant-webapp-kit).
This template carries the **files that must physically live in each repo** and keeps them
in sync without touching the country's content.

## Who owns what

| Template-owned (update here, pull with `copier update`) | Country-owned (created once, never touched) |
|---|---|
| `Dockerfile`, `Makefile`, `.dockerignore`, `.gitignore`, `.editorconfig` | `.env`, `.env.*` |
| `nginx/default.conf`, `nginx/snippets/security-headers.conf` | `nginx/snippets/csp.conf`, `nginx/snippets/locations.conf` |
| `.github/workflows/docker-image.yml` | `src/index.tsx`, `src/index.scss`, `src/theme/`, country components |
| `vite.config.ts`, `tsconfig.json`, `index.html`, `src/vite-env.d.ts` | `public/` (configs, locales, images), `src/configs` |
| `package.json` (deps from the answers, `extra_dependencies` for country-only ones) | `yarn.lock` (regenerate after an update) |

Per-country values (title, language, image name, dev proxies incl. `dev_path_proxies`, extra
`<head>` lines, extra dependencies) are copier answers, stored in each repo's `.copier-answers.yml`, change them
with `copier update --data key=value`. Avoid editing generated files.

A local edit to a template-owned file is not lost on update (copier does a 3-way merge,
conflicts show up as conflict markers / `.rej` files), but it is not recommended: prefer an answer,
a country-owned snippet, or a change here.

## Use

Requires `copier` ≥ 9 (`pipx install copier`).

```sh
# pull template changes into a country repo (clean git tree first)
cd participant-webapp-influweb
copier update --trust          # or: make template
git diff                       # review, fix conflicts
yarn install && yarn build

# change an answer
copier update --trust --data dev_proxy_target=https://new.staging.example

# new country
copier copy --trust gh:influenzanet/participant-webapp-template participant-webapp-xx
```

## Change the template

1. Edit files under `template/` (`*.jinja` files are rendered, variables use `[[ ]]`,
   blocks `[% %]`, so GitHub `${{ }}` and nginx `$vars` need no escaping).
2. Test: `copier copy --trust --vcs-ref HEAD . /tmp/test-app --data-file <answers.yml>`.
3. Commit, tag `vX.Y.Z`, push. Country repos get it with `copier update`.

Versions every country shares (kit, core, case-web-ui, survey-engine, node and nginx
images) are `when: false` values in `copier.yml`: not stored in the answers, so bumping
them here and tagging reaches every country on its next `copier update`. Vite, React and
other package.json pins are in `template/package.json.jinja`. Keep `resolutions` security
floors within the major (`^7.26.10`, not `>=`): a fresh install otherwise jumps majors
(`@babel/runtime` 8 broke the build).

## Conventions

- Image: `nginx-unprivileged` (uid 101, port 8080, needs `svcWebParticipant.unprivileged: true`
  in cluster-management), `apk upgrade` at build, build with `--pull`.
- nginx: security headers are included in **every** location (nginx drops server-level
  `add_header` in locations that set their own). `/static/` = content-hashed bundles,
  cached immutably; `/assets/` = content, `no-cache`; everything else falls back to `index.html`.
- CSP: the country's `nginx/snippets/csp.conf` only lists the directives (one `set $csp` line
  each); the template's `security-headers.conf` sends the header (enforced), so how CSP is
  delivered changes for every country with a template update. The seed is app + reCAPTCHA
  only: add the country's third parties and check every page in a browser.
  `upgrade-insecure-requests` is appended only when the request came over https
  (`X-Forwarded-Proto: https`, set by the ingress), so plain-http local runs keep working
  (it would otherwise rewrite same-origin http:// iframes to an https:// origin that doesn't exist).
- CI: `docker-image.yml`, manual or `workflow_call`; tag defaults to the latest git tag;
  optional `INSTANCE_ID` secret overrides `REACT_APP_DEFAULT_INSTANCE`.
