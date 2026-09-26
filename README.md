# Agentic_AI_Chatbot_Security

This project demonstrates two versions of an AI customer support application that uses MCP tools with a LangChain agent:

- `Ai_Agent_demo`: an MCP server with no authentication or per-tool authorization.
- `secure-ai-agent`: an MCP server protected with Descope-issued JWTs and explicit scope checks on sensitive tools.

The app is intentionally small and uses an in-memory mock support database so the difference between insecure and secure tool access is easy to see.

## What This Project Shows

The support agent can use MCP tools to answer customer support questions, look up customer data, look up orders, get product information, and process refunds.

The unsecure version exposes tools directly. Any client that can reach the MCP endpoint can call the tools exposed by the server.

The secure version protects the MCP endpoint with Descope JWT authentication. Tools require scopes, and the refund tool requires the additional `refunds:create` scope.

## Project Structure

```text
.
├── pyproject.toml
├── uv.lock
├── Ai_Agent_Demo/
│   ├── api/index.py
│   ├── customer_support_agent.py
│   ├── mcp_server.py
│   └── tools.py
└── secure-ai-agent/
    ├── api/index.py
    ├── customer_support_secure.py
    ├── secure_mcp_server.py
    └── tools.py
```

## Main Components

### MCP Servers

`Ai_Agent_demo/mcp_server.py` creates a FastMCP server named `Support Tools Server` and exposes these tools:

- `answer_user_query`
- `process_refund`
- `get_order_id`
- `lookup_customer_info`
- `list_customers`
- `get_product_info`

`secure-ai-agent/secure_mcp_server.py` creates a FastMCP server named `Scoped Support Tools Server` and protects it with Descope JWT verification. It exposes these scoped tools:

- `answer_user_query`: requires `support:tools`
- `process_refund`: requires `support:tools` and `refunds:create`
- `get_order_id`: requires `support:tools`
- `lookup_customer_info`: requires `support:tools`
- `get_product_info`: requires `support:tools`

The secure version intentionally does not expose `list_customers`, which limits broad customer data access.

### Agent Clients

`Ai_Agent_demo/customer_support_agent.py` connects to an MCP server using `MCP_SERVER_URL`, loads the available tools, and runs a LangChain/Groq support agent.

`secure-ai-agent/customer_support_secure.py` gets or uses a Descope access token, connects to the secure MCP server with an `Authorization: Bearer <token>` header, loads the scoped tools, and runs a LangChain/Groq support agent.

### Mock Data

Both `tools.py` files use in-memory dictionaries for customers, orders, refunds, and products. Data resets when the server process restarts.

## Requirements

- Python 3.11 or newer
- `uv` for dependency installation and running scripts
- A Groq API key for the LangChain chat model
- A Descope project and MCP application/server configuration for the secure version
- Vercel account for deployment

Use this Descope setup link for this project:

https://descope.plug.dev/RPSqEGs

## Install Dependencies

From the repository root:

```bash
uv sync
```

This installs the dependencies from `pyproject.toml` and `uv.lock`.

## Local Environment Variables

Create a local `.env` file in the repository root for local runs.

Do not commit real secrets. If secrets have already been committed, rotate them in the provider dashboard.

### Shared Local Variables

```env
GROQ_API_KEY=your_groq_api_key
MCP_SERVER_URL=http://localhost:8000/mcp
```

`GROQ_API_KEY` is used by `ChatGroq`.

`MCP_SERVER_URL` is used by the unsecure client to connect to the MCP server. For a deployed server, set this to the Vercel `/mcp` URL.

### Secure Local Variables

The secure MCP server requires these values:

```env
DESCOPE_PROJECT_ID=your_descope_project_id
DESCOPE_MCP_SERVER_ID=your_descope_mcp_server_or_audience_id
```

Optional secure server overrides:

```env
DESCOPE_BASE_URL=https://api.descope.com
DESCOPE_JWKS_URI=https://api.descope.com/<DESCOPE_PROJECT_ID>/.well-known/jwks.json
DESCOPE_ISSUER=https://api.descope.com/<DESCOPE_PROJECT_ID>
```

The secure client can also use:

```env
DESCOPE_ACCESS_TOKEN=existing_access_token_for_testing
DESCOPE_TOKEN_ENDPOINT=https://your-token-endpoint
```

`DESCOPE_ACCESS_TOKEN` skips the client credentials request and uses the provided token directly.

`DESCOPE_TOKEN_ENDPOINT` overrides discovery from the OpenID configuration URL.

## Run Locally

### Run The Unsecure MCP Server

```bash
uv run python Ai_Agent_demo/mcp_server.py
```

The server exposes the MCP app at `/mcp`.

### Run The Unsecure Agent Client

In another terminal, set `MCP_SERVER_URL` to the running MCP endpoint, then run:

```bash
uv run python Ai_Agent_demo/customer_support_agent.py
```

### Run The Secure MCP Server

Set the required Descope variables first:

```bash
export DESCOPE_PROJECT_ID=your_descope_project_id
export DESCOPE_MCP_SERVER_ID=your_descope_mcp_server_or_audience_id
uv run python secure-ai-agent/secure_mcp_server.py
```

The server exposes the protected MCP app at `/mcp`.

### Run The Secure Agent Client

Make sure the secure client has a valid Descope token flow or a `DESCOPE_ACCESS_TOKEN`, then run:

```bash
uv run python secure-ai-agent/customer_support_secure.py
```

## Deploy To Vercel

The Vercel entrypoint for each server is the `api/index.py` file in that server directory:

- `Ai_Agent_demo/api/index.py` imports `app` from `mcp_server.py`.
- `secure-ai-agent/api/index.py` imports `app` from `secure_mcp_server.py`.

Deploy the secure and unsecure servers as separate Vercel projects so each deployment has its own root directory and environment variables.

Recommended setup:

- Project 1 root directory: `Ai_Agent_demo`
- Project 2 root directory: `secure-ai-agent`
- Production endpoint path for both: `/mcp`

If Vercel does not automatically install dependencies from the root `pyproject.toml`, add a `requirements.txt` or configure the Vercel install command for the selected project.

## Vercel Environment Variables

Set environment variables in Vercel from:

`Vercel Project -> Settings -> Environment Variables`

Add variables to the correct environments, usually `Production`, `Preview`, and `Development` if you use all three.

### Required For The Secure Vercel MCP Server

These variables must be set on the Vercel project that deploys `secure-ai-agent`:

| Variable | Required | Example | Description |
| --- | --- | --- | --- |
| `DESCOPE_PROJECT_ID` | Yes | `Pxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Descope project ID used to build the default issuer and JWKS URL. |
| `DESCOPE_MCP_SERVER_ID` | Yes | `RSxxxxxxxxxxxxxxxxxxxxxxxxxx` | Expected JWT audience for this MCP server. This must match the audience/resource configured in Descope. |
| `DESCOPE_BASE_URL` | No | `https://api.descope.com` | Base Descope API URL. Defaults to `https://api.descope.com`. |
| `DESCOPE_JWKS_URI` | No | `https://api.descope.com/<project-id>/.well-known/jwks.json` | JWKS URL used to verify token signatures. Defaults from `DESCOPE_BASE_URL` and `DESCOPE_PROJECT_ID`. |
| `DESCOPE_ISSUER` | No | `https://api.descope.com/<project-id>` | Expected JWT issuer. Defaults from `DESCOPE_BASE_URL` and `DESCOPE_PROJECT_ID`. |

The secure Vercel server will fail at startup if `DESCOPE_PROJECT_ID` or `DESCOPE_MCP_SERVER_ID` is missing.

### Required For The Unsecure Vercel MCP Server

The unsecure MCP server does not currently require any environment variables on Vercel.

| Variable | Required | Description |
| --- | --- | --- |
| None | No | The server uses only the mock in-memory data from `tools.py`. |

### Variables For Running Agent Clients Against Vercel

These are needed wherever you run the client scripts. They are not required by the MCP server itself unless you also deploy the client logic.

| Variable | Used By | Required | Description |
| --- | --- | --- | --- |
| `GROQ_API_KEY` | Both clients | Yes | API key used by `ChatGroq`. |
| `MCP_SERVER_URL` | Unsecure client | Yes | Full MCP endpoint, for example `https://your-unsecure-project.vercel.app/mcp`. |
| `DESCOPE_ACCESS_TOKEN` | Secure client | No | Existing bearer token. Useful for testing without requesting a new token. |
| `DESCOPE_TOKEN_ENDPOINT` | Secure client | No | Overrides token endpoint discovery. |

The secure client also needs Descope client credentials and the well-known configuration URL. These values are loaded from environment variables and should be set locally in `.env` or in the environment where the client runs.

Recommended production secure-client variables:

```env
DESCOPE_WELL_KNOWN_URL=https://api.descope.com/v1/apps/agentic/<project-id>/<server-id>/.well-known/openid-configuration
DESCOPE_REQUIRED_SCOPES=support:tools refunds:create
DESCOPE_TOKEN_RESOURCE=https://your-secure-project.vercel.app/mcp
DESCOPE_CLIENT_ID=your_descope_client_id
DESCOPE_CLIENT_SECRET=your_descope_client_secret
```

## Descope Scope Configuration

Use this Descope setup link for the project configuration:

https://descope.plug.dev/RPSqEGs

The secure MCP server expects JWTs that satisfy these checks:

- Token signature validates against the configured JWKS URI.
- Token issuer matches `DESCOPE_ISSUER`.
- Token audience matches `DESCOPE_MCP_SERVER_ID`.
- Token includes `support:tools` for normal support tools.
- Token includes both `support:tools` and `refunds:create` for `process_refund`.

When creating the Descope application or machine-to-machine client, make sure the issued token includes the scopes required for the tools you want the agent to call.

## Security Notes

- Never commit real API keys, client secrets, access tokens, or passwords.
- Store production secrets in Vercel Environment Variables.
- Rotate any secret that was accidentally committed or shared.
- Use the secure MCP server for sensitive operations such as refunds.
- Keep high-risk tools behind narrow scopes like `refunds:create`.
- Avoid exposing broad tools such as `list_customers` unless the caller is explicitly authorized.

## Troubleshooting

### `Missing required environment variable: DESCOPE_PROJECT_ID`

Set `DESCOPE_PROJECT_ID` in Vercel or in your local shell before starting `secure-ai-agent/secure_mcp_server.py`.

### `Missing required environment variable: DESCOPE_MCP_SERVER_ID`

Set `DESCOPE_MCP_SERVER_ID` in Vercel or in your local shell. This value must match the expected JWT audience.

### `401 Unauthorized` Or Tool Calls Fail On The Secure Server

Check that the token was issued by the configured Descope issuer, uses the correct audience, and includes the required scopes.

### Refund Tool Is Not Available

The access token probably does not include `refunds:create`. Add that scope in Descope and request it when getting the token.

### Groq Authentication Fails

Set `GROQ_API_KEY` in the environment where the client script runs.

## Useful Commands

```bash
uv sync
uv run python Ai_Agent_demo/mcp_server.py
uv run python Ai_Agent_demo/customer_support_agent.py
uv run python secure-ai-agent/secure_mcp_server.py
uv run python secure-ai-agent/customer_support_secure.py
```
