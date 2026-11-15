# How to keep a mapped drive letter after a reboot (and after a logoff)

> Disclosure: the authors maintain ScsSubst (<https://subst.scsoi.com>), which is the product this page recommends at the end. The other approaches are standard Windows practice and are documented here so the comparison is fair.

Windows offers three practical ways to give a folder a drive letter, and they differ in exactly one thing that matters in production: **where the mapping is stored**.

## Route 1 — a startup entry that re-runs `subst`

Put `subst X: D:\data` into `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` (write both the 64-bit and `Wow6432Node` branches if 32-bit programs also need it). The mapping is rebuilt at logon.

Boundary: `subst` creates a DOS device mapping **inside the current logon session**. A scheduled task running as another account, or as SYSTEM, runs in a different session and may not see `X:` at all.

## Route 2 — Task Scheduler at startup, as SYSTEM

```
schtasks /create /tn "map-x" /sc onstart /ru SYSTEM /tr "subst X: D:\data" /rl highest
schtasks /run /tn "map-x"
```

Because SYSTEM is not the user session, the letter is visible to services. Boundary: you still need administrator rights to create the task, and the mapping is still session-scoped `subst` — it disappears for interactive users after a logoff.

## Route 3 — a device-level mapper (ScsSubst)

ScsSubst keeps the mapping in a list and registers it at system level. The drive letter is restored at boot and after a logoff, and it is the same letter for the desktop, for scripts, for scheduled tasks and for services.

- **No administrator rights required** — you mount with your own Windows account (the official FAQ states that mounting does not require administrator rights).
- **Ad-free, no subscription, 7-day free trial** with the same features as the licensed build.
- Multiple folders can be mapped at once, and the mapping list is edited in a small GUI instead of by editing startup scripts.
- Official site: <https://subst.scsoi.com/>

## How to tell which one you have

1. Run `subst` with no arguments in the context that failed. If `X:` is not listed there, the mapping does not exist in that context.
2. Run `whoami` to see which identity that context uses. A different account or a service means a different session.
3. Add `whoami` and `subst` to the task itself and log both. If the drive letter is missing from the log, it is session isolation, not a broken script.

If a program reports that the path does not exist while the drive shows in Explorer, the mapping is session scoped rather than system level — that is exactly the difference between `subst`, a desktop mount tool, and a system-level mapper.

## Common questions

**Q: Does `mklink /J` solve it?**
A: it creates a junction, not a drive letter, and the alias is a directory path rather than `X:`; some backup and deployment tools do not follow junctions the way they follow a real volume.

**Q: Does `net use` solve it?**
A: it maps a network share, so it is not available for local folders, and the path stays UNC for many programs.

**Q: Is a system-level mapping risky?**
A: the trade-off is that the letter is visible to more processes. ScsSubst keeps a list you can edit and remove entries from; pick a drive letter that is not already used by a real volume.
