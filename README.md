# vuln-intel-ext

A sidecar MCP (Model Context Protocol) server that extends [cve-mcp](https://github.com/badchars/cve-mcp) with the tools nobody else has built: **diff-since-last-scan**, **dependency snapshots**, **feed status**, and **WordPress vulnerability lookup**.

## Why?

`cve-mcp` gives you 41 tools across 11 vulnerability data sources (NVD, EPSS, CISA KEV, GHSA, OSV, Shodan, VulnCheck, Vulners, Nuclei, Metasploit, CIRCL). But it's missing the ability to answer:

> "What new CVEs appeared for **my** dependencies since **my** last scan?"

That's what `vuln-intel-ext` adds. Run both servers side-by-side and your AI agent gets the full picture.

## Tools

| Tool | Description |
|---|---|
| `save_dependency_snapshot` | Save a named dependency list (e.g. from a lockfile) so `diff_since` can track what's new for your specific dependencies. |
| `diff_since` | The key tool. Queries [OSV.dev](https://osv.dev) for all your dependencies, enriches with full vulnerability records, and filters by `published`/`modified`/`withdrawn` timestamps vs your last scan. Returns only the delta — what changed. |
| `feed_status` | Checks OSV API reachability, reports last scan summary, and warns if data is stale. |
| `wp_scan` | WordPress core/plugin/theme vulnerability lookup via the [WPScan](https://wpscan.com/api/) API with 24-hour SQLite caching. Requires `WPSCAN_TOKEN` env var. |

## Install

```bash
npm install -g vuln-intel-ext
```

Or use directly with npx:

```bash
npx vuln-intel-ext
```

## Configure

Add both `cve-mcp` and `vuln-intel-ext` to your MCP client config (e.g. Claude Desktop, Cursor, etc.):

```json
{
  "mcpServers": {
    "cve-mcp": {
      "command": "npx",
      "args": ["cve-mcp"]
    },
    "vuln-intel-ext": {
      "command": "npx",
      "args": ["vuln-intel-ext"]
    }
  }
}
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `WPSCAN_TOKEN` | No | WPScan API token for WordPress vulnerability lookups. Get one at [wpscan.com/api](https://wpscan.com/api/). Without it, `wp_scan` returns an error. |
| `VULN_INTEL_DB_PATH` | No | Custom SQLite database path. Defaults to `~/.vuln-intel-ext/snapshot.db`. |

## Usage Examples

### Save a dependency snapshot

```
save_dependency_snapshot({
  packages: [
    { name: "express", version: "4.17.0", ecosystem: "npm" },
    { name: "requests", version: "2.31.0", ecosystem: "PyPI" }
  ],
  label: "production"
})
```

### Check what's new since last scan

```
diff_since({
  since: "2025-01-01T00:00:00Z"
})
```

Returns:

```json
{
  "since": "2025-01-01T00:00:00Z",
  "scannedAt": "2026-07-23T16:25:52Z",
  "packagesChecked": 1,
  "newVulnerabilities": [],
  "modifiedVulnerabilities": [
    {
      "id": "GHSA-rv95-896h-c2vc",
      "cve": "CVE-2024-29041",
      "summary": "Express.js Open Redirect in malformed URLs",
      "severity": "MEDIUM",
      "package": "express",
      "ecosystem": "npm"
    }
  ],
  "withdrawnAdvisories": [],
  "summary": {
    "newCount": 0,
    "modifiedCount": 2,
    "withdrawnCount": 0,
    "worstSeverity": "MEDIUM"
  }
}
```

### Check feed freshness

```
feed_status({})
```

### WordPress vulnerability lookup

```
wp_scan({
  type: "plugin",
  name: "elementor",
  version: "3.20.0"
})
```

## Architecture

- **Data source:** [OSV.dev](https://osv.dev) — 30+ ecosystems (npm, PyPI, Go, Rust, RubyGems, Maven, NuGet, Packagist, Debian, Alpine, Ubuntu, and more). No API key required. No rate limits.
- **Caching:** SQLite with WAL mode at `~/.vuln-intel-ext/snapshot.db`. Stores dependency snapshots, scan history, and WPScan cache.
- **Enrichment:** The OSV batch endpoint returns minimal records. `diff_since` fetches full vulnerability details in parallel via `GET /v1/vulns/{id}` for rich summaries, CVE aliases, and severity scores.
- **Batch size:** Queries are chunked at 1000 packages per request to stay within OSV's recommended limits.

## Build from source

```bash
git clone https://github.com/Chykalophia/vuln-intel-ext.git
cd vuln-intel-ext
npm install
npm run build
npm start
```

## License

MIT - see [LICENSE](LICENSE)

## Acknowledgments

- [cve-mcp](https://github.com/badchars/cve-mcp) — the base vulnerability intelligence server this extends
- [OSV.dev](https://osv.dev) — the open source vulnerability database that powers `diff_since`
- [WPScan](https://wpscan.com) — WordPress vulnerability data for `wp_scan`
