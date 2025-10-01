# Chess MCP Server - Architecture Documentation

## Table of Contents
1. [Introduction](#introduction)
2. [What is MCP?](#what-is-mcp)
3. [High-Level Architecture](#high-level-architecture)
4. [System Components](#system-components)
5. [Request Lifecycle](#request-lifecycle)
6. [Session Management](#session-management)
7. [Data Flow](#data-flow)
8. [UI Integration](#ui-integration)
9. [Code Walkthrough](#code-walkthrough)
10. [How Everything Works Together](#how-everything-works-together)

---

## Introduction

This document provides an in-depth explanation of the Chess MCP Server architecture. Whether you're new to Model Context Protocol (MCP) or just want to understand how this specific implementation works, this guide will walk you through every component, explain the workflow, and show you how all the pieces fit together.

The Chess MCP Server is an interactive chess game built using the Model Context Protocol. It allows AI agents (like Claude or ChatGPT) to play chess by exposing chess functionality through standardized tools, and provides an interactive web-based UI for humans to play along.

---

## What is MCP?

### The Problem MCP Solves

Imagine you want to give an AI assistant the ability to interact with external systems - like databases, APIs, or in our case, a chess game. Without a standard protocol, every integration would be custom-built, making it hard to:
- Share tools between different AI systems
- Maintain consistent behavior
- Scale to many tools

### MCP Solution

**Model Context Protocol (MCP)** is a standardized way for AI models to interact with external tools and resources. Think of it like REST APIs for AI assistants. It defines:

- **Tools**: Actions the AI can perform (like "make a chess move")
- **Resources**: Data the AI can access (like "current board state")
- **Prompts**: Pre-defined instructions the AI can use

### Key Concepts

1. **Server**: Exposes tools and resources (this is what we're building)
2. **Client**: Uses those tools (like Claude Desktop, or nanobot)
3. **Transport**: How messages travel between server and client (we use HTTP)
4. **Session**: A stateful connection that persists across multiple requests

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         MCP Client                               │
│                    (e.g., Claude, Nanobot)                       │
│                                                                   │
│  - Sends tool requests                                           │
│  - Receives tool responses                                       │
│  - Renders UI components                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ HTTP Transport
                         │ (JSON-RPC over HTTP)
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    Express.js Server                             │
│                   (http://localhost:3000)                        │
│                                                                   │
│  ┌───────────────────────────────────────────────────┐          │
│  │         MCP Middleware                            │          │
│  │  (/mcp endpoint)                                  │          │
│  │                                                    │          │
│  │  • Session Management                             │          │
│  │  • Transport Handling                             │          │
│  │  • Request Routing                                │          │
│  └──────────────────┬────────────────────────────────┘          │
│                     │                                             │
│  ┌──────────────────▼───────────────────────────────┐          │
│  │         McpServer Instance                       │          │
│  │  (Created per session)                           │          │
│  │                                                   │          │
│  │  ┌───────────────────────────────────────┐      │          │
│  │  │  Registered Tools:                    │      │          │
│  │  │  • chess_get_board_state             │      │          │
│  │  │  • chess_move                        │      │          │
│  │  └───────────────────────────────────────┘      │          │
│  └──────────────────┬───────────────────────────────┘          │
│                     │                                             │
│  ┌──────────────────▼───────────────────────────────┐          │
│  │         Game State Store                         │          │
│  │  (File System Persistence)                       │          │
│  │                                                   │          │
│  │  • Load game by session ID                       │          │
│  │  • Save game after each move                     │          │
│  │  • Creates games/ directory                      │          │
│  └──────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

                         │
                         │ File I/O
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    File System                                   │
│                   ./games/ directory                             │
│                                                                   │
│  • {session-id}.json - Stores FEN notation                      │
│  • One file per game session                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## System Components

### 1. Express Server (`src/server.ts`)

The entry point of the application. This is where everything starts.

**Purpose**: 
- Creates an HTTP server
- Registers the MCP middleware at the `/mcp` endpoint
- Listens on port 3000

**Key Code**:
```typescript
const app = express();
app.use("/mcp", Middleware(newServer));
app.listen(3000);
```

**What happens**:
1. Express starts listening on port 3000
2. Any request to `/mcp` is handled by the MCP middleware
3. The `newServer` function is passed to the middleware (more on this below)

---

### 2. MCP Server Factory (`newServer` function in `src/server.ts`)

This function creates a new MCP server instance for each session.

**Purpose**: 
- Define what tools the server exposes
- Implement the business logic for each tool
- Configure server metadata (name, version)

**Why a factory function?**
Each client session gets its own isolated MCP server instance. This allows:
- Session-specific state
- Independent tool execution
- Clean lifecycle management

**Key Code Structure**:
```typescript
function newServer() {
  // 1. Create server instance
  const server = new McpServer({
    name: "Chess MCP Server",
    version: "0.0.1",
  });

  // 2. Register tool: chess_get_board_state
  server.registerTool(
    "chess_get_board_state",
    { /* schema */ },
    async (_, ctx) => { /* handler */ }
  );

  // 3. Register tool: chess_move
  server.registerTool(
    "chess_move",
    { /* schema */ },
    async ({ move }, ctx) => { /* handler */ }
  );

  // 4. Return the configured server
  return server;
}
```

---

### 3. Tool Definitions

#### Tool: `chess_get_board_state`

**Purpose**: Get the current state of the chess board

**Input**: None (empty object `{}`)

**Output**: 
- Text representation (ASCII board + FEN + valid moves)
- Interactive UI resource (chessboard)

**How it works**:
1. Receives the context object with `sessionId`
2. Attempts to load existing game from file system
3. If no game exists, creates a new one
4. Returns current board state in two formats:
   - Text: For the AI to read
   - UI: For humans to see

**Code Flow**:
```typescript
async (_, ctx) => {
  // Load existing game or create new one
  let game = await loadGame(ctx.sessionId);
  if (!game) {
    game = new Chess();
  }
  
  // Return both text and UI content
  return {
    content: [
      gameTextResource(game),  // ASCII + FEN + moves
      gameUI(game.fen())       // Interactive chessboard
    ],
  };
}
```

---

#### Tool: `chess_move`

**Purpose**: Make a chess move or start a new game

**Input**: 
- `move`: String in algebraic notation (e.g., "e4", "Nf3", "O-O") or "new"

**Output**: 
- Updated board state (text + UI)

**How it works**:
1. Receives move and session context
2. Loads existing game (or creates new if none exists)
3. If move is "new", starts fresh game
4. Applies the move using chess.js library
5. Saves updated game state to file
6. Returns new board state

**Code Flow**:
```typescript
async ({ move }, ctx) => {
  // Load game
  let game = await loadGame(ctx.sessionId);
  
  // Handle "new game" command
  if (move === "new") {
    game = new Chess();
  } else if (!game) {
    game = new Chess();
  }

  // Make the move
  game.move(move.trim());
  
  // Persist to disk
  await saveGame(ctx.sessionId, game);

  // Return updated state
  return {
    content: [
      gameTextResource(game),
      gameUI(game.fen())
    ],
  };
}
```

**Error Handling**: 
If an invalid move is made, chess.js throws an error which propagates back to the client.

---

### 4. MCP Middleware (`src/lib/mcp/middleware.ts`)

This is the heart of the MCP integration with Express. It's a custom implementation that bridges HTTP requests to MCP protocol.

**Purpose**:
- Handle HTTP transport for MCP protocol
- Manage sessions and their lifecycle
- Route requests to appropriate MCP server instances
- Handle connection initialization and cleanup

**Architecture**:

```typescript
export function Middleware(newServer) {
  const middleware = new McpServerMiddleware(newServer);
  return middleware.middleware;
}
```

The `Middleware` function returns an Express `RequestHandler` that can be mounted on any route.

---

#### McpServerMiddleware Class

**State Management**:
```typescript
class McpServerMiddleware {
  // Maps session IDs to their transport instances
  private readonly transports: Record<string, StreamableHTTPServerTransport> = {};
  
  // Express app for routing
  private readonly app = express();
  
  // Factory function to create MCP servers
  private readonly newServer: (...) => Promise<McpServer> | McpServer;
}
```

**Key Properties**:
- `transports`: A registry of active sessions. Each session ID maps to its transport layer
- `app`: An internal Express application that handles routing
- `newServer`: The factory function passed from `server.ts`

---

**Routes**:
```typescript
private setup = () => {
  this.app.use(express.json());      // Parse JSON bodies
  this.app.post("/", this.post);     // Handle MCP requests
  this.app.get("/", this.post);      // Some clients use GET
  this.app.delete("/", this.getDelete); // Session cleanup
};
```

---

**Session Initialization** (`newTransport` method):

This is called when a new session needs to be created.

```typescript
private newTransport = async (sessionId, req, res) => {
  let transport;
  
  if (sessionId) {
    // Client provided a session ID (reconnection)
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,
    });
    transport.sessionId = sessionId;
  } else {
    // New session - generate random UUID
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (sessionId) => {
        this.transports[sessionId] = transport;
      },
    });
  }

  // Register cleanup handler
  transport.onclose = () => {
    if (transport.sessionId) {
      delete this.transports[transport.sessionId];
    }
  };

  // Create and connect MCP server
  const server = await this.newServer(req, res, transport);
  await server.connect(transport);
  
  return transport;
};
```

**What's happening**:
1. Check if client provided a session ID (reconnecting) or needs a new one
2. Create appropriate transport with either explicit or generated session ID
3. Register cleanup handler to remove transport when session ends
4. Call the `newServer` factory to get an MCP server instance
5. Connect the server to the transport
6. Return the configured transport

---

**Request Handling** (`post` method):

This handles every tool invocation request.

```typescript
private post = async (req, res) => {
  // Extract session ID from header
  const sessionId = header(req.headers, "mcp-session-id");

  let transport;

  // Check if we have an existing session
  if (sessionId && this.transports[sessionId]) {
    transport = this.transports[sessionId];
  } 
  // Or if this is a new session initialization
  else if (
    (!sessionId && isInitializeRequest(req.body)) ||
    validUUID(sessionId)
  ) {
    transport = await this.newTransport(sessionId, req, res);
  } 
  // Invalid request
  else {
    res.status(404).json({
      jsonrpc: "2.0",
      error: {
        code: -32000,
        message: "Bad Request: No valid session ID provided",
      },
      id: null,
    });
    return;
  }

  // Forward request to transport
  await transport.handleRequest(req, res, req.body);
};
```

**Decision Flow**:
1. Extract session ID from `mcp-session-id` header
2. If valid session exists → Use existing transport
3. If new initialization request → Create new transport
4. Otherwise → Return error
5. Forward request to transport layer for processing

---

### 5. Transport Layer (StreamableHTTPServerTransport)

This is provided by the MCP SDK (`@modelcontextprotocol/sdk`). It handles:
- Converting HTTP requests to MCP protocol messages
- Converting MCP responses back to HTTP
- Managing the JSON-RPC communication
- Session ID generation and tracking

**What we don't see**: Most of this is abstracted by the SDK. We just call:
```typescript
await transport.handleRequest(req, res, req.body);
```

And it:
1. Parses the JSON-RPC request
2. Calls the appropriate tool handler
3. Formats the response
4. Sends it back via HTTP

---

### 6. Game State Store (`src/lib/store.ts`)

This module handles persistence of chess games to the file system.

**Why File System?**
- Simple for this use case
- No database setup required
- Easy to inspect and debug (just open the JSON files)
- Good enough for session-based games

**Production Considerations**: 
For a production system, you'd typically use a database like PostgreSQL or Redis.

---

#### `saveGame` Function

**Purpose**: Persist game state to disk

```typescript
export async function saveGame(
  sessionId: string | undefined,
  game: Chess,
): Promise<void> {
  // Ensure games directory exists
  if (!existsSync("./games")) {
    await mkdir("./games", { recursive: true });
  }

  // Save FEN string to JSON file
  await writeFile(
    `./games/${sessionId}.json`,
    JSON.stringify(game.fen(), null, 2),
  );
}
```

**How it works**:
1. Check if `./games` directory exists, create if not
2. Write game state as JSON to `./games/{sessionId}.json`
3. We only store the FEN (Forsyth-Edwards Notation) string, not the full game object

**Why FEN?**
FEN is a standard notation that completely describes a chess position:
- Piece positions
- Whose turn it is
- Castling rights
- En passant square
- Halfmove clock
- Fullmove number

Example FEN: `rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1`

---

#### `loadGame` Function

**Purpose**: Retrieve game state from disk

```typescript
export async function loadGame(sessionId?: string): Promise<Chess | null> {
  // Check if file exists
  if (!existsSync(`./games/${sessionId}.json`) || !sessionId) {
    return null;
  }

  // Read FEN from file
  const data = await readFile(`./games/${sessionId}.json`, "utf-8");
  const fen = JSON.parse(data) as string;
  
  // Create Chess instance from FEN
  return new Chess(fen);
}
```

**How it works**:
1. Check if game file exists for this session
2. If not, return null (indicating new game needed)
3. If yes, read FEN string from file
4. Create new Chess instance initialized with that position
5. Return the game object

**Error Handling**: Returns `null` if no game exists, letting the caller decide what to do.

---

### 7. Helper Utilities

#### `header` Function (`src/lib/mcp/header.ts`)

**Purpose**: Safely extract header values from HTTP request

```typescript
export default function header(
  headers: Record<string, string | string[] | undefined> | undefined,
  name: string,
): string | undefined {
  if (!headers) return;
  const value = headers[name];
  
  // Handle array values (some headers can be arrays)
  if (Array.isArray(value)) {
    return value[0];
  }
  return value;
}
```

**Why needed**: HTTP headers can be strings or arrays of strings. This normalizes them.

---

#### `validUUID` Function (`src/lib/mcp/uuid.ts`)

**Purpose**: Validate if a string is a proper UUID

```typescript
export function validUUID(id?: string): boolean {
  if (!id) return false;
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(id);
}
```

**Why needed**: Session IDs must be UUIDs. This prevents invalid session IDs from being accepted.

---

### 8. UI Generation

#### `gameUI` Function (`src/server.ts`)

**Purpose**: Generate interactive chessboard UI

This function creates an MCP UI resource - a special content type that clients can render as interactive HTML.

**Key Features**:
- Drag-and-drop chess pieces
- Visual representation of the board
- Bidirectional communication (UI can trigger tool calls)

**Structure**:
```typescript
function gameUI(fen: string) {
  return createUIResource({
    uri: `ui://chess_board/${fen}`,
    uiMetadata: {
      "preferred-frame-size": ["400px", "400px"],
    },
    encoding: "text",
    content: {
      type: "rawHtml",
      htmlString: `<!DOCTYPE html>
        <!-- Full HTML page with embedded JavaScript -->
      `
    },
  });
}
```

**Components**:
1. **URI**: Unique identifier for this UI resource
2. **Metadata**: Hints for rendering (size preferences)
3. **HTML Content**: Complete webpage with:
   - ChessBoard.js library
   - Chess.js library for validation
   - jQuery for DOM manipulation
   - Event handlers for piece movement

---

**Interactive Features**:

The generated HTML includes JavaScript that:

1. **Initializes the board**:
```javascript
var board = Chessboard('myBoard', {
  position: pos,           // Current FEN position
  draggable: true,         // Enable drag-and-drop
  pieceTheme: '...',      // Chess piece images
  showNotation: true,      // Show rank/file labels
  onDrop: function (source, target) { /* ... */ }
});
```

2. **Handles piece drops**:
```javascript
onDrop: function (source, target) {
  // Send move to parent window (the MCP client)
  window.parent.postMessage({
    type: 'tool',
    payload: {
      toolName: 'chess_move',
      params: {
        move: source + "-" + target
      }
    }
  }, '*')
}
```

**How it works**:
1. User drags a piece (e.g., from e2 to e4)
2. `onDrop` handler is called with source="e2" and target="e4"
3. JavaScript sends `postMessage` to parent window
4. MCP client (nanobot/Claude) receives the message
5. Client calls `chess_move` tool with `move: "e2-e4"`
6. Server processes move and returns updated board
7. UI re-renders with new position

---

#### `gameTextResource` Function

**Purpose**: Create text representation of board state

```typescript
function gameTextResource(game: Chess): TextContent {
  return {
    type: "text",
    text: `Current board state:
      FEN: ${game.fen()}
      VALID MOVES: ${game.moves()}
      
      ${game.ascii()}`,
  };
}
```

**Why both text and UI?**
- **Text**: For the AI to understand the position
- **UI**: For humans to visualize and interact

---

## Request Lifecycle

Let's trace a complete request from start to finish. We'll follow a move request.

### Scenario: User makes the move "e4"

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Client Initiates Request                                 │
└─────────────────────────────────────────────────────────────────┘

Client (nanobot) sends HTTP POST to http://localhost:3000/mcp

Headers:
  Content-Type: application/json
  mcp-session-id: 550e8400-e29b-41d4-a716-446655440000

Body:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "chess_move",
    "arguments": { "move": "e4" }
  },
  "id": 1
}

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Express Routes to MCP Middleware                         │
└─────────────────────────────────────────────────────────────────┘

Express receives request on /mcp route
↓
Middleware function is called
↓
Request enters McpServerMiddleware.post()

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Session Lookup                                           │
└─────────────────────────────────────────────────────────────────┘

Extract sessionId from header: "550e8400-e29b-41d4-a716-446655440000"
↓
Check transports registry:
  this.transports["550e8400-e29b-41d4-a716-446655440000"]
↓
If found: Use existing transport
If not found: Create new transport (see below)

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Transport Handles Request                                │
└─────────────────────────────────────────────────────────────────┘

transport.handleRequest(req, res, req.body)
↓
StreamableHTTPServerTransport parses JSON-RPC:
  - Method: "tools/call"
  - Tool name: "chess_move"
  - Arguments: { move: "e4" }
↓
Finds registered tool handler for "chess_move"

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Tool Handler Executes                                    │
└─────────────────────────────────────────────────────────────────┘

async ({ move }, ctx) => {
  // ctx.sessionId = "550e8400-e29b-41d4-a716-446655440000"
  // move = "e4"
  
  1. Load game:
     let game = await loadGame(ctx.sessionId);
     
  2. Check file system:
     ./games/550e8400-e29b-41d4-a716-446655440000.json
     
  3. If exists:
     - Read FEN string
     - Create Chess instance
     
  4. If not exists:
     - Create new Chess instance (starting position)
     
  5. Make move:
     game.move("e4")
     
  6. Save updated state:
     await saveGame(ctx.sessionId, game)
     Writes new FEN to file
     
  7. Return response:
     return {
       content: [
         gameTextResource(game),
         gameUI(game.fen())
       ]
     }
}

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 6: Response Formatting                                      │
└─────────────────────────────────────────────────────────────────┘

Tool handler returns:
{
  content: [
    {
      type: "text",
      text: "Current board state:\n  FEN: ...\n  VALID MOVES: ..."
    },
    {
      type: "resource",
      resource: {
        uri: "ui://chess_board/...",
        text: "<!DOCTYPE html>...",
        mimeType: "text/html"
      }
    }
  ]
}

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 7: JSON-RPC Response                                        │
└─────────────────────────────────────────────────────────────────┘

Transport wraps response in JSON-RPC format:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [ /* ... */ ]
  },
  "id": 1
}

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 8: HTTP Response                                            │
└─────────────────────────────────────────────────────────────────┘

Response sent back to client with:
  Status: 200 OK
  Content-Type: application/json
  Body: JSON-RPC response

▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 9: Client Renders Response                                  │
└─────────────────────────────────────────────────────────────────┘

Client receives response:
1. Parses JSON-RPC result
2. Extracts content array
3. Renders text content in chat
4. Renders UI resource in iframe
5. User sees updated chessboard
```

---

## Session Management

### Session Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Session Initialization                                        │
└─────────────────────────────────────────────────────────────────┘

Client connects without session ID
↓
Server receives initialize request
↓
newTransport() called with sessionId = undefined
↓
StreamableHTTPServerTransport generates UUID
↓
New session created: "550e8400-e29b-41d4-a716-446655440000"
↓
Transport added to registry:
  transports["550e8400-e29b-41d4-a716-446655440000"] = transport
↓
Server responds with session ID in header
↓
Client stores session ID for future requests

▼

┌─────────────────────────────────────────────────────────────────┐
│ 2. Session Active                                                │
└─────────────────────────────────────────────────────────────────┘

Client sends requests with mcp-session-id header
↓
Server looks up existing transport
↓
Reuses same MCP server instance
↓
Game state persists across requests
↓
File: ./games/{sessionId}.json updated after each move

▼

┌─────────────────────────────────────────────────────────────────┐
│ 3. Session Termination                                           │
└─────────────────────────────────────────────────────────────────┘

Option A: Client sends DELETE request
↓
transport.onclose() called
↓
Session removed from registry
↓
Game file persists on disk

Option B: Server restart
↓
All transports lost
↓
Game files remain on disk
↓
Client can reconnect with same session ID
↓
New transport created
↓
Game state loaded from file
```

### Session Isolation

Each session is completely isolated:

1. **Separate MCP Server Instances**
   - Each session gets its own `McpServer` via `newServer()`
   - Tool handlers execute in session context

2. **Separate Game State**
   - Each session has its own game file
   - One session can't affect another

3. **Separate Transport**
   - Each session has its own HTTP transport
   - Concurrent requests to different sessions don't conflict

---

## Data Flow

### Making a Move

```
┌──────────┐     ┌────────────┐     ┌──────────┐     ┌────────┐
│  Client  │────▶│ Middleware │────▶│   Tool   │────▶│  Store │
└──────────┘     └────────────┘     │ Handler  │     └────────┘
     │                  │            └──────────┘          │
     │                  │                  │               │
   move: "e4"      sessionId          loadGame()      Read FEN
                      ↓                    ↓               │
                  Lookup              game.move()          │
                  transport               ↓                │
                      │              saveGame()            │
                      │                   │                ▼
                      │                   │           Write FEN
                      │                   │                │
                      │                   ▼                │
                      │              gameTextResource()    │
                      │              gameUI()              │
                      │                   │                │
     ┌────────────────┼───────────────────┘                │
     │                │                                     │
     ▼                ▼                                     ▼
┌──────────┐     ┌────────────┐                    ┌────────────┐
│  Client  │◀────│  Response  │                    │ games/     │
│          │     │  JSON-RPC  │                    │ {id}.json  │
└──────────┘     └────────────┘                    └────────────┘
     │
     ▼
  Render UI
  Show text
```

### State Persistence

```
Memory (Runtime)                    Disk (Persistent)
─────────────────                   ─────────────────

Chess instance                      FEN string in JSON
  ├─ Board position        ────▶    {
  ├─ Move history                     "rnbqkbnr/pppp..."
  ├─ Turn                           }
  ├─ Castling rights
  └─ En passant

After each move:
  game.move("e4")           ────▶   saveGame()
  state in memory                   ↓
                                   writeFile()
                                   ↓
                                   ./games/{sessionId}.json

Before each move:
  loadGame()                ────▶   readFile()
  ↓                                 ↓
  new Chess(fen)                   FEN string
  ↓
  state in memory
```

---

## UI Integration

### How UI Communicates Back to Server

This is one of the most interesting parts of the architecture.

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. UI Rendered in Client                                         │
└─────────────────────────────────────────────────────────────────┘

Server returns UI resource:
  type: "resource"
  content: HTML with embedded JavaScript

Client (nanobot) renders HTML in iframe:
  <iframe src="blob:..." sandbox="...">
    <!-- Chessboard.js UI -->
  </iframe>

▼

┌─────────────────────────────────────────────────────────────────┐
│ 2. User Interacts with UI                                        │
└─────────────────────────────────────────────────────────────────┘

User drags piece from e2 to e4
↓
ChessBoard.js calls onDrop handler:
  onDrop: function(source, target) {
    // source = "e2", target = "e4"
  }

▼

┌─────────────────────────────────────────────────────────────────┐
│ 3. UI Posts Message to Parent                                    │
└─────────────────────────────────────────────────────────────────┘

JavaScript in iframe:
  window.parent.postMessage({
    type: 'tool',
    payload: {
      toolName: 'chess_move',
      params: {
        move: 'e2-e4'
      }
    }
  }, '*')

Message travels from iframe to parent window

▼

┌─────────────────────────────────────────────────────────────────┐
│ 4. Client Receives Message                                       │
└─────────────────────────────────────────────────────────────────┘

Nanobot listens for postMessage events
↓
Receives tool invocation request from UI
↓
Validates message format

▼

┌─────────────────────────────────────────────────────────────────┐
│ 5. Client Calls Tool                                             │
└─────────────────────────────────────────────────────────────────┘

Client makes HTTP request to server:
  POST /mcp
  mcp-session-id: {current-session}
  Body: {
    "method": "tools/call",
    "params": {
      "name": "chess_move",
      "arguments": { "move": "e2-e4" }
    }
  }

▼

┌─────────────────────────────────────────────────────────────────┐
│ 6. Server Processes Move                                         │
└─────────────────────────────────────────────────────────────────┘

(Same flow as regular tool call)
↓
Returns updated board state

▼

┌─────────────────────────────────────────────────────────────────┐
│ 7. Client Updates UI                                             │
└─────────────────────────────────────────────────────────────────┘

Client receives new UI resource with updated FEN
↓
Replaces iframe content
↓
User sees piece moved on board
```

### Security Considerations

**iframe sandbox**: The UI runs in a sandboxed iframe with limited permissions

**postMessage origin**: Using `'*'` for origin is not ideal for production. You should specify the exact origin:
```javascript
window.parent.postMessage(msg, 'https://nanobot.example.com')
```

**Input validation**: The server validates all moves through chess.js, preventing illegal moves

---

## Code Walkthrough

Let's walk through the complete codebase file by file.

### File: `src/server.ts`

**Purpose**: Main entry point and tool definitions

**Line-by-line breakdown**:

```typescript
// 1-8: Imports
import express from "express";                    // Web server
import { Middleware } from "./lib/mcp";           // MCP middleware
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";  // MCP SDK
import { loadGame, saveGame } from "./lib/store"; // Persistence
import { z } from "zod";                          // Schema validation
import { Chess } from "chess.js";                 // Chess engine
import { createUIResource } from "@mcp-ui/server"; // UI resources
import type { TextContent } from "@modelcontextprotocol/sdk/types.js";

// 10-63: gameUI function
// Creates interactive chessboard HTML
function gameUI(fen: string) {
  return createUIResource({
    uri: `ui://chess_board/${fen}`,  // Unique URI per position
    uiMetadata: {
      "preferred-frame-size": ["400px", "400px"],  // Size hint
    },
    encoding: "text",
    content: {
      type: "rawHtml",
      htmlString: `...`  // Full HTML page with ChessBoard.js
    },
  });
}

// 65-74: gameTextResource function
// Creates text representation of board
function gameTextResource(game: Chess): TextContent {
  return {
    type: "text",
    text: `Current board state:
      FEN: ${game.fen()}           // Position
      VALID MOVES: ${game.moves()} // Legal moves
      
      ${game.ascii()}`             // ASCII board
  };
}

// 76-130: newServer function
// Factory that creates MCP server instances
function newServer() {
  // Create server with metadata
  const server = new McpServer({
    name: "Chess MCP Server",
    version: "0.0.1",
  });

  // Register tool: chess_get_board_state
  server.registerTool(
    "chess_get_board_state",        // Tool name
    {
      description: "...",           // Human-readable description
      inputSchema: {},              // No parameters needed
    },
    async (_, ctx) => {             // Handler function
      let game = await loadGame(ctx.sessionId);
      if (!game) {
        game = new Chess();         // New game if none exists
      }
      return {
        content: [
          gameTextResource(game),   // Text for AI
          gameUI(game.fen())       // UI for human
        ],
      };
    },
  );

  // Register tool: chess_move
  server.registerTool(
    "chess_move",                   // Tool name
    {
      description: "...",           // Human-readable description
      inputSchema: {
        move: z.string().describe("...") // Zod schema
      },
    },
    async ({ move }, ctx) => {      // Handler with move parameter
      let game = await loadGame(ctx.sessionId);
      
      // Handle special "new" command
      if (move === "new") {
        game = new Chess();
      } else if (!game) {
        game = new Chess();
      }

      // Make the move (throws if invalid)
      game.move(move.trim());
      
      // Persist to disk
      await saveGame(ctx.sessionId, game);

      // Return updated state
      return {
        content: [
          gameTextResource(game),
          gameUI(game.fen())
        ],
      };
    },
  );

  return server;  // Return configured server
}

// 132-136: Express setup
const app = express();
app.use("/mcp", Middleware(newServer));  // Mount MCP middleware

console.log("Starting server on http://localhost:3000/mcp");
app.listen(3000);  // Start listening
```

---

### File: `src/lib/store.ts`

**Purpose**: Game state persistence

```typescript
// 1-3: Imports
import { writeFile, readFile, mkdir } from "node:fs/promises";
import { existsSync } from "node:fs";
import { Chess } from "chess.js";

// 5-18: saveGame function
export async function saveGame(
  sessionId: string | undefined,
  game: Chess,
): Promise<void> {
  // Ensure games directory exists
  if (!existsSync("./games")) {
    await mkdir("./games", { recursive: true });
  }

  // Write FEN as JSON
  await writeFile(
    `./games/${sessionId}.json`,
    JSON.stringify(game.fen(), null, 2),  // Pretty print
  );
}

// 20-28: loadGame function
export async function loadGame(sessionId?: string): Promise<Chess | null> {
  // Check if file exists
  if (!existsSync(`./games/${sessionId}.json`) || !sessionId) {
    return null;  // No game for this session
  }

  // Read and parse FEN
  const data = await readFile(`./games/${sessionId}.json`, "utf-8");
  const fen = JSON.parse(data) as string;
  
  // Create Chess instance from FEN
  return new Chess(fen);
}
```

**Key Points**:
- Uses async file operations
- Creates directory on first save
- Returns null if no game exists (not an error)
- Only stores FEN, not full game history

---

### File: `src/lib/mcp/middleware.ts`

**Purpose**: HTTP transport for MCP protocol

```typescript
// 1-11: Imports
import express, { type Request, type RequestHandler, type Response } from "express";
import { randomUUID } from "node:crypto";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";
import header from "./header";
import { validUUID } from "./uuid";

// 13-22: Middleware function
// Public API - returns Express middleware
export function Middleware(
  newServer: (...) => Promise<McpServer> | McpServer,
): RequestHandler {
  const middleware = new McpServerMiddleware(newServer);
  return middleware.middleware;  // Returns Express RequestHandler
}

// 24-134: McpServerMiddleware class
class McpServerMiddleware {
  // Transport registry
  private readonly transports: Record<string, StreamableHTTPServerTransport> = {};
  
  // Internal Express app for routing
  private readonly app = express();
  
  // Server factory function
  private readonly newServer: (...) => Promise<McpServer> | McpServer;

  // Getter returns Express app as middleware
  get middleware(): RequestHandler {
    return this.app;
  }

  // Constructor sets up routes
  constructor(newServer) {
    this.newServer = newServer;
    this.setup();
  }

  // Setup routes
  private setup = () => {
    this.app.use(express.json());        // Parse JSON
    this.app.post("/", this.post);       // Tool calls
    this.app.get("/", this.post);        // Also handle GET
    this.app.delete("/", this.getDelete); // Session cleanup
  };

  // Create new transport and server
  private newTransport = async (sessionId, req, res) => {
    let transport;
    
    if (sessionId) {
      // Reconnecting with existing ID
      transport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined,
      });
      transport.sessionId = sessionId;
      this.transports[sessionId] = transport;
      transport.onclose = () => {
        delete this.transports[transport.sessionId];
      };
    } else {
      // New session - generate UUID
      transport = new StreamableHTTPServerTransport({
        sessionIdGenerator: () => randomUUID(),
        onsessioninitialized: (sessionId) => {
          this.transports[sessionId] = transport;
        },
      });
    }

    // Cleanup handler
    transport.onclose = () => {
      if (transport.sessionId) {
        delete this.transports[transport.sessionId];
      }
    };

    // Create and connect server
    const server = await this.newServer(req, res, transport);
    await server.connect(transport);
    return transport;
  };

  // Handle POST/GET requests
  private post = async (req, res) => {
    const sessionId = header(req.headers, "mcp-session-id");

    let transport;

    // Find or create transport
    if (sessionId && this.transports[sessionId]) {
      // Existing session
      transport = this.transports[sessionId];
    } else if (
      (!sessionId && isInitializeRequest(req.body)) ||
      validUUID(sessionId)
    ) {
      // New session or valid reconnection
      transport = await this.newTransport(sessionId, req, res);
    } else {
      // Invalid request
      res.status(404).json({
        jsonrpc: "2.0",
        error: {
          code: -32000,
          message: "Bad Request: No valid session ID provided",
        },
        id: null,
      });
      return;
    }

    // Forward to transport
    await transport.handleRequest(req, res, req.body);
  };

  // Handle DELETE requests
  private getDelete = async (req, res) => {
    const sessionId = header(req.headers, "mcp-session-id");
    const transport = sessionId && this.transports[sessionId];

    if (!transport) {
      res.status(400).send(`Invalid or missing session ID`);
      return;
    }
    
    await transport.handleRequest(req, res);
  };
}
```

**Key Points**:
- Encapsulates all MCP transport logic
- Manages session lifecycle
- Routes requests to correct transport
- Handles errors gracefully

---

## How Everything Works Together

Let's trace the complete flow with a real example:

### Scenario: Starting a Chess Game

```
User: "Let's play chess, make your first move"

┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: Session Initialization                                  │
└─────────────────────────────────────────────────────────────────┘

1. Client (Claude/nanobot) receives user message
2. Decides to use chess_move tool
3. No session exists yet
4. Sends initialize request to server
5. Server generates session ID: abc-123
6. Returns session ID in response header
7. Client stores: mcp-session-id = abc-123

┌─────────────────────────────────────────────────────────────────┐
│ Phase 2: First Tool Call                                         │
└─────────────────────────────────────────────────────────────────┘

Client sends:
  POST /mcp
  mcp-session-id: abc-123
  {
    "method": "tools/call",
    "params": {
      "name": "chess_move",
      "arguments": { "move": "e4" }
    }
  }

Server:
  1. Middleware extracts session ID
  2. Finds transport in registry
  3. Transport routes to chess_move handler
  4. Handler calls loadGame("abc-123")
  5. No file exists → returns null
  6. Handler creates new Chess()
  7. Handler calls game.move("e4")
  8. Handler calls saveGame("abc-123", game)
  9. Writes: ./games/abc-123.json with FEN
  10. Returns text + UI content

Client receives:
  {
    "result": {
      "content": [
        {
          "type": "text",
          "text": "Current board state:\n  FEN: rnbqkbnr/pppppppp/..."
        },
        {
          "type": "resource",
          "resource": {
            "uri": "ui://chess_board/...",
            "text": "<!DOCTYPE html>..."
          }
        }
      ]
    }
  }

Client:
  1. Shows text in chat
  2. Renders UI in iframe
  3. User sees chessboard with e4 played

┌─────────────────────────────────────────────────────────────────┐
│ Phase 3: User Response                                           │
└─────────────────────────────────────────────────────────────────┘

User drags piece from e7 to e5 in UI

UI JavaScript:
  1. onDrop("e7", "e5") called
  2. Posts message to parent:
     {
       type: 'tool',
       payload: {
         toolName: 'chess_move',
         params: { move: 'e7-e5' }
       }
     }

Client:
  1. Receives postMessage
  2. Calls tool with move: "e7-e5"

Server:
  1. Middleware finds session abc-123
  2. Handler calls loadGame("abc-123")
  3. File exists → reads FEN → creates Chess
  4. Game is at position after e4
  5. Handler calls game.move("e7-e5")
  6. Handler calls saveGame("abc-123", game)
  7. Writes updated FEN to file
  8. Returns new text + UI

Client:
  1. Shows updated text
  2. Renders new UI with both moves

┌─────────────────────────────────────────────────────────────────┐
│ Phase 4: Game Continues                                          │
└─────────────────────────────────────────────────────────────────┘

Each subsequent move:
  1. Loads game from file (current position)
  2. Applies move
  3. Saves updated position
  4. Returns new state

File system:
  ./games/abc-123.json keeps getting updated with latest FEN

Session:
  Persists until:
    - Server restart (can be restored from file)
    - Client explicitly deletes session
    - Transport timeout (implementation specific)
```

---

## Summary

### Key Architectural Decisions

1. **File-based Persistence**
   - Simple, no database required
   - Good for development and small scale
   - Easy to inspect and debug

2. **Session-based State**
   - Each user gets isolated game
   - State persists across requests
   - Can resume after disconnect

3. **Factory Pattern for Servers**
   - Each session gets own MCP server
   - Clean separation of concerns
   - Easy to test and maintain

4. **Dual Content Types**
   - Text for AI consumption
   - UI for human interaction
   - Both updated together

5. **HTTP Transport**
   - Easy to deploy and access
   - Standard web infrastructure
   - Works with proxies and load balancers

### Data Flow Summary

```
User Input
    ↓
MCP Client (Claude/nanobot)
    ↓
HTTP Request
    ↓
Express Server
    ↓
MCP Middleware
    ↓
Transport Layer
    ↓
MCP Server Instance
    ↓
Tool Handler
    ↓
Chess.js (move validation)
    ↓
File System (state persistence)
    ↓
Response (text + UI)
    ↓
HTTP Response
    ↓
MCP Client
    ↓
User sees updated board
```

### Extension Points

Want to add features? Here's where to modify:

**Add new tool**: `src/server.ts` → `newServer()` → `server.registerTool()`

**Change storage**: `src/lib/store.ts` → Replace file operations with database

**Modify UI**: `src/server.ts` → `gameUI()` → Update HTML

**Add middleware**: `src/server.ts` → Add Express middleware before MCP

**Session customization**: `src/lib/mcp/middleware.ts` → Modify `McpServerMiddleware`

---

## Conclusion

The Chess MCP Server demonstrates a clean, modular architecture for building MCP servers. Key takeaways:

1. **MCP provides standard protocol** - No need to invent your own API
2. **Express integration is straightforward** - Custom middleware bridges HTTP to MCP
3. **Session management is crucial** - Enables stateful interactions
4. **UI resources are powerful** - Interactive experiences within AI chat
5. **File-based storage is simple** - Good enough for many use cases

The architecture is designed to be:
- **Understandable**: Clear separation of concerns
- **Extensible**: Easy to add new tools
- **Testable**: Each component can be tested independently
- **Deployable**: Standard web server, works anywhere

Whether you're building your own MCP server or just trying to understand this one, remember: at its core, it's just HTTP requests being routed to function calls, with some smart session management and state persistence in between.
