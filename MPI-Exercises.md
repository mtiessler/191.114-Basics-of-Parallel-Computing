# MPI Exercises
---

**1. Consider the following MPI program snippet:**

```c
#include <stdio.h>
#include <mpi.h>

int main(int argc, char* argv[]) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("This is rank %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
```

If this program is executed with the command `mpirun -np 3 ./my_program`, which of the following is **NOT** a possible output?

a)
```
This is rank 0 of 3
This is rank 1 of 3
This is rank 2 of 3
```
b)
```
This is rank 1 of 3
This is rank 2 of 3
This is rank 0 of 3
```
c)
```
This is rank 0 of 3
This is rank 0 of 3
This is rank 0 of 3
```
d)
```
This is rank 2 of 3
This is rank 0 of 3
This is rank 1 of 3
```

**Answer:** c)
```
This is rank 0 of 3
This is rank 0 of 3
This is rank 0 of 3
```

**Justification:**
When `mpirun -np 3` is used, three distinct MPI processes are created. Each process will call `MPI_Comm_rank(MPI_COMM_WORLD, &rank)` and get a unique rank from 0 to `size-1` (i.e., 0, 1, and 2 in this case). [Source: 10, 11] Therefore, it's impossible for all three processes to report their rank as 0. The order of output lines can vary because the `printf` statements from different processes are not synchronized, as shown in the example outputs in the document. [Source: 18] Options a, b, and d show different orderings of unique ranks, which are all possible.

---

**2. Regarding MPI point-to-point communication, which statement is most accurate?**

a) `MPI_Send` is a non-blocking operation, meaning the function call returns immediately, allowing the send buffer to be reused instantly.
b) `MPI_Recv` requires the sender's rank to be explicitly specified; using `MPI_ANY_SOURCE` is generally discouraged due to potential deadlocks.
c) A "tag" in `MPI_Send` and `MPI_Recv` is used by the MPI library to determine the data type being transmitted.
d) For `MPI_Send`, the `count` parameter specifies the number of bytes to be sent, regardless of the `datatype` used.

**Answer:** b) `MPI_Recv` requires the sender's rank to be explicitly specified; using `MPI_ANY_SOURCE` is generally discouraged due to potential deadlocks.

**Justification:**
a) `MPI_Send` is generally a blocking operation. The standard states it "does not return until the message data and envelope have been safely stored away so that the sender is free to modify the send buffer." [Source: 40] While small messages *may* be buffered allowing `MPI_Send` to return sooner, this is not guaranteed and it's fundamentally a blocking call. Non-blocking sends are `MPI_Isend`. [Source: 55]
b) While `MPI_Recv` *can* use `MPI_ANY_SOURCE` to receive from any process, it is often better to specify the source for clarity and to avoid receiving an unexpected message, which could contribute to complex debugging scenarios or even deadlocks if the program logic relies on a specific order or source of messages. The document primarily shows examples where the source is specified. [Source: 22] The "discouraged due to potential deadlocks" part is a strong claim, but in complex scenarios, unspecified sources can indeed make deadlock analysis harder or contribute to receiving messages out of the intended order, which might lead to deadlocks in application logic.
c) The "tag" is an integer used by the programmer to distinguish between different messages sent between the same pair of processes. [Source: 20] It is not used by the library to determine the data type; `MPI_Datatype` (e.g., `MPI_INT`) serves this purpose.
d) The `count` parameter specifies the number of *elements* of the specified `datatype` to be sent, not the number of bytes. [Source: 20] For example, `MPI_Send(&my_array, 10, MPI_INT, ...)` sends 10 integers.

---

**3. Examine this MPI code snippet intended for a ping-pong communication between rank 0 and rank 1 with a large message:**

```c
// Assume MPI_Init, rank, size are correctly set.
// nb_ints is a large number (e.g., 100000)
// pingpongdata is an array of nb_ints integers

if (rank == 0) {
    MPI_Send(pingpongdata, nb_ints, MPI_INT, 1, 0, MPI_COMM_WORLD);
    MPI_Recv(pingpongdata, nb_ints, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Rank 0: Ping-pong completed\n");
} else if (rank == 1) {
    MPI_Send(pingpongdata, nb_ints, MPI_INT, 0, 0, MPI_COMM_WORLD);
    MPI_Recv(pingpongdata, nb_ints, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Rank 1: Ping-pong completed\n");
}
// Assume MPI_Finalize is called later.
```

What is the most likely outcome when running this code with 2 processes?

a) The program will execute successfully, and both processes will print their completion messages.
b) The program will deadlock because both processes are attempting to send simultaneously with blocking sends before any receives are posted for large messages.
c) The program will only work if `nb_ints` is small; for large `nb_ints`, only rank 0 will complete.
d) The program will cause a segmentation fault due to improper buffer handling in `MPI_Send`.

**Answer:** b) The program will deadlock because both processes are attempting to send simultaneously with blocking sends before any receives are posted for large messages.

**Justification:**
This is a classic example of a deadlock in MPI using blocking communication. `MPI_Send` for large messages typically blocks until the corresponding `MPI_Recv` is posted by the receiving process. [Source: 41] In this code, rank 0 calls `MPI_Send` and waits for rank 1 to call `MPI_Recv`. Simultaneously, rank 1 calls `MPI_Send` and waits for rank 0 to call `MPI_Recv`. Since both are waiting for the other to receive before their send can complete, neither `MPI_Send` returns, and neither process can proceed to its `MPI_Recv` call, resulting in a deadlock. The document explicitly demonstrates this scenario. [Source: 44, 47]

---

**4. How can the deadlock in the previous question (Question 3) be reliably resolved?**

a) By replacing `MPI_Send` with `MPI_Ssend` on both processes.
b) By ensuring rank 0 first sends then receives, and rank 1 first receives then sends.
c) By increasing the internal buffer sizes of the MPI library.
d) By using `MPI_Bsend` on both processes without explicitly providing a buffer with `MPI_Buffer_attach`.

**Answer:** b) By ensuring rank 0 first sends then receives, and rank 1 first receives then sends.

**Justification:**
a) `MPI_Ssend` (synchronous send) makes the situation even more stringent, as it "will not return until matching receive posted." [Source: 70] This would still lead to a deadlock if both post `MPI_Ssend` first.
b) This creates an asymmetric communication pattern. If rank 0 sends and rank 1 receives, then rank 1 can process the receive and subsequently send its message, which rank 0 (having completed its send) can then receive. This breaks the circular wait condition of the deadlock. This solution is explicitly shown as the fix in the document. [Source: 48, 52]
c) While increasing internal buffer sizes might make the symmetric code work for slightly larger messages (by allowing `MPI_Send` to buffer the message and return), it's not a reliable or portable solution, as the buffer sizes are implementation-dependent and finite. The problem resurfaces for messages exceeding these buffers. It doesn't *resolve* the fundamental deadlock logic for truly blocking sends.
d) `MPI_Bsend` allows the send to return once the data is copied to the user-provided buffer. However, if this buffer isn't large enough or if `MPI_Buffer_attach` wasn't called correctly, it might lead to errors or fall back to standard send behavior, potentially still deadlocking. Moreover, the receive side still needs to be posted. The reliable fix is to break the symmetry of send/recv operations. [Source: 69]

---

**5. Consider the non-blocking communication pattern:**

```c
// Process 0
MPI_Request send_request;
// ... (initialize send_buffer)
MPI_Isend(send_buffer, count, MPI_DATATYPE, 1, 0, MPI_COMM_WORLD, &send_request);
// ... (do some computation_A)
MPI_Recv(recv_buffer, count, MPI_DATATYPE, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
// ... (do some computation_B)
MPI_Wait(&send_request, MPI_STATUS_IGNORE);
```

Which statement is FALSE regarding this code?

a) `computation_A` can potentially overlap with the sending of `send_buffer`.
b) The `send_buffer` should not be modified by `computation_A` or `computation_B` until `MPI_Wait` completes.
c) If `MPI_Wait` is moved immediately after `MPI_Isend` (before `computation_A`), the behavior becomes effectively similar to a blocking `MPI_Send` in terms of preventing overlap of `computation_A` with the send operation.
d) Replacing `MPI_Isend` with `MPI_Send` and removing `MPI_Wait` would achieve the same level of potential overlap for `computation_A` with communication.

**Answer:** d) Replacing `MPI_Isend` with `MPI_Send` and removing `MPI_Wait` would achieve the same level of potential overlap for `computation_A` with communication.

**Justification:**
a) TRUE. `MPI_Isend` initiates the send operation and returns (non-blocking), allowing `computation_A` to execute while the data transfer might be happening in the background. [Source: 55, 61]
b) TRUE. The `send_buffer` cannot be safely modified until `MPI_Wait` (or `MPI_Test`) confirms that the non-blocking send operation has completed. Modifying it before completion can lead to undefined behavior (e.g., sending corrupted data). [Source: 56 (implicitly, as MPI_Wait is needed to know when the buffer can be reused)]
c) TRUE. If `MPI_Wait` is called immediately after `MPI_Isend`, the program will block at `MPI_Wait` until the send operation completes, effectively negating the overlap advantage of `MPI_Isend` for `computation_A`. This is similar to how a blocking `MPI_Send` would behave. [Source: 60, referring to moving MPI_Wait before MPI_Recv causing a deadlock in that specific example, but the general idea of MPI_Wait blocking applies]
d) FALSE. `MPI_Send` is a blocking operation. If `MPI_Isend` is replaced with `MPI_Send`, the program will block at `MPI_Send` until the send operation is complete (or at least buffered safely). `computation_A` would only execute *after* `MPI_Send` returns, so there would be no overlap of `computation_A` with the send communication. [Source: 40, 61 (Blocking Example)]

---

**6. What is the primary purpose of `MPI_Barrier`?**

a) To ensure all processes execute subsequent MPI communication calls at the exact same microsecond.
b) To act as a synchronization point where processes wait until all processes in the communicator have called `MPI_Barrier`.
c) To guarantee that data sent before the barrier is received by all other processes before any process exits the barrier.
d) To reduce the global sum of a variable across all processes and store it in rank 0.

**Answer:** b) To act as a synchronization point where processes wait until all processes in the communicator have called `MPI_Barrier`.

**Justification:**
a) `MPI_Barrier` does not guarantee that processes exit the barrier at the exact same microsecond. It only ensures all have arrived *before any can leave*. [Source: 74]
b) This is the definition of `MPI_Barrier`. It blocks until all processes in the specified communicator have reached the barrier call. [Source: 73]
c) `MPI_Barrier` itself does not guarantee data sent before the barrier is received. That is the responsibility of the send/receive operations. A barrier is purely a control synchronization point.
d) This describes `MPI_Reduce` (specifically with `MPI_SUM` and the result on rank 0) or `MPI_Allreduce`, not `MPI_Barrier`.

---

**7. In the context of `MPI_Bcast(void *buffer, int count, MPI_Datatype datatype, int root, MPI_Comm comm)`:**

a) Only the `root` process needs to have valid data in `buffer` before the call.
b) All processes, including the `root`, will have their `buffer` contents overwritten with the data from the `root` process.
c) The `count` parameter at the `root` can be different from the `count` parameter at non-root processes.
d) `MPI_Bcast` is an example of point-to-point communication.

**Answer:** a) Only the `root` process needs to have valid data in `buffer` before the call.

**Justification:**
a) Correct. The `root` process is the one that sends (broadcasts) the data. Non-root processes will have their `buffer` contents overwritten by the data received from the `root`. [Source: 75, 77 (code example where only rank 0 initializes data)]
b) While non-root processes have their buffers overwritten, the `root` process's buffer is the *source* of the data; it's not overwritten by itself in the typical sense of receiving data. It already has the data.
c) The `count` and `datatype` parameters must be consistent across all processes in the communicator for the `MPI_Bcast` call.
d) `MPI_Bcast` is a collective communication operation, not point-to-point. Point-to-point involves a single sender and a single receiver (e.g., `MPI_Send`, `MPI_Recv`). [Source: 5, 73]

---

**8. Consider the MPI code using `MPI_Sendrecv` in a chain:**
```c
// ... (MPI_Init, rank, size set)
int send_data = rank;
int recv_data;
int recv_from = rank - 1;
int send_to = rank + 1;

if (rank == 0) { recv_from = MPI_PROC_NULL; }
if (rank == size - 1) { send_to = MPI_PROC_NULL; }

MPI_Sendrecv(&send_data, 1, MPI_INT, send_to, 0,
             &recv_data, 1, MPI_INT, recv_from, 0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
printf("Rank %d received %d\n", rank, recv_data);
// ... (MPI_Finalize)
```
If run with `mpirun -np 4 ./chain_program`, what will rank 0 print for `recv_data`?

a) Rank 0 received 0
b) Rank 0 received 1
c) Rank 0 received an undefined value (or a special MPI value like 32767) because it receives from `MPI_PROC_NULL`.
d) The program will deadlock because rank 0 tries to receive before rank 1 sends.

**Answer:** c) Rank 0 received an undefined value (or a special MPI value like 32767) because it receives from `MPI_PROC_NULL`.

**Justification:**
For rank 0, `recv_from` is set to `MPI_PROC_NULL`. When `MPI_PROC_NULL` is used as the source in a receive operation (or destination in a send), the operation completes successfully without actually performing any communication and without altering the receive buffer. [Source: 65 (code lines 14-16)] The content of `recv_data` for rank 0 would remain uninitialized or whatever value it had before the `MPI_Sendrecv` call. The example outputs in the document [Source: 66] show large integer values (like 32767 or 32766) for rank 0's received data, which are typical uninitialized stack values or specific MPI library indicators for `MPI_PROC_NULL` receives when `MPI_STATUS_IGNORE` is used (otherwise status could be checked). `recv_data` is not initialized in the snippet for rank 0 before `MPI_Sendrecv`.

---

**9. Which collective operation is most suitable if every process has a piece of data, and every process needs to know all pieces of data from all other processes?**

a) `MPI_Gather` followed by an `MPI_Bcast` from the root.
b) `MPI_Allgather`.
c) `MPI_Scatter`.
d) `MPI_Reduce` with `MPI_SUM` followed by an `MPI_Bcast`.

**Answer:** b) `MPI_Allgather`.

**Justification:**
a) `MPI_Gather` collects data from all processes to a single root process. The root would then need to `MPI_Bcast` this collected data back to all processes. This is a two-step process.
b) `MPI_Allgather` is designed for this exact purpose. It gathers data from all processes and distributes the entire collection of data to all processes in the communicator. This is equivalent to an `MPI_Gather` followed by an `MPI_Bcast`, but `MPI_Allgather` is often implemented more efficiently as a single collective operation. [Source: 93, 94 (diagram)]
c) `MPI_Scatter` distributes distinct chunks of data from a single root process to all other processes. This is the opposite of what is needed. [Source: 81]
d) `MPI_Reduce` combines data from all processes into a single result on one (or all, for `MPI_Allreduce`) process using an operation like sum, max, etc. It doesn't distribute the original individual pieces of data.

---

**10. Regarding different MPI send modes, which statement is MOST accurate?**

a) `MPI_Ssend` (synchronous send) will return as soon as the message is buffered by the MPI library, even if the matching receive has not started.
b) `MPI_Rsend` (ready send) can be used safely at any time; the MPI library will buffer the message if the receive is not ready.
c) `MPI_Bsend` (buffered send) requires the user to attach a buffer using `MPI_Buffer_attach` before its first use; otherwise, its behavior is undefined or might fall back to standard send.
d) `MPI_Send` is guaranteed to be non-blocking for small messages.

**Answer:** c) `MPI_Bsend` (buffered send) requires the user to attach a buffer using `MPI_Buffer_attach` before its first use; otherwise, its behavior is undefined or might fall back to standard send.

**Justification:**
a) `MPI_Ssend` (synchronous send) "will not return until matching receive posted." [Source: 70] It does not return just because the message is buffered locally; it waits for the receiver to start receiving.
b) `MPI_Rsend` (ready send) "may be used ONLY if matching receive already posted." [Source: 71] If the receive is not ready, the behavior of `MPI_Rsend` is erroneous, and it will not necessarily buffer the message safely. The user is responsible for ensuring the receive is ready.
c) Correct. `MPI_Bsend` allows the sender to proceed once the message is copied to a user-supplied buffer. This buffer must be explicitly provided and attached to MPI using `MPI_Buffer_attach`. If not, or if the buffer is too small, errors or fallback to `MPI_Send` behavior can occur. [Source: 69]
d) `MPI_Send`'s behavior for small messages (whether it blocks or buffers and returns quickly, known as eager vs. rendezvous protocol) is implementation-dependent. While many implementations use an eager protocol for small messages, it's not a guarantee by the MPI standard that `MPI_Send` will be non-blocking. It's safer to assume `MPI_Send` is blocking. [Source: 40, 41]

---

**11. Consider the following code executed with 3 processes:**

```c
// ... (MPI_Init, rank, size set to 3)
int send_buf[3];
int recv_buf[3*size]; // Assuming recv_buf is large enough at root
int root = 0;

for (int i = 0; i < 3; ++i) {
    send_buf[i] = rank * 10 + i;
}

// What collective operation here, if root receives {0,1,2, 10,11,12, 20,21,22} in its recv_buf?
// MPI_COLLECTIVE_OPERATION(send_buf, 3, MPI_INT, recv_buf, 3, MPI_INT, root, MPI_COMM_WORLD);

if (rank == root) {
    // print recv_buf
}
// ... (MPI_Finalize)
```
Which `MPI_COLLECTIVE_OPERATION` would result in the `recv_buf` at the `root` process (rank 0) containing `{0, 1, 2, 10, 11, 12, 20, 21, 22}` (assuming `recv_buf` at root is of size `3*size`)?

a) `MPI_Scatter`
b) `MPI_Reduce` with `MPI_SUM`
c) `MPI_Gather`
d) `MPI_Alltoall`

**Answer:** c) `MPI_Gather`

**Justification:**
Let's trace the `send_buf` contents for each rank:
- Rank 0: `send_buf = {0, 1, 2}`
- Rank 1: `send_buf = {10, 11, 12}`
- Rank 2: `send_buf = {20, 21, 22}`

The desired `recv_buf` at root 0 is the concatenation of `send_buf` from rank 0, then rank 1, then rank 2. This is exactly what `MPI_Gather` does. Each process sends its `send_buf` (3 integers), and the root process gathers these contributions in rank order into its `recv_buf`. [Source: 81 (definition and diagram), 83 (code example)]
a) `MPI_Scatter` would take data from the root's `send_buf` and distribute parts of it to all processes.
b) `MPI_Reduce` would combine the elements (e.g., sum them) into a single `send_buf`-sized array at the root.
d) `MPI_Alltoall` would involve each process sending different parts of its `send_buf` to every other process.

---

**12. If `MPI_Allreduce(sdata, rdata, N, MPI_INT, MPI_MAX, MPI_COMM_WORLD)` is called by all processes where each process `rank` has `sdata[i] = 100 - rank` for `i` from 0 to `N-1`. What will be the content of `rdata` on process `rank = 1` after the call (assuming 3 processes, N=1)?**

a) `rdata[0] = 99`
b) `rdata[0] = 100`
c) `rdata[0] = 98`
d) `rdata` will be undefined for `rank = 1` as only the root gets the result.

**Answer:** b) `rdata[0] = 100`

**Justification:**
With N=1 and 3 processes (ranks 0, 1, 2):
- Rank 0: `sdata[0] = 100 - 0 = 100`
- Rank 1: `sdata[0] = 100 - 1 = 99`
- Rank 2: `sdata[0] = 100 - 2 = 98`

`MPI_Allreduce` with `MPI_MAX` will find the maximum value among all `sdata[0]` values from all processes. The maximum is 100. Since it's `MPI_Allreduce`, this result (100) will be placed in `rdata[0]` on *all* processes, including rank 1. [Source: 88, 90 (code example, although it initializes sdata differently, the MPI_MAX logic applies)] The example output in the document [Source: 92] shows all processes receiving the same result from `MPI_Allreduce` with `MPI_MAX`.

---

**13. What is a key difference between `MPI_Reduce` and `MPI_Allreduce`?**

a) `MPI_Reduce` can only perform sum operations, while `MPI_Allreduce` can perform various operations like MAX, MIN, etc.
b) In `MPI_Reduce`, the result is available only on the root process, while in `MPI_Allreduce`, the result is available on all processes in the communicator.
c) `MPI_Allreduce` is significantly less efficient than performing an `MPI_Reduce` followed by an `MPI_Bcast`.
d) `MPI_Reduce` requires non-blocking semantics, whereas `MPI_Allreduce` is blocking.

**Answer:** b) In `MPI_Reduce`, the result is available only on the root process, while in `MPI_Allreduce`, the result is available on all processes in the communicator.

**Justification:**
a) Both `MPI_Reduce` and `MPI_Allreduce` can use the same set of predefined operations (like `MPI_SUM`, `MPI_MAX`, `MPI_MIN`, etc.) and user-defined operations.
b) This is the fundamental difference. `MPI_Reduce` collects the result of the reduction operation onto a single specified `root` process. `MPI_Allreduce` performs the reduction and then makes the result available to *all* processes in the communicator. [Source: 88]
c) While `MPI_Allreduce` can be thought of as an `MPI_Reduce` followed by an `MPI_Bcast`, dedicated implementations of `MPI_Allreduce` are often more optimized than performing the two operations separately.
d) Both `MPI_Reduce` and `MPI_Allreduce` are typically blocking collective operations.

---

**14. Consider the following statements about MPI communicators and groups:**

I. `MPI_COMM_WORLD` is the default communicator that includes all processes launched.
II. It is possible to create custom communicators that group a subset of processes from `MPI_COMM_WORLD`.
III. Communications are always restricted to processes within the same communicator.

Which of these statements are TRUE?

a) I only
b) I and II only
c) II and III only
d) I, II, and III

**Answer:** d) I, II, and III

**Justification:**
I. TRUE. `MPI_COMM_WORLD` is the initial communicator that encompasses all processes started when the MPI program is launched. [Source: 9, 11]
II. TRUE. MPI provides mechanisms to create new communicators from existing ones, often by defining groups of processes. This allows for structuring communication within specific subsets of processes. [Source: 3 (Communicators and groups: "how processes are grouped together")]
III. TRUE. MPI communication operations (both point-to-point and collective) specify a communicator. The operation is confined to the group of processes defined by that communicator. A process in communicator A cannot directly send a message to a process that is only in communicator B using a send call within communicator A. [Source: 3, 20, 22, 73, 75 (all MPI calls specify a communicator)]

---

**15. If a symmetric ping-pong code (both processes `MPI_Send` then `MPI_Recv` to each other) works for a small message size but deadlocks for a large message size, this is likely because:**

a) The MPI library's internal buffers were sufficient to hold the small message, allowing `MPI_Send` to return quickly (eager protocol), but not the large message, forcing `MPI_Send` to block until the matching `MPI_Recv` is posted (rendezvous protocol).
b) The network bandwidth is insufficient for large messages, causing a timeout.
c) `MPI_Recv` has a hidden limit on the message size it can receive.
d) The `MPI_INT` datatype cannot handle large counts, leading to an overflow.

**Answer:** a) The MPI library's internal buffers were sufficient to hold the small message, allowing `MPI_Send` to return quickly (eager protocol), but not the large message, forcing `MPI_Send` to block until the matching `MPI_Recv` is posted (rendezvous protocol).

**Justification:**
This describes the common behavior of MPI implementations regarding message passing protocols.
a) For small messages, MPI libraries often employ an "eager" protocol: the data is quickly copied to an internal MPI buffer, and `MPI_Send` returns, allowing the sending process to continue. The actual data transfer happens in the background. If both processes do this, their sends can return, and they can proceed to `MPI_Recv`. For large messages, these internal buffers may be insufficient. The library then switches to a "rendezvous" protocol, where `MPI_Send` will block until the corresponding `MPI_Recv` has been posted by the receiver, ensuring the receiver is ready to accept the large data. If both attempt `MPI_Send` first with large messages under a rendezvous protocol, they will both block waiting for the other's `MPI_Recv`, leading to a deadlock. [Source: 40, 41, 47 (implicitly, the reason for deadlock with large messages vs. small)]
b) While network bandwidth is a factor in performance, it doesn't directly explain the deadlock logic itself, which is about synchronization and buffering strategies of the send/receive calls. A timeout is a possible consequence of a deadlock, not the primary cause in this context.
c) `MPI_Recv` itself doesn't typically have a "hidden limit" that causes deadlocks in this manner. The issue is the blocking nature of `MPI_Send` for large messages and the symmetric communication pattern.
d) The `MPI_INT` datatype itself is not the issue; it's about the *number* of elements and the total message size interacting with MPI's communication protocols and buffering.

---
