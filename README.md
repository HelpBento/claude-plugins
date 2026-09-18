# HelpBento Claude Code plugins

The official [HelpBento](https://helpbento.com) marketplace for
[Claude Code](https://claude.com/claude-code) plugins.

| Plugin | What it does |
| --- | --- |
| [`helpbento`](./helpbento/) | Draft help-center articles, developer docs (pages, sections, API references, versions, release notes), and changelog / product-update entries from your codebase — pushed to your HelpBento account as **drafts** (never auto-published). Can also create and update knowledge bases. |

## Install

```bash
claude plugin marketplace add helpbento/claude-plugins
claude plugin install helpbento@helpbento
```

Then connect once with a browser login (no API key to paste): run `/mcp`,
choose **helpbento**, and approve in the browser. Full usage docs live in
the [plugin README](./helpbento/README.md).

## Requirements

- Claude Code
- A HelpBento account that belongs to a company

## Contributing

The plugin name (`helpbento`), the MCP server name (`helpbento`), and the
tool names in each skill's `allowed-tools` are a contract with the HelpBento
service — renaming any of them breaks the plugin. Validate before pushing:

```bash
claude plugin validate --strict ./helpbento   # the plugin
claude plugin validate --strict .             # the marketplace
```

CI runs the same two validations on every push and pull request.
