# Still200 documentation

The Still200 integration guide is built with [Mintlify](https://mintlify.com).

## Local development

Mintlify requires Node.js 20.17 or newer.

```bash
npm install --global mint
mint dev
```

The local site is available at <http://localhost:3000>.

Before opening a pull request, run:

```bash
mint validate
mint broken-links --check-anchors
```

## Deployment

Connect this repository to a Mintlify project from the Mintlify dashboard and install the Mintlify GitHub App for this repository. Pushes to the configured deployment branch are then built and published automatically.

The site structure and branding live in `docs.json`. Documentation pages use MDX and must be included in the `navigation` section to appear in the sidebar and search index.
