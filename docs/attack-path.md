# Attack path - NOAD Core

Inspired by how [GOAD](https://orange-cyberdefense.github.io/GOAD/) structures
its scenarios: distinct entry points, each requiring a genuinely different
technique, rather than a single foothold that unlocks everything. Three
independent origins are described below; each leads to `mission-srv01` and/or
Domain Admin through a different technique class. You don't need to complete
all three - pick one and follow it through.

A note on realism before the paths: **Kerberoasting is universally
available** to any authenticated domain account, by design of Kerberos itself
- there's no way to "gate" that without breaking how AD actually works, and
GOAD doesn't pretend otherwise either. So yes, `naruto.uzumaki` alone
(Origin A) can Kerberoast `svc_backup` and land local admin on `mission-srv01`
in five minutes, skipping Origins B and C entirely. That's a legitimate
shortcut, not a bug - real environments have these too. What actually keeps
the three origins from collapsing into one are the **hard walls**:
`svc_inventory`, `svc_monitoring` and `svc_print` carry **no SPN** (not
Kerberoastable, full stop), ACL grants only ever benefit their specific
trustee, and RBCD can only be configured by whoever holds the
`msDS-AllowedToActOnBehalfOfOtherIdentity` write right - nobody else. No
password is shared between the "jutsu-themed" account group and the
"generic IT" account group, so cracking one wordlist never hands you the
other for free.

## Arc 1 - Academy (reconnaissance, common to all origins)

- Anonymous SMB/LDAP/DNS enumeration against `noad.local` (no anonymous bind
  succeeds - real AD behavior).
- Username enumeration via OSINT (see `docs/NOAD.md`): the list of Naruto
  characters on Wikipedia yields plausible `firstname.lastname` accounts.
- The internal IT wiki on `anbu-srv01` and the `\\mission-srv01\*` shares
  need an authenticated session first - there's nothing to grab pre-auth
  beyond usernames and service banners.

## Origin A - AS-REP Roasting -> the Genin/Team 7 chain

**Technique: unauthenticated AS-REP roasting, then a themed password
mutation.**

1. `GetNPUsers.py noad.local/ -no-pass -usersfile naruto_users.txt` finds
   `naruto.uzumaki` (`DoesNotRequirePreAuth`). Crack offline -> `Rasengan1!`.
2. The password is his signature technique + a digit + `!`. Scrape the
   jutsu/technique names off the same character pages (a second, distinct
   OSINT pass) and spray that pattern across the Genin/Jonin/ANBU
   roster - this is a themed wordlist, not rockyou, and it only hits
   accounts that actually follow the convention: `sasuke.uchiha`
   (`Sharingan1!`), `kakashi.hatake` (`Sharingan2!`), `hinata.hyuga`
   (`Byakugan1!`), and others. It will **not** hit `svc_inventory` or
   `svc_monitoring` - those don't use jutsu names.
3. From `sasuke.uchiha`: `GenericAll` -> reset `sakura.haruno`.
4. From `sakura.haruno`: `WriteOwner` on `shizune.kato` - take ownership
   (`owneredit.py`), grant yourself `GenericAll`, reset her password.
5. From `shizune.kato`: `WriteDacl` on `GG_TempAdmins` - write yourself an
   ACE (`dacledit.py` / `Add-DomainObjectAcl`), add yourself to the group.
6. `GG_TempAdmins` is local admin on `mission-srv01`. Done.

Each individual ACL looks like ordinary "medical staff managing each
other's records" noise; only a full BloodHound path query from
`sasuke.uchiha` reveals it reaches `mission-srv01`.

## Origin B - Generic password spray -> Kerberoasting -> the Kage Bunshin (RBCD)

**Technique: a spray of generic IT/corporate-style guesses (unrelated to
jutsu names) directly against every enumerated username - no prior
credential required, since Kerberos pre-auth attempts (`kerbrute`) don't
need one.**

1. Spray role-based guesses (`MissionDesk1!`, `InventoryScan1!`,
   `Backup2026!`-style patterns derived from account descriptions/naming) -
   hits `kotetsu.hagane`/`izumo.kamizuki` (Mission desk).
2. From there, `\\mission-srv01\Backups` and `\MissionData` are readable:
   `backup_job.ps1` and `mission_db_config.xml` leak `svc_backup`'s and
   `svc_sql`'s plaintext passwords directly - no cracking needed.
3. Kerberoast from this foothold: `svc_sql`, `svc_mission` (same password,
   reuse confirmed), `svc_anbu` (reveals `itachi.uchiha`'s password too -
   a bonus lead, not required for this path).
4. The same role-based spray also hits `svc_inventory`
   (`InventoryScan1!`) directly - this is the **only** way to get it, since
   it has no SPN and its password isn't leaked anywhere.
5. **The Kage Bunshin**: `ms-DS-MachineAccountQuota` is untouched (default
   `10`) - any domain user, even the very first one compromised, can
   register up to 10 fake computer accounts:
   ```bash
   addcomputer.py -computer-name 'KAGEBUNSHIN$' -computer-pass 'Passw0rd123!' \
     noad.local/kotetsu.hagane:MissionDesk1!
   ```
6. As `svc_inventory`, configure RBCD on `mission-srv01$` to trust the
   clone:
   ```bash
   rbcd.py -delegate-to 'mission-srv01$' -delegate-from 'KAGEBUNSHIN$' \
     -action write noad.local/svc_inventory:InventoryScan1!
   ```
7. Impersonate anyone via S4U2Self/S4U2Proxy and get a ticket for
   `mission-srv01`:
   ```bash
   getST.py -spn cifs/mission-srv01.noad.local -impersonate Administrator \
     -dc-ip 192.168.56.10 'noad.local/KAGEBUNSHIN$:Passw0rd123!'
   ```

You don't clone someone else's identity - you clone *yourself*, and a
forgotten delegation right does the rest.

## Origin C - Generic spray -> straight to Domain Admin

**Technique: the same role-based spray as Origin B, but aimed at the two
keystone Tier-0-adjacent accounts instead. Fully independent of Origins A
and B.**

- `svc_monitoring` (`Monitor2026!`) holds
  `DS-Replication-Get-Changes` + `-All` (an over-provisioned monitoring
  tool, no SPN, never leaked): `secretsdump.py
  noad.local/svc_monitoring@hokage-dc01 -just-dc` -> full domain compromise.
- `danzo.shimura` (`RootShadow1!`) holds `GenericWrite` over
  `hiruzen.sarutobi` - see **Edo Tensei** below.

### The Edo Tensei twist

`hiruzen.sarutobi` (Third Hokage) **died in battle against Orochimaru**, so
his account is disabled (`ACCOUNTDISABLE`) but was never deleted. He's
still a member of `GG_LegacyAdmins`, which holds **DCSync rights directly**
(`DS-Replication-Get-Changes` + `-All`) - deliberately *not* nested into
Domain Admins, exactly like a real forgotten backdoor would avoid
protected-group membership to stay off the radar (see the AdminSDHolder
note below). `danzo.shimura`'s `GenericWrite` covers **both**
`userAccountControl` and `msDS-KeyCredentialLink` (kept unscoped on purpose
- a property-scoped grant would only cover one of the two and break this):

1. Clear `ACCOUNTDISABLE` (`Set-ADAccountControl -Enabled $true`) -
   "reanimate" the account. A password-based logon is still impossible.
2. Add a shadow public key (Whisker/Rubeus `shadowcreds`) and authenticate
   via PKINIT without ever knowing his password.
3. `secretsdump.py noad.local/hiruzen.sarutobi@hokage-dc01 -just-dc` -
   DCSync via `GG_LegacyAdmins`.

Technically, this is exactly what an Edo Tensei represents: a dead account
is brought back and answers to whoever cast the jutsu - and it hands over
full domain compromise through the forgotten `GG_LegacyAdmins` grant.

**Why not just nest `GG_LegacyAdmins` into Domain Admins?** Membership in
Domain Admins (even indirect, via a nested group) flags an account as
"protected" by AD (`adminCount=1`), and the SDProp process silently resets
its ACL to the AdminSDHolder template roughly every 60 minutes - which
would wipe `danzo.shimura`'s `GenericWrite` before an attacker could ever
use it. Granting DCSync rights directly (the same mechanism as
`svc_monitoring`) reaches the same outcome without ever touching a
protected group, so the ACE actually survives. This mirrors how real
"forgotten" privilege escalation paths are built in practice - Domain
Admins nesting gets noticed and cleaned up (or protected by AD itself);
a directly-delegated right on an obscure legacy group does not.

**PKINIT requires the `NOADCA` Enterprise CA on `hokage-dc01`** (installed
specifically so this works - no vulnerable templates, see `docs/NOAD.md`
Scope section).

## Independent bonus: unconstrained delegation

Available from **any** authenticated foothold (Origin A, B or C), since it
only needs network position, not a specific account: `anbu-srv01` is marked
`TRUSTED_FOR_DELEGATION`. Coercing `hokage-dc01` into authenticating to it
(PrinterBug/PetitPotam) puts the DC machine account's TGT in LSASS on
`anbu-srv01` - pass-the-ticket -> DCSync, a fourth road to full compromise.

## Decoys (BloodHound noise, deliberately going nowhere)

- `mizuki.touji` holds `ForceChangePassword` (not a blanket write) over
  `konohamaru.sarutobi` - you can reset his password, and that's it.
- `kabuto.yakushi` holds `GenericAll` over `GG_ProjectPhoenix` - an
  archived, empty group.
- `GG_Archive` - pure noise, no rights anywhere.
- `svc_ibiki` and `svc_iis` are Kerberoastable but carry long random
  passwords - don't waste hashcat time on them (`svc_iis`'s real password
  does leak in `\\anbu-srv01\IT`, independent of Kerberoasting).
- `GG_FormerHokage` groups the four historical Hokage
  (`hashirama.senju`, `tobirama.senju`, `minato.namikaze`,
  `hiruzen.sarutobi`) - all disabled, all deceased in-lore. Only
  `hiruzen.sarutobi` is actually reachable, through `GG_LegacyAdmins`
  (a separate membership). The other three have no ACL grants pointing
  at them anywhere - they're in the domain for the story, not the attack
  path.

## Suggested tools

- Enumeration: BloodHound (SharpHound or bloodhound-python).
- OSINT: `curl`/`grep`/`awk` against Wikipedia (see `docs/NOAD.md`), twice -
  once for usernames, once for jutsu/technique names.
- Spraying: `kerbrute password` (no prior credential needed).
- Roasting: Impacket (`GetNPUsers.py`, `GetUserSPNs.py`), Rubeus.
- ACL abuse: Impacket `owneredit.py`/`dacledit.py`, PowerView
  `Set-DomainObjectOwner`/`Add-DomainObjectAcl`.
- RBCD/Kage Bunshin: Impacket `addcomputer.py`, `rbcd.py`, `getST.py`.
- Delegation/tickets: Mimikatz, Rubeus, `secretsdump.py`.
- Shadow Credentials: Whisker, Rubeus (`asktgt /getcredentials`).
