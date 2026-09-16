# System Diagrams — Java Chat Application

These diagrams are written in [Mermaid](https://mermaid.js.org/), which GitHub renders
automatically when viewing `.md` files in a repository — no extra tools needed.

---

## 1. System Architecture Diagram

Shows the overall structure: one server process handling many concurrent client
connections, each on its own thread, sharing state through the room/user registries.

```mermaid
flowchart TB
    subgraph ClientSide["Client Side (N instances)"]
        C1["ChatClient #1<br/>(main thread: send)<br/>(listener thread: receive)"]
        C2["ChatClient #2"]
        C3["ChatClient #N"]
    end

    subgraph ServerSide["Server Side (single process)"]
        SS["ServerSocket<br/>(accepts connections)"]
        CH1["ClientHandler Thread #1"]
        CH2["ClientHandler Thread #2"]
        CH3["ClientHandler Thread #N"]
        REG["ChatServer<br/>Shared State:<br/>- clients: Map&lt;username, ClientHandler&gt;<br/>- rooms: Map&lt;roomName, Set&lt;ClientHandler&gt;&gt;"]
    end

    C1 <-->|"TCP Socket"| SS
    C2 <-->|"TCP Socket"| SS
    C3 <-->|"TCP Socket"| SS

    SS -->|"spawns"| CH1
    SS -->|"spawns"| CH2
    SS -->|"spawns"| CH3

    CH1 <-->|"read/write"| REG
    CH2 <-->|"read/write"| REG
    CH3 <-->|"read/write"| REG

    CH1 <-.->|"socket I/O"| C1
    CH2 <-.->|"socket I/O"| C2
    CH3 <-.->|"socket I/O"| C3
```

**Key points:**
- The server follows a **thread-per-client** model: `ChatServer` only accepts connections; each `ClientHandler` runs independently.
- `ChatServer`'s maps are the single source of shared truth, so they use thread-safe collections (`ConcurrentHashMap`, `CopyOnWriteArraySet`).
- Each client runs two threads: one to send (keyboard input) and one to receive (server output), so sending and receiving never block each other.

---

## 2. Use Case Diagram

```mermaid
flowchart LR
    User(("User"))

    UC1(["Register with a<br/>unique username"])
    UC2(["Send message<br/>to current room"])
    UC3(["Join / create<br/>a chat room"])
    UC4(["List active rooms"])
    UC5(["List users in<br/>current room"])
    UC6(["Send private<br/>message to a user"])
    UC7(["Disconnect from<br/>the server"])

    User --> UC1
    User --> UC2
    User --> UC3
    User --> UC4
    User --> UC5
    User --> UC6
    User --> UC7

    UC2 -.->|"requires"| UC1
    UC3 -.->|"requires"| UC1
    UC6 -.->|"requires"| UC1
```

**Actor:** A single actor type, **User**, since every connected client has identical
capabilities (no admin/moderator role in the current scope).

---

## 3. Class Diagram

```mermaid
classDiagram
    class ChatServer {
        -Map~String, ClientHandler~ clients
        -Map~String, Set~ClientHandler~~ rooms
        +String DEFAULT_ROOM
        +main(args) void
        +start(port) void
        +registerClient(username, handler) boolean
        +removeClient(username) void
        +getClient(username) ClientHandler
        +getAllUsernames() Set~String~
        +joinRoom(roomName, handler) void
        +leaveRoom(roomName, handler) void
        +getAllRoomNames() Set~String~
        +getRoomMembers(roomName) Set~ClientHandler~
        +broadcastToRoom(roomName, message, exclude) void
    }

    class ClientHandler {
        -Socket socket
        -ChatServer server
        -BufferedReader in
        -PrintWriter out
        -String username
        -String currentRoom
        +run() void
        -promptForUsername() boolean
        -handleLine(line) void
        -switchRoom(newRoom) void
        -handlePrivateMessage(rest) void
        +sendMessage(message) void
        -disconnect() void
    }

    class ChatClient {
        +main(args) void
    }

    class Runnable {
        <<interface>>
        +run() void
    }

    ChatServer "1" o-- "many" ClientHandler : manages
    ClientHandler ..|> Runnable : implements
    ClientHandler "1" --> "1" ChatServer : calls back into
    ChatClient ..> ClientHandler : connects to (via socket)
```

**Notes:**
- `ClientHandler` implements `Runnable` so each client can run on its own `Thread`.
- `ChatServer` owns the shared registries; `ClientHandler` never stores room/user state itself — it always asks `ChatServer`, which keeps a single source of truth.

---

## 4. Sequence Diagram — Connect, Join Room, Send Message, Private Message

```mermaid
sequenceDiagram
    actor U as User
    participant CC as ChatClient
    participant SS as ServerSocket
    participant CH as ClientHandler (Thread)
    participant CS as ChatServer (shared state)
    participant CH2 as ClientHandler #2 (other user)

    U->>CC: run java ChatClient
    CC->>SS: open Socket connection
    SS->>CH: accept() returns socket, spawn thread
    CH->>CC: "Enter a username:"
    U->>CC: types username
    CC->>CH: sends username
    CH->>CS: registerClient(username, this)
    CS-->>CH: true (unique) / false (taken)
    CH->>CS: joinRoom("general", this)
    CH->>CC: "Welcome! You're in room 'general'"
    CH->>CS: broadcastToRoom("general", "user joined", exclude=self)
    CS->>CH2: sendMessage("user joined")

    U->>CC: types a chat message
    CC->>CH: sends message text
    CH->>CS: broadcastToRoom("general", stamped message, exclude=null)
    CS->>CH: sendMessage(...)
    CS->>CH2: sendMessage(...)
    CH-->>CC: echoes message back
    CH2-->>CC: (other client) receives message

    U->>CC: types "/msg otherUser hello"
    CC->>CH: sends "/msg otherUser hello"
    CH->>CS: getClient("otherUser")
    CS-->>CH: returns ClientHandler #2
    CH->>CH2: sendMessage("[PM from user]: hello")
    CH->>CC: sendMessage("[PM to otherUser]: hello")
```

---

## 5. Room-Switch / Disconnect Flow (Process Flow Diagram)

```mermaid
flowchart TD
    A["Client sends a line of input"] --> B{"Starts with '/'?"}
    B -->|No| C["Broadcast as chat message<br/>to current room"]
    B -->|Yes| D{"Which command?"}
    D -->|"/join room"| E["leaveRoom(currentRoom)<br/>broadcast 'left' notice"]
    E --> F["joinRoom(newRoom)<br/>broadcast 'joined' notice"]
    D -->|"/rooms"| G["Return list of all room names"]
    D -->|"/users"| H["Return members of current room"]
    D -->|"/msg user text"| I{"Target user online?"}
    I -->|Yes| J["Deliver private message<br/>+ echo confirmation to sender"]
    I -->|No| K["Reply: user not found/offline"]
    D -->|"/quit"| L["leaveRoom + broadcast 'disconnected'<br/>removeClient + close socket"]
    D -->|Unknown| M["Reply: unknown command"]
```
