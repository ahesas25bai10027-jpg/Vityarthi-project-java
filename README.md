# Vityarthi-project-java
Java Multi-Client Chat Application
Overview

A real-time, multi-user chat application built in core Java using TCP sockets and multithreading — no external frameworks or libraries required. The application follows a classic client-server architecture: a central server accepts multiple simultaneous client connections, each handled on its own thread, and routes messages between clients based on chat rooms or direct (private) targeting.

This project demonstrates practical application of:

Socket programming (java.net)
Multithreading and concurrency (Thread, Runnable, concurrent collections)
Client-server architecture and protocol design
Basic text-based command parsing
Features
Multi-client support — any number of clients can connect to the server simultaneously, each on an independent thread.
Chat rooms — users start in a default general room and can create or join other rooms on the fly with /join <room>.
Private messaging — direct, one-to-one messages between users regardless of which room they're in, via /msg <username> <message>.
Room and user discovery — /rooms lists all active rooms; /users lists everyone in your current room.
Live join/leave notifications — other users in a room are notified when someone joins, switches rooms, or disconnects.
Graceful disconnect handling — /quit, or simply closing the client, cleanly removes the user from the server's state.
Timestamped messages — every chat and private message is stamped with the local time it was sent.
Technologies / Tools Used
Category	Tool
Language	Java (JDK 17+ recommended)
Networking	java.net.Socket, java.net.ServerSocket
Concurrency	java.lang.Thread, java.util.concurrent.ConcurrentHashMap, java.util.concurrent.CopyOnWriteArraySet
I/O	java.io.BufferedReader, java.io.PrintWriter
Version Control	Git / GitHub
Diagrams	Mermaid (rendered natively by GitHub — see diagrams.md)

No external dependencies or build tools (Maven/Gradle) are required — the project compiles with the standard JDK alone.

Project Structure
ChatServer.java       # Accepts connections; owns shared room/user state
ClientHandler.java    # Per-client thread; parses commands, routes messages
ChatClient.java       # Client app; separate send/receive threads
diagrams.md           # Architecture, use case, class, sequence diagrams
statement.md          # Problem statement, scope, target users
README.md             # This file

Installation & How to Run
Prerequisites
JDK 17 or later installed (java -version and javac -version to confirm)
1. Compile

From the project folder:

javac *.java

2. Start the server
3. 
java ChatServer

By default this listens on port 5000. To use a different port:

java ChatServer 6000

3. Start one or more clients

In a separate terminal window for each user:

java ChatClient

This connects to localhost:5000 by default. To connect to a server on another machine or port:

java ChatClient <server-ip> <port>

You'll be prompted to enter a username on connecting.

Commands
Command	Description
(plain text)	Broadcasts the message to your current room
/join <room>	Leaves your current room and joins (or creates) <room>
/rooms	Lists all currently active rooms
/users	Lists all users in your current room
/msg <username> <message>	Sends a private message to a specific user
/quit	Disconnects from the server
Testing

Manual/functional testing steps:

Start the server, then start two or more client instances.
Confirm each client can pick a unique username (try a duplicate to confirm rejection).
Send a plain message from one client and confirm all clients in the same room receive it.
Use /join newroom on one client and confirm it no longer receives general messages, and that other clients see the "left/joined" notices.
Use /msg <username> <text> and confirm only the target receives it, along with a confirmation echo to the sender.
Use /rooms and /users to confirm they reflect current server state accurately.
Disconnect a client with /quit and confirm the server removes them and notifies their room.
