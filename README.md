# Weather MCP Server by Meteosource

Weather data for AI agents. Meteosource hosts the server at

```
https://mcp.meteosource.com/mcp
```

It speaks the Model Context Protocol over Streamable HTTP. Connect it to Claude Code, Claude
Desktop, Codex, Antigravity or any other MCP client, send your Meteosource API key in a header,
and the agent can look up forecasts, official weather warnings, air quality, twenty years of
historical weather and the production of PV systems and wind farms.

There is nothing to install and nothing to host. The server keeps no key of its own: every
request is served on the key it carries, within the plan that key is on.

## Tools

| Tool | What it does |
|---|---|
| `search_places` | Turns a place name, a postal code or coordinates into a `place_id` for the other tools |
| `get_forecast` | Current conditions, hourly (up to 168 h) and daily (up to 30 days) forecast, minute-by-minute precipitation and warnings for up to 20 locations in one call; a `variables` filter keeps the answer small |
| `get_alerts` | Official severe-weather warnings for up to 20 locations; returns only the locations that have one. USA, EU and Canada |
| `get_weather_history` | Past weather, hourly (up to 14 days per call), daily or monthly (up to 366 days per call), aggregated on the server. `normals=true` adds long-term normals and records; `wind_distribution` gives hours per wind-speed bin for turbine siting |
| `get_air_quality` | Hourly air quality about four days ahead: index, PM10, PM2.5, NO₂, SO₂, O₃ and more |
| `get_solar_production` | Output of a PV system, given its location, module and inverter power, tilt and orientation: power per hour, energy per day or month, from 20 years back to four days ahead |
| `get_wind_production` | Output of a wind farm, given its location, turbine model, count or capacity, hub height and terrain: power per hour, energy per day or month, from 20 years back to four days ahead |

The server tells the agent how the tools fit together, so a question like "will it rain in
Brno tomorrow" ends up as a `search_places` call followed by `get_forecast` without you
spelling that out.

## Before you connect

You need a Meteosource API key. Sign up at [meteosource.com](https://www.meteosource.com)
(the free plan is enough to try the forecast tools) and copy the key from your account.

The key travels with every request in the `Authorization` header as `Bearer <key>`. Each setup
below comes down to two values: the URL above and that header.

The rest the server works out itself. It asks Meteosource which plan the key is on and serves
the tools that plan includes; a tool outside the plan answers with a sentence saying so. See
[Plans and limits](#plans-and-limits).

## Connect

### Claude Code

One command:

```bash
claude mcp add --transport http meteosource https://mcp.meteosource.com/mcp \
  --header "Authorization: Bearer YOUR_METEOSOURCE_API_KEY"
```

This registers the server for the current project. Add `--scope user` to have it in every
project. `claude mcp list` shows whether it connected; inside a session, `/mcp` lists the tools.

To share the setup with a team without sharing the key, commit a `.mcp.json` at the project
root and let everyone keep their key in an environment variable:

```json
{
  "mcpServers": {
    "meteosource": {
      "type": "http",
      "url": "https://mcp.meteosource.com/mcp",
      "headers": {
        "Authorization": "Bearer ${METEOSOURCE_API_KEY}"
      }
    }
  }
}
```

### Claude Desktop and claude.ai

Connectors belong to your account, so one added on the web is also there in the desktop app
and the other way round.

1. Open **Customize > Connectors** (in the desktop app: **Settings > Connectors**) and click
   **Add custom connector**.
2. Name it `Meteosource`; the URL is `https://mcp.meteosource.com/mcp`.
3. Under **Authentication** choose **No sign-in**.
4. Under **Request headers** add `Authorization` with the value
   `Bearer YOUR_METEOSOURCE_API_KEY`. Type the word `Bearer` and the space yourself: Claude
   sends the value exactly as you enter it.
5. Click **Add**. In a chat, open the **+** menu, choose **Connectors** and switch Meteosource on.

On Team and Enterprise plans an Owner adds the connector under **Organization settings >
Connectors** and members connect to it from **Customize > Connectors**. The free plan allows
one custom connector.

### Codex

Codex CLI, the Codex app and the IDE extension read one file, `~/.codex/config.toml`, so the
server is configured once for all three. From a terminal:

```bash
codex mcp add meteosource --url https://mcp.meteosource.com/mcp \
  --bearer-token-env-var METEOSOURCE_API_KEY
```

and export `METEOSOURCE_API_KEY` in your shell profile. The section this writes to the file:

```toml
[mcp_servers.meteosource]
url = "https://mcp.meteosource.com/mcp"
bearer_token_env_var = "METEOSOURCE_API_KEY"
```

The Codex app has its own dialog (**Settings > MCP servers > Add server**, type
**Streamable HTTP**), but no field for the key. If the app shows the server as failed, it did
not get the variable: an app started from the Dock or the Start menu does not see what you
export in a terminal. Put the key in the file instead:

```toml
[mcp_servers.meteosource]
url = "https://mcp.meteosource.com/mcp"
http_headers = { Authorization = "Bearer YOUR_METEOSOURCE_API_KEY" }
```

Codex gives a tool call 60 seconds by default. The first request for a year of archive at a
location sends 365 requests to Meteosource and can take longer than that, so give this server
more time:

```toml
tool_timeout_sec = 600
```

Restart Codex after editing the file (in the app, **Restart** on the MCP servers page).
`codex mcp list` shows the result; `/mcp` in a session lists the connected servers.

### Antigravity

The IDE and the CLI (`agy`) share one configuration file: `~/.gemini/config/mcp_config.json`
for all projects, or `.agents/mcp_config.json` inside a project. Add:

```json
{
  "mcpServers": {
    "meteosource": {
      "serverUrl": "https://mcp.meteosource.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_METEOSOURCE_API_KEY"
      }
    }
  }
}
```

The key is written out literally; Antigravity does not expand environment variables in this
file. If you use the project-level file, keep it out of version control.

In the IDE the same file is reachable from the agent panel: the **…** menu, **MCP Servers**,
**Manage MCP Servers**, **View raw config**. In the CLI, `/mcp` opens the MCP manager, which
shows the connection status and reloads the configuration.

### Other clients

Any client that supports MCP over Streamable HTTP works with the same two values, the URL and
the header. The server also accepts the key as `X-API-Key: <key>`, for clients that have a
field for an API key but not for a whole header.

## Try it

- What's the weather in Brno this weekend?
- Are there any weather warnings for Prague, Vienna and Munich right now?
- Compare last January in Berlin with the long-term normal, day by day.
- How many hours of wind above 10 m/s did 49.2, 16.6 see in 2025, month by month?
- Air quality in Kraków for the next two days, hour by hour.
- How much would a 10 kWp south-facing roof at 50.08, 14.43 have produced last week? (Renewables plan)

## Plans and limits

One key is on one Meteosource plan, and the plan decides what the server serves. The weather
tools are for the Free, Startup and Standard plans; what each plan includes, and how far back
its archive reaches, is on the [pricing page](https://www.meteosource.com/pricing). The
production tools need the Renewables plan, which in turn has no weather tools.

Days older than the plan's archive need Meteosource's historical add-on. Without it the API
refuses such a day with "not allowed to access historical data this old"; the tool reports that
for the location concerned and the rest of the call succeeds, so start inside the plan's range.

Per call: up to 20 locations; hourly archive and production up to 14 days, daily or monthly up
to 366 days; at most 400 location-days (locations × days) in one call, so a long analysis is
several calls. Each location, and each archive or production day, is one request against the
key's quota, at the rate limit of its plan. The server keeps a cache, so repeating a call soon
afterwards usually costs little or nothing. Cached data is cleared after a while, so a repeat
after a longer gap counts against the quota again.

## What the agent receives

Time series come back as tables, `columns` once and `rows` as arrays, with nested fields
flattened (`wind_speed`, `probability_storm`, `cloud_cover_total`). That is four to seven times
fewer tokens than the raw API: a week of hourly forecast is about 12 kB, a year of daily
archive about 16 kB. The `variables` parameter trims it further when the agent only needs a
few fields. Units default to the local convention of each location; `units="metric"` forces SI.

## Your key and your data

- Your key is used only to call Meteosource on your behalf. It is never stored in full and
  never appears in a URL.
- Caches are kept per key. Nothing fetched for one key is ever served to another.
- The service records that a call happened, which tool it used and how it went, so the service
  can be operated.
- HTTPS only. A request over plain HTTP is refused, so a key cannot be sent in the clear by
  mistake.

## Troubleshooting

The tools appear in the client without a key; only a tool call needs one. When a call fails, the
agent gets a sentence it can show you.

| The agent reports | What to do |
|---|---|
| "No Meteosource API key was supplied…" | The header did not arrive. Check its name and that the value starts with `Bearer ` (with the space). |
| "Meteosource does not know this API key…" | The key is mistyped or revoked. Copy it again from your account. |
| "This Meteosource API key is disabled…" (or blocked, revoked) | Contact Meteosource support. |
| "This API key is on the 'standard' tier, which has no production endpoints…" | The production tools need the Renewables plan. |
| "This API key is on the 'renewables' tier, which has only the production endpoints…" | The weather tools need a weather plan. |
| "Meteosource cannot verify this API key right now…" | Meteosource's key service did not answer. Try again in a few minutes. |
| "Meteosource request quota exceeded for this API key…" | The plan's request quota is used up for the period. |
| "not allowed to access historical data this old" for a location | The day is outside the plan's archive. Shorten the range or add the historical add-on. |
| "This call would need N location-days (limit 400)…" | Split the request into shorter ranges or fewer locations. |
| The call times out in the client | A first archive call over many location-days takes a while. Raise the client's timeout (Codex: `tool_timeout_sec`) or ask for a shorter range first; the second run is served from cache. |

Questions about your key or plan go to Meteosource support. For the server itself, open an
issue in this repository.

## About this repository

This repository holds the documentation of the hosted server and its entry for the
[MCP Registry](https://registry.modelcontextprotocol.io), `server.json`. The server is listed
there as [`com.meteosource/weather`](https://registry.modelcontextprotocol.io/v0.1/servers/com.meteosource%2Fweather/versions); clients and catalogs that read the registry
pick it up from that entry. The server itself runs at Meteosource; there is nothing here to
install.

## License

MIT, see [LICENSE](LICENSE).
