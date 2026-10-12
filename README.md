# BalenaMCP

A simple MCP server that wraps the `balena` CLI to provide Balena capabilities
to supported clients.

## Proposed fork roadmap

This public fork proposes an API-native MCP implementation with optional
documentation and client integrations. This section is a proposal, not a list
of available features: the current Go server, CLI prerequisite, and stdio
transport remain unchanged. No TypeScript package, documentation site, remote
endpoint, or Marketplace listing is introduced by this proposal.

### API-native MCP core

- Build a small TypeScript core around the
  [Balena API](https://docs.balena.io/reference/api/overview.md), using validated
  inputs, structured results, pagination, request deadlines, and explicit
  error handling instead of CLI subprocesses and terminal-output parsing.
- Start with read-only fleet, device, release, and tag capabilities. Verify
  API coverage for OS versions and bounded device logs before promising
  parity with the existing tools. CLI-specific version and help options
  need an explicit migration decision rather than an invented API equivalent.
- Add an authenticated Streamable HTTP endpoint on Cloudflare Workers using
  the [stateless MCP handler](https://developers.cloudflare.com/agents/model-context-protocol/apis/handler-api/).
  Ordinary Workers cannot run the Go executable or spawn the Balena CLI.
- Keep the existing Go/stdio implementation available until the replacement
  has passed compatibility checks and any tool-contract changes are documented.
  The API-native implementation would not require Go, the Balena CLI, local
  CLI login state, or a specific desktop client.

### Supporting integrations

- **Astro/Starlight: optional documentation and onboarding.** Publish setup,
  permissions, client configuration, and troubleshooting guides as a static
  site using [Starlight](https://starlight.astro.build/). Host it on
  [Workers Static Assets](https://developers.cloudflare.com/workers/static-assets/)
  or Cloudflare Pages. The site is not the MCP server; server-rendered
  onboarding is a separate feature, and credentials must never enter static
  assets or browser bundles.
- **Node.js: development tooling and an optional local stdio adapter.** Use a
  supported Node.js LTS release for development, builds, and local package
  execution. Share tool and API logic between the stdio adapter and Worker
  transport. A hosted Node server and Claude Desktop are not requirements
  for the Worker endpoint; document supported MCP clients after testing them.
- **Containers: not required for the API-native design.** Do not add Docker
  as a prerequisite. Container hosting remains an alternative for retaining
  the legacy CLI wrapper, not a dependency of the proposed Worker. Local
  device provisioning, image builds, and long-running CLI operations remain
  outside the edge runtime.
- **GitHub Marketplace: optional later.** Distribute versioned packages or
  releases and publish discovery metadata to the
  [MCP Registry](https://modelcontextprotocol.io/registry/about) when those
  artifacts or endpoints actually exist. A
  [Marketplace listing](https://docs.github.com/en/apps/github-marketplace/github-marketplace-overview/about-github-marketplace-for-apps)
  would require a useful companion Action or App; it neither hosts the server
  nor makes a standalone MCP package a Marketplace integration.

### Authorization and licensing boundaries

Balena access still requires an account and a credential. Named
[Balena API keys](https://docs.balena.io/learn/accounts/account.md#access-tokens)
inherit user-level permissions; exposing only read tools does not reduce the
underlying key's privileges.

Start with a self-deployable, authenticated Worker using the operator's own
credential. Keep credentials server-side, redact sensitive output, and enforce
resource allowlists, finite result sizes, and request limits. A centrally
operated service requires a separate design for
[MCP-compatible authorization](https://developers.cloudflare.com/agents/model-context-protocol/protocol/authorization/),
per-user credential isolation, encrypted storage, revocation, and support.
Public availability must not expose fleet data anonymously or share one
administrator credential across customers. Write operations are deferred.

Preserve upstream attribution. The current README states MIT, but this tree
does not include a standalone license notice; clarify the grant and required
copyright notice before publishing new derivatives. Do not incorporate other
implementations under incompatible terms or replace inherited licensing.

### Delivery and acceptance

1. Confirm licensing and define the read-only tool contracts and migration
   behavior before introducing the TypeScript runtime.
2. Implement and test the shared API core, authenticated Worker transport, and
   optional stdio adapter. Exercise initialization, tool discovery and calls,
   invalid input, authorization failures, redaction, pagination, and timeouts.
   Test the actual Workers runtime, not only Node.js.
3. Verify reads against an explicitly authorized Balena test fleet. Add
   client setup guides and the optional Astro/Starlight site after the
   integrations they describe work.
4. Publish verified artifacts and a self-deployment guide, then MCP Registry
   metadata. Consider a managed endpoint or a companion Marketplace
   integration only as separately reviewed follow-up work.

This roadmap does not enable CI workflows, add secrets, publish releases,
deploy infrastructure, or remove the current implementation.

## Prerequisites

- Go 1.21 or later
- `balena` CLI installed and available in PATH
- Claude Desktop installed

## Installation

```bash
# Clone the repository
git clone https://github.com/klutchell/balenamcp.git
cd balenamcp

# Install dependencies
go mod download

# Build the binary
go build -o bin/balenamcp
```

## Configuration

### Claude Desktop Setup

1. Open Claude Desktop
2. Go to the Claude menu and select "Settings..."
3. Click on "Developer" in the left-hand bar

   You may need to enable Developer Mode to see this option.

4. Click on "Edit Config"

   This will create or open a configuration file at:

   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`

5. Add the following configuration to the file:

   ```json
   {
     "mcpServers": {
       "balenamcp": {
         "command": "/full/path/to/your/balenamcp/bin/balenamcp",
         "args": []
       }
     }
   }
   ```

   Make sure to replace `/full/path/to/your/balenamcp` with the absolute path to
   your project directory.

6. Save the file and restart Claude Desktop

### Authentication

Before using the tools, make sure you're authenticated with Balena:

```bash
balena login
```

## Available MCP Commands

The following commands are available through BalenaMCP:

- `device-list`: List all devices
- `device-logs`: Show device logs (requires device UUID or name)
- `fleet-list`: List all fleets
- `os-versions`: Show available balenaOS versions for a device type (requires
  device type)
- `release-list`: List all releases of a fleet (requires fleet name)
- `release-info`: Get info for a release (requires release ID)
- `version`: Display version information for the balena CLI
- `tag-list`: List all tags for a fleet, device or release (requires resource
  type and name)

Each command can be accessed directly through Claude Desktop.

## Development

The server is built using the
[MCP Go library](https://github.com/mark3labs/mcp-go) and communicates with
Claude Desktop through standard input/output streams.

### Project Structure

- `main.go`: Main server implementation with tool definitions
- `server/setup.go`: Server setup and tool implementations
- `main_test.go`: Tests for the server implementation
- `bin/`: Built binaries (gitignored)

### Building

```bash
# Build for current platform
go build -o bin/balenamcp

# Build for specific platforms
GOOS=linux GOARCH=amd64 go build -o bin/balenamcp-linux-amd64
GOOS=windows GOARCH=amd64 go build -o bin/balenamcp-windows-amd64.exe
GOOS=darwin GOARCH=amd64 go build -o bin/balenamcp-darwin-amd64
GOOS=darwin GOARCH=arm64 go build -o bin/balenamcp-darwin-arm64
```

### Running Tests

```bash
go test ./...
```

## Troubleshooting

If you encounter issues with the MCP server:

1. Check the logs in:

   - macOS: `~/Library/Logs/Claude/mcp-server-balenamcp.log`
   - Windows: `%APPDATA%\Claude\logs\mcp-server-balenamcp.log`

2. Make sure the path to the balenamcp binary is correct and absolute
3. Ensure you have the necessary permissions to execute the binary
4. Verify that balena is installed and accessible in your PATH

## License

MIT
