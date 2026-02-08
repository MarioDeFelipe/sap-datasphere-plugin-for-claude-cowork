# SAP Datasphere Plugin for Claude

Connect Claude to your SAP Datasphere tenant for hands-free data exploration, modeling, integration, and management. This plugin includes 13 specialized skills covering every major Datasphere workflow — from browsing spaces and designing views to troubleshooting data flows and publishing data products. Powered by a production-grade MCP server with 45 tools, OAuth 2.0 authentication, and enterprise-level security including SQL sanitization and PII filtering.

## Skills

### Exploration & Discovery

| Skill | Description |
|-------|-------------|
| **datasphere-explorer** | Guided exploration and discovery — browse spaces, search the catalog, inspect schemas, profile data quality, trace lineage, and build queries interactively |

### Data Modeling

| Skill | Description |
|-------|-------------|
| **datasphere-view-architect** | Design Graphical and SQL views with proper semantic usage (Fact, Dimension, Text, Hierarchy), associations, persistence strategies, and performance optimization |
| **datasphere-analytic-model-creator** | Create Analytic Models for SAP Analytics Cloud with measures, dimensions, variables, currency conversion, and exception aggregation |
| **datasphere-intelligent-lookup** | Configure fuzzy matching and intelligent lookups for data harmonization across sources — matching strategies, threshold tuning, and review workflows |

### Data Integration

| Skill | Description |
|-------|-------------|
| **datasphere-data-flows** | Orchestrate replication flows (with CDC), data flows (visual ETL), transformation flows (SQL-based delta), and task chains |
| **datasphere-transformation-logic** | Generate and validate SQLScript and Python transformations — SCD Type 2, deduplication, pivoting, delta handling patterns |
| **datasphere-s4hana-import** | Import entities from SAP S/4HANA and BW/4HANA — CDS views, ODP extractors, Cloud Connector setup, and delta extraction |
| **datasphere-connections** | Create and manage 35+ connection types including SAP S/4HANA, BigQuery, Redshift, Kafka, and generic JDBC/OData connectors |

### Administration & Governance

| Skill | Description |
|-------|-------------|
| **datasphere-admin** | Space management, user and role administration, system monitoring, capacity planning, and transport operations |
| **datasphere-data-product-publisher** | Publish data products through the Data Sharing Cockpit — product descriptions, license terms, visibility settings, and marketplace management |
| **datasphere-transport-manager** | Manage CSN/JSON transport packages — dependency checking, export/import workflows, conflict resolution, and Content Network integration |

### Monitoring & Troubleshooting

| Skill | Description |
|-------|-------------|
| **datasphere-flow-doctor** | Diagnose and resolve errors in Data Flows, Replication Flows, and Transformation Flows — error catalogs, root cause analysis, and fix recommendations |
| **datasphere-performance-optimizer** | Analyze and optimize performance — View Analyzer, Explain Plans, persistence strategies, partitioning, storage tiering, and query tuning |

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
- *"Help me design a view for customer analytics"*
- *"Create an analytic model with revenue measures"*
- *"Import CDS views from our S/4HANA system"*
- *"My replication flow is failing — help me diagnose"*
- *"Optimize this slow-running view"*
- *"Set up a transport package for production deployment"*
- *"Publish our sales data as a data product"*
- *"Generate SCD Type 2 logic for the customer dimension"*

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
