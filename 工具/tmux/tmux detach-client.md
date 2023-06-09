
---
tag: tmux
---

With `tmux list-client`, you can list all clients connected to `tmux` sessions. For instance:

```shell
$ tmux list-client
/dev/pts/6: 0 [25x80 xterm] (utf8)
/dev/pts/8: 0 [25x80 xterm] (utf8)
```

From this point, you can choose to detach a specified client, or all clients of a specified session. Say I want to detach everyone connected to `session 0`:

```shell
$ tmux detach-client -s 0
```

Then, you can attach the session so the size will be yours.

Actually, all that can be done with `tmux attach -d` (the -d option force all other clients to detach).