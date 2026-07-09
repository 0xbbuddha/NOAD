# Attack path - NOAD Core

Progression inspired by a ninja's career. Each arc builds on the
accounts/objects described in `docs/NOAD.md`.

## Arc 1 - Academy (reconnaissance)

Goal: discover the environment without any credentials.

- Anonymous SMB/LDAP/DNS enumeration against `noad.local`.
- Username enumeration via OSINT (see `docs/NOAD.md`, OSINT section): the
  list of Naruto characters on Wikipedia directly yields plausible account
  names in `firstname.lastname` format.
- The internal IT wiki on `anbu-srv01` (`http://192.168.56.11/`) and the
  `\\anbu-srv01\IT` share are accessible to any authenticated user and
  document the infrastructure (first lead toward `svc_iis`).
- `\\mission-srv01\Backups` and `\\mission-srv01\MissionData` shares, also
  readable by any authenticated user.

## Arc 2 - Genin (initial access)

- **AS-REP Roasting (unauthenticated)**: `naruto.uzumaki` has
  `DoesNotRequirePreAuth=$true`. Grab his AS-REP hash
  (`GetNPUsers.py noad.local/ -no-pass -usersfile users.txt` or Rubeus
  `asreproast`) and crack it offline (`Rasengan1!`).
- Once authenticated as `naruto.uzumaki`, explore `academy-ws01` and
  identify the domain's other 3 machines.

## Arc 3 - Chunin (local escalation on academy-ws01)

Several independent vectors, all deployed by the `services_setup` role:

- **AlwaysInstallElevated**: the registry key is enabled in both HKLM and
  HKCU -> any MSI installs with SYSTEM privileges
  (`msiexec /quiet /qn /i malicious.msi`).
- **Unquoted Service Path**: the `NarutoUpdateSvc` service points to
  `C:\Program Files\Naruto Tools\Update Service\svc.exe` without quotes,
  and `Authenticated Users` has `Modify` rights on `C:\Program Files\Naruto
  Tools` -> drop a `Naruto.exe` at the root to hijack the Windows search
  path.
- **GPP cpassword (MS14-025)**: a `Groups.xml` encrypted with the public
  MS14-025 AES key is present in SYSVOL
  (`\\hokage-dc01\SYSVOL\noad.local\Policies\{7F3E9A10-...}\Machine\Preferences\Groups\Groups.xml`),
  decryptable with `gpp-decrypt` or `Get-GPPPassword.ps1` (local account
  `svcdesk`, password `HelpDesk2026!`).
- **Plaintext credentials on the shares**: `deploy_svc_iis.ps1` on
  `\\anbu-srv01\IT` (svc_iis's password), `backup_job.ps1` and
  `mission_db_config.xml` on `mission-srv01` (svc_backup's and svc_sql's
  passwords).

## Arc 4 - Jonin (Active Directory escalation)

- **Kerberoasting**: `svc_ibiki`, `svc_sql`, `svc_mission`, `svc_backup`,
  `svc_iis` and `svc_anbu` all carry SPNs and are reported as
  Kerberoastable by any tool. Request a TGS for each
  (`GetUserSPNs.py noad.local/naruto.uzumaki`) and crack them offline -
  but not all of them are worth the compute: `svc_ibiki` and `svc_iis`
  have long random passwords and are deliberate dead ends, while
  `svc_sql`, `svc_mission`, `svc_backup` and `svc_anbu` have weak,
  crackable ones. Realistic Kerberoasting means triaging which hashes are
  actually worth cracking, not blindly running hashcat on everything.
- **Password reuse**: `svc_sql` and `svc_mission` share the same
  password, as do `svc_anbu` and `itachi.uchiha` - cracking one reveals
  the other. `kotetsu.hagane` and `izumo.kamizuki` also share their
  password.
- **ACL abuse**:
  - `sasuke.uchiha` holds `GenericAll` over `sakura.haruno` (password
    reset or Shadow Credentials).
  - `mizuki.touji` holds a forgotten `GenericWrite` over
    `konohamaru.sarutobi`.
  - `kabuto.yakushi` holds a `GenericAll` over the `GG_ProjectPhoenix`
    group - a deliberate dead end (BloodHound shows a path that leads
    nowhere).
- **RBCD (Resource-Based Constrained Delegation)**: the `svc_inventory`
  service account holds a forgotten `GenericWrite` over the
  `mission-srv01$` computer object. Once compromised (weak password), it
  allows configuring `msDS-AllowedToActOnBehalfOfOtherIdentity` and
  obtaining SYSTEM access on `mission-srv01` via `getST.py`/Rubeus `s4u`.
- **Unconstrained Kerberos delegation**: `anbu-srv01` is marked
  `TRUSTED_FOR_DELEGATION`. By coercing `hokage-dc01` into authenticating
  to `anbu-srv01` (PrinterBug/PetitPotam), the DC machine account's TGT
  (or an admin's) ends up cached in LSASS on `anbu-srv01`.
- **Excessive local administrators**: `zabuza.momochi` (mercenary retained
  for the Wave Country bridge security detail, killed in action) and the
  `GG_TempAdmins` group are local administrators of `mission-srv01`, never
  revoked after his death / after the migration.

## Arc 5 - ANBU (advanced techniques)

- **Shadow Credentials**: `danzo.shimura` holds a `GenericWrite` (covers
  writing `msDS-KeyCredentialLink`) over `hiruzen.sarutobi`. From a
  `danzo.shimura` foothold, add a shadow public key on `hiruzen.sarutobi`
  (Whisker/Rubeus) and authenticate as him via PKINIT without ever
  knowing his password.
- **Forgotten group nesting**: `hiruzen.sarutobi` is a member of
  `GG_LegacyAdmins`, itself nested into **Domain Admins** from an old
  delegation that was never cleaned up - compromising `hiruzen.sarutobi`
  (via the Shadow Credentials above) directly yields Domain Admin rights.
- **DCSync**: `svc_monitoring` holds
  `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` rights
  (an over-provisioned monitoring tool). Once compromised, it allows a
  `secretsdump.py noad.local/svc_monitoring@hokage-dc01 -just-dc` without
  ever touching LSASS -> full compromise of `noad.local`.

## Narrative twist: Edo Tensei

`hiruzen.sarutobi` (Third Hokage) **died in battle against Orochimaru**,
so his account is disabled (`ACCOUNTDISABLE`) but was never deleted. His
`GG_LegacyAdmins` membership - nested into Domain Admins - still exists.
`danzo.shimura`'s `GenericWrite` over him covers **both**
`userAccountControl` and `msDS-KeyCredentialLink`, which means the attack
has two required steps rather than one:

1. Clear the `ACCOUNTDISABLE` flag (`Set-ADAccountControl -Identity
   hiruzen.sarutobi -Enabled $true` or the equivalent LDAP write) -
   "reanimate" the account. A classic password-based logon is still
   impossible without knowing his password.
2. Add a shadow public key (Whisker/Rubeus `shadowcreds`) on
   `hiruzen.sarutobi` and authenticate as him via PKINIT.

Technically, this is exactly what an Edo Tensei represents: a dead
account is brought back and answers to whoever cast the jutsu (Danzo) -
and it directly hands over Domain Admin.

## Suggested tools

- Enumeration: BloodHound (SharpHound or bloodhound-python), run from
  `academy-ws01` or a separate attack machine on the same private
  network.
- OSINT: `curl`/`grep`/`awk` against Wikipedia (see `docs/NOAD.md`).
- Roasting: Impacket (`GetNPUsers.py`, `GetUserSPNs.py`), Rubeus.
- GPP: `gpp-decrypt` (Kali) or `Get-GPPPassword.ps1`.
- Delegation/tickets/RBCD: Mimikatz, Rubeus, Impacket
  (`getST.py`, `secretsdump.py`).
- Shadow Credentials: Whisker, Rubeus (`asktgt /getcredentials`).
