---
title: Jupyter kernel sub-shells
authors: David Brochart (@davidbrochart), Sylvain Corlay (@SylvainCorlay)
issue-number: XX
pr-number: XX
date-started: XXXX-XX-XX
---

# Summary



# Motivation

Users have been asking for ways to interact with a kernel while it is busy executing CPU-bound code,
for the following reasons:
- inspect the kernel's state to check the progress or debug a long-running computation.
- visualize some intermediary result before the final result is computed.

Unfortunately, it is currently not possible to do so because the kernel cannot process other
[execution requests](https://jupyter-client.readthedocs.io/en/stable/messaging.html#execute) until
it is idle. The goal of this JEP is to offer a way to run code concurrently.

# Proposed Enhancement

The [kernel protocol](https://jupyter-client.readthedocs.io/en/stable/messaging.html) only allows
for one
[shell channel](https://jupyter-client.readthedocs.io/en/stable/messaging.html#messages-on-the-shell-router-dealer-channel)
where execution requests are queued. Accepting other shells would allow users to connect to a kernel
and submit execution requests that would be processed in parallel.

We propose to allow the creation of optional "sub-shells", in addition to the current "main shell".
This will be made possible by adding new message types to the
[control channel](https://jupyter-client.readthedocs.io/en/stable/messaging.html#messages-on-the-control-router-dealer-channel)
for:
- creating a sub-shell,
- deleting a sub-shell,
- listing existing sub-shells.

A kernel could choose to implement sub-shells by running them in separate threads, although it is
not enforced.

A sub-shell should be advertised to the front-end with a new kernel ID, so that any client can
connect to it (console, notebook, etc.). Conceptually, clients see a new kernel, although it is
actually the same kernel that the main shell is connected to.

Essentially, a client connecting through a sub-shell should see no difference with a connection
through the main shell, and it does not need to be aware of it. However, a front-end should provide
some visual information indicating that the kernel execution mode offered by the sub-shell has to be
used at the user's own risks. In particular, because sub-shells may be implemented with threads, it
is the responsibility of users to not corrupt the kernel state with non thread-safe instructions.

# New control channel messages

## Create sub-shell

Message type: `create_subshell_request`:

```py
content = {
    # The ID of the kernel that the main shell refers to (optional).
    # If provided, the kernel ID in the reply should append an ID to this ID, e.g.:
    # 'ab89' -> 'ab89_0'
    'kernel_id': str
}
```

Message type: `create_subshell_reply`:

```py
content = {
    # 'ok' if the request succeeded or 'error', with error information as in all other replies.
    'status': 'ok',

    # The ID of the kernel that the sub-shell refers to, e.g.: 'ab89_0'
    'kernel_id': str
}
```

## Delete sub-shell

Message type: `delete_subshell_request`:

```py
content = {
    # The ID of the kernel that the sub-shell refers to.
    'kernel_id': str
}
```

Message type: `delete_subshell_reply`:

```py
content = {
    # 'ok' if the request succeeded or 'error', with error information as in all other replies.
    'status': 'ok',
}
```

## List sub-shells

Message type: `list_subshells_request`: this message has no content.

Message type: `list_subshells_reply`:

```py
content = {
    # A list of kernel IDs created by sub-shells.
    'kernel_ids': [str]
}
```
