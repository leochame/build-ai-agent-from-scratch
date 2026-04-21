# Agent Team

## 1. What an Agent Team Is

An agent team can be described as:

one leader, zero or more teammates, and a shared state plus messaging protocol.

The key idea is not "open a few temporary agents," but "multiple members share tasks, communicate, and collaborate over time."

![img.png](images/img.png)

## 2. How an Agent Team Works

### 2.1 Team Creation

When a team is created, the system usually prepares three kinds of data:

- a team file with member metadata
- a shared task list
- the leader's current in-memory team view

The team file stores persistent facts such as:

- who belongs to the team
- who the leader is
- member types
- whether a member is active
- which session a member belongs to

The shared task list acts as a common work pool. Members can create, claim, and complete tasks against the same shared state.

### 2.2 Coordination

Collaboration usually depends on two main channels:

- a shared task list for visible work
- a mailbox for control messages

The mailbox is especially important for messages like:

- shutdown requests
- plan approvals
- permission requests
- direct team communication

This means coordination is not based on shared memory. It is based on shared files and explicit message passing.

Priority ordering matters too. Control messages should usually be processed before normal chatter, otherwise the team may keep running in the wrong direction.

### 2.3 Process vs In-Process Teammates

There are two common execution shapes:

- process teammates run in separate processes or sessions
- in-process teammates run concurrently inside the same process

The distinction affects how identity and isolation are handled. Process teammates often receive identity through startup parameters, while in-process teammates rely on runtime-local isolation.

### 2.4 Concurrency Control

A good team design also needs concurrency protection:

- execution contexts should be isolated
- task claiming should avoid duplicate work
- file conflicts should be controlled

When multiple agents edit code, worktrees are often safer than a single shared directory because they shift conflicts from runtime file overwrites to version-control merges.

## 3. Team Shutdown

A clean shutdown usually happens in two steps:

1. send shutdown requests to active teammates
2. delete the team only after all members have stopped

If active members still exist, cleanup should be blocked until they are closed properly.
