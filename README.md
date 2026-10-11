# Ashik's Calendar

A small [Model Context Protocol](https://modelcontextprotocol.io) server that looks up events on a Google Calendar for a given date.

An MCP client (such as Cursor) starts this process and talks to it over stdio. The server exposes one tool, `getMyCalendarDataByDate`.

## What the tool does

`getMyCalendarDataByDate` takes a `date` string and returns events for that UTC day:

- The date must parse with `Date.parse`.
- The window is midnight UTC through the next midnight.
- Up to 10 events are returned, ordered by start time, as `"title at start time"`.
- An empty list means nothing is scheduled that day.
- API failures come back as `{ "error": "..." }`.

## Setup

```bash
npm install
```

Create a `.env` file in the project root (it is gitignored):

```
GOOGLE_PUBLIC_API_KEY=your_google_api_key
CALENDAR_ID=your_calendar_id
```

The calendar must be readable with that API key. A public calendar works with a public API key.

## Run

```bash
npm start
```

That runs `node server.js`. The process waits on stdin for MCP messages, so it looks idle if you start it in a terminal by itself. Point your MCP client at this folder's `server.js` instead.

## Setup flow

```mermaid
flowchart TD
    A[Open the project] --> B[Run npm install]
    B --> C[Create a .env file]
    C --> D["Set GOOGLE_PUBLIC_API_KEY and CALENDAR_ID"]
    D --> E[Register the server in the MCP client]
    E --> F["Client starts node server.js"]
    F --> G[Server loads .env]
    G --> H["Create MCP server Ashik's Calendar"]
    H --> I[Register getMyCalendarDataByDate]
    I --> J[Connect over stdio and wait for the client]
```

## How it talks to an MCP client

The client owns the process. It starts `server.js`, then sends JSON-RPC messages on stdin and reads replies from stdout. This server does not open a port.

```mermaid
sequenceDiagram
    actor User
    participant Client as MCP client
    participant Server as Ashik's Calendar
    participant Google as Google Calendar API

    User->>Client: Ask for events on a date
    Client->>Server: Spawn process and send initialize
    Server-->>Client: Name, version, and tool capability
    Client->>Server: notifications/initialized
    Client->>Server: tools/list
    Server-->>Client: getMyCalendarDataByDate schema
    Client->>Server: tools/call with date
    alt Date does not parse
        Server-->>Client: Validation error
    else Date is valid
        Server->>Google: events.list for that UTC day
        Google-->>Server: Up to 10 events, or an API error
        Server-->>Client: Text content with meetings JSON
    end
    Client-->>User: Show the meetings
```

## Tech stack

- Node.js (ES modules)
- [@modelcontextprotocol/sdk](https://modelcontextprotocol.io) — MCP server and stdio transport
- [googleapis](https://github.com/googleapis/google-api-nodejs-client) — Google Calendar v3
- [zod](https://zod.dev) — tool input validation
- [dotenv](https://github.com/motdotla/dotenv) — loads `.env`
