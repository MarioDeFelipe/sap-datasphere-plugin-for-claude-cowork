# SAP Datasphere Plugin for Claude

Connect Claude to your SAP Datasphere tenant for hands-free data exploration and management. Browse spaces, search the data catalog, inspect table schemas, profile data quality, and build queries — all through natural conversation. The plugin includes four specialized skills: an Explorer for guided data discovery, an Admin module for space and security management, a Connections manager supporting 35+ data source types, and a Data Flows orchestrator for replication, ETL, and task chains. Powered by a production-grade MCP server with 45 tools, OAuth 2.0 authentication, and enterprise-level security including SQL sanitization and PII filtering.

## Skills

| Skill | Description |
|-------|-------------|
| **datasphere-explorer** | Guided exploration and discovery — browse spaces, search the catalog, inspect schemas, profile data quality, trace lineage, and build queries interactively |
| **datasphere-admin** | Space management, user and role administration, system monitoring, capacity planning, and transport operations |
| **datasphere-connections** | Create and manage 35+ connection types including SAP S/4HANA, BigQuery, Redshift, Kafka, and generic JDBC/OData connectors |
| **datasphere-data-flows** | Orchestrate replication flows (with CDC), data flows (visual ETL), transformation flows (SQL-based delta), and task chains |

## Prerequisites

- [Claude Code](https://claude.com/claude-code) v1.0.33+ or Claude Desktop with Cowork mode
- An SAP Datasphere tenant with OAuth 2.0 client credentials
- Node.js 18+ (for the MCP server)

## Installation

Install from a marketplace or load directly:

```bash
claude --plugin-dir ./sap-datasphere-plugin-for-claude-cowork
```

## Configuration

After installation, configure your SAP Datasphere connection by setting the environment variables in `.mcp.json`:

| Variable | Description | Example |
|----------|-------------|---------|
| `DATASPHERE_BASE_URL` | Your tenant URL | `https://mytenant.eu20.hcs.cloud.sap` |
| `DATASPHERE_CLIENT_ID` | OAuth client ID | From SAP BTP cockpit |
| `DATASPHERE_CLIENT_SECRET` | OAuth client secret | From SAP BTP cockpit |
| `DATASPHERE_TOKEN_URL` | OAuth token endpoint | `https://mytenant.authentication.eu20.hana.ondemand.com/oauth/token` |
| `DATASPHERE_AUTH_URL` | OAuth auth endpoint | `https://mytenant.authentication.eu20.hana.ondemand.com` |

### Setting up OAuth credentials in SAP BTP

1. Open the SAP BTP Cockpit for your subaccount
2. Navigate to **Security > Instances and Subscriptions**
3. Create a service instance for SAP Datasphere with the appropriate scopes
4. Create a service key to obtain your client ID and secret

## Usage

Once configured, just talk to Claude naturally:

- *"What spaces do we have in Datasphere?"*
- *"Show me the tables in the SALES space"*
- *"Find data about customer revenue"*
- *"What columns does the ORDERS table have?"*
- *"Profile the data quality of the REGION column"*
- *"Create a connection to our S/4HANA system"*
- *"Set up a replication flow for sales orders"*

## MCP Server

This plugin uses the [`@mariodefe/sap-datasphere-mcp`](https://www.npmjs.com/package/@mariodefe/sap-datasphere-mcp) MCP server, which provides 45 tools covering:

- **Foundation**: Connection testing, user info, tenant config
- **Catalog & Discovery**: Space browsing, asset search, marketplace
- **Schema & Metadata**: Table schemas, OData metadata, analytical models
- **Data Query**: Smart queries, SQL execution, OData queries
- **Data Profiling**: Column distribution analysis, cross-asset column search
- **Repository & Lineage**: Object search, lineage tracing, deployment status
- **Database Users**: Full CRUD for space-level database users

## Security

The MCP server includes enterprise-grade security:

- OAuth 2.0 with automatic token refresh
- RBAC-based authorization enforcement
- SQL injection prevention and query sanitization
- PII redaction and credential masking
- Input validation on all tool parameters

## License

MIT
