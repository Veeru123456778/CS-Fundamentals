# How single threaded server maintains the multiple concurrent connections through system call interface (using epoll or poll system calls)

## Phase 1: Initialization & Setup

Before any clients arrive, the server must prepare its environment.

• Step 1.1: Server Socket Creation

The server starts up and creates a listening socket to welcome incoming connections. In Linux, this socket is assigned a unique File Descriptor integer (e.g., FD 3).

• Step 1.2: Set to Non-Blocking Mode

The server explicitly configures FD 3 (and all future client sockets) to non-blocking mode. This ensures that if the server ever attempts to read or accept data when nothing is there, the system call will return immediately with an error (EWOULDBLOCK) instead of freezing the thread.

• Step 1.3: Create the epoll Instance

The server requests the Kernel to create an isolated tracking interest list. The Kernel sets this up and returns an epoll file descriptor (e.g., FD 4).

• Step 1.4: Register the Listening Socket

The server uses epoll_ctl() to add its listening socket (FD 3) to the epoll list. It tells the Kernel: "Alert me whenever a new client is trying to connect."

## Phase 2: The Core Event Loop

The single-threaded server enters an infinite loop, acting as a reactive coordinator.

• Step 2.1: The epoll_wait() Call

The single thread calls epoll_wait().
	• If all connected clients are silent, the thread goes to sleep efficiently, consuming 0% CPU.
	• The moment network data arrives, the Kernel instantly wakes the thread up.


• Step 2.2: Kernel Checks the Readiness List

Instead of scanning thousands of idle connections, the Kernel checks a dedicated internal "ready list." It populates an array for the server containing only the file descriptors that actually have active events.


• Step 2.3: Return to User Space

epoll_wait() returns a number indicating how many descriptors are active, along with the specific list of active FDs (e.g., "You have 2 events: FD 3 has an incoming connection, and FD 6 sent data").

## Phase 3: Handling New Connections (Asynchronously)

The loop iterates through the active list returned by the kernel. If it encounters the listening socket (FD 3), it handles the new client.

• Step 3.1: Accepting the Client

The server calls accept(3). Because the Kernel already verified a connection is waiting, this call finishes instantly.

• Step 3.2: Mapping the New File Descriptor

The operating system creates a brand-new socket for this specific client and assigns it a new integer (e.g., FD 6).

• Step 3.3: Registering the Client with epoll

The server sets FD 6 to non-blocking and uses epoll_ctl() to add it to the epoll monitoring list. The server tells the Kernel: "Keep an eye on this new user (FD 6) and let me know when they send a query."

• Step 3.4: Instant Return

The thread does not wait for the client to speak. It immediately jumps back to the loop to check on other users.


## Phase 4: Data Processing & Sequential Execution

If the active list contains an existing client socket (like FD 6), the server processes their query.

• Step 4.1: The Server Pulls the Data

The server issues a standard read(6, buffer) system call. Because the kernel already guaranteed data is waiting in FD 6's kernel-space buffer, the OS instantly copies those bytes directly into the server's application memory (user space).

• Step 4.2: Sequential Logic Execution

The single thread now executes the business logic. It parses the request, queries a database, or computes a response. Because there is only one thread, this processing happens completely sequentially (one task after another).

• Step 4.3: Writing the Response

The server calls write(6, response_buffer) to send the result back to the user.

• Step 4.4: Loop Reset

With the request fully completed, the single thread completely clears its local memory for that request and loops right back to Step 2.1 to see who else needs attention.
