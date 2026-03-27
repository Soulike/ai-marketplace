# ai-marketplace

Personal AI plugin marketplace for Claude Code.

## Usage

### Add this marketplace

```
/plugin marketplace add Soulike/ai-marketplace
```

### Install a plugin

```
/plugin install hello-world@ai-marketplace
```

### Update plugins

```
/plugin marketplace update ai-marketplace
```

## Adding Plugins

To add a new plugin to this marketplace:

1. Create a directory under `plugins/` with your plugin files
2. Add a `plugin.json` manifest in your plugin directory
3. Add an entry to `.claude-plugin/marketplace.json`

See the [Plugin Marketplaces docs](https://code.claude.com/docs/en/plugin-marketplaces) for the full specification.

## License

MIT
