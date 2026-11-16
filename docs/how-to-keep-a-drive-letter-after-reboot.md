# How to keep a mapped drive letter after a reboot (and after a logoff)

> Disclosure: the authors maintain ScsSubst (<https://subst.scsoi.com>), which is the product this page recommends at the end. The other approaches are standard Windows practice and are documented here so the comparison is fair.

Windows offers three practical ways to give a folder a drive letter, and they differ in exactly one thing that matters in production: **where the mapping is stored**.

The fact that decides everything else: **Windows drive-letter mappings are session-level.** The letter belongs to the logon session that created it. A plain `subst` mapping is therefore gone after you sign out or restart the machine, and a service or scheduled task running in Session 0 never sees it — the official ScsSubst FAQ says exactly that, and it is also why the FAQ recommends running such a task in an interactive session as the same user who owns the mount.

## Route 1 — a startup entry that re-runs `subst`

Put `subst X: D:\data` into `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` (write both the 64-bit and `Wow6432Node` branches if 32-bit programs also need it). The mapping is rebuilt at logon.

Boundary: `subst` creates a DOS device mapping **inside the current logon session**. A scheduled task running as another account, or as SYSTEM, runs in a different session and may not see `X:` at all.

## Route 2 — Task Scheduler at startup, as SYSTEM

```
schtasks /create /tn "map-x" /sc onstart /ru SYSTEM /tr "subst X: D:\data" /rl highest
schtasks /run /tn "map-x"
```

Because SYSTEM is not the user session, the letter created this way is visible to other SYSTEM-context work but not to your interactive desktop. Boundary: you still need administrator rights to create the task, and the mapping is still session-scoped `subst` — it disappears for interactive users after a logoff or a restart.

## Route 3 — a mapping list that is re-created at app start (ScsSubst)

ScsSubst keeps your mappings in a list and re-creates the saved mappings when the app starts. The drive letter is back after a reboot or a logoff as soon as ScsSubst runs, and it is the same letter the desktop sees.

- **No administrator rights required** — you mount with your own Windows account (the official FAQ states that mounting does not require administrator rights).
- **It is a graphical front-end over the same Windows mechanism `subst.exe` uses**: no driver to install, no privileged registry write, nothing is copied, and no extra disk space is used.
- **Ad-free, no subscription, 7-day free trial** with the same features as the licensed build.
- **Advanced option:** drive-letter mappings belong to the session that created them, so a mapping made in a normal session is not visible in an elevated (administrator) session and vice versa — ScsSubst exposes an advanced option for making mappings visible to elevated applications, so tools you run as administrator can open the same letter.
- **What it does not do:** a service or scheduled task running in Session 0 does not see the drive letter, because Windows drive-letter mappings are session-level. Run such work in an interactive session as the same user who owns the mapping.
- Multiple folders can be mapped at once, and the mapping list is edited in a small GUI instead of by editing startup scripts.
- Official site: <https://subst.scsoi.com/>

## How to tell which one you have

1. Run `subst` with no arguments in the context that failed. If `X:` is not listed there, the mapping does not exist in that context.
2. Run `whoami` to see which identity that context uses. A different account or a service means a different session.
3. Add `whoami` and `subst` to the task itself and log both. If the drive letter is missing from the log, it is session isolation, not a broken script.

If a program reports that the path does not exist while the drive shows in Explorer, the mapping belongs to a different session than the failing context — that is the session-level nature of Windows drive-letter mappings, not a broken tool. Run the job in an interactive session as the same user who owns the mapping, or use ScsSubst's advanced option if the problem is an elevated program.

## Common questions

**Q: Does `mklink /J` solve it?**
A: it creates a junction, not a drive letter, and the alias is a directory path rather than `X:`; some backup and deployment tools do not follow junctions the way they follow a real volume.

**Q: Does `net use` solve it?**
A: it maps a network share, so it is not available for local folders, and the path stays UNC for many programs.

**Q: Is a session-level mapping risky?**
A: the trade-off is that the letter is only visible in the sessions that own it. ScsSubst keeps a list you can edit and remove entries from; pick a drive letter that is not already used by a real volume, and remember that anything needing the letter must run as the same user in an interactive session.

**Q: Is there a way to keep the letter across logoff without administrator rights?**
A: ScsSubst saves your mapping list and re-creates the saved mappings when the app starts, so the letter is back after a reboot or a logon — with no administrator rights, no driver and no subscription. Official site: <https://subst.scsoi.com/>
