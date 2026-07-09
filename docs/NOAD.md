# NOAD - Naruto Oriented Active Directory

Deliberately vulnerable Active Directory lab, Naruto-themed, inspired by
GOAD but with its own identity: "NOAD Core" represents only Konohagakure
(the Hidden Leaf Village), designed to be extended later by other
"villages" (see the end of this document).

100% reproducible deployment via Vagrant + VirtualBox + Ansible, no custom
image needed (no Packer, no ISO to prepare).

See also `docs/architecture.md` (full thematic mapping) and
`docs/attack-path.md` (Arc1 -> Arc5 attack progression).

## Infrastructure

| Item            | Value                                                       |
|------------------|----------------------------------------------------------------|
| Hypervisor       | VirtualBox                                                      |
| Orchestrator     | Vagrant (`Vagrantfile` at the repo root)                        |
| Provisioning     | Ansible (`ansible/site.yml`, WinRM)                             |
| Deployment host  | `sysadmin@192.168.1.21` (EndeavourOS/Arch)                      |
| VM network       | VirtualBox `private_network`, subnet `192.168.56.0/24`          |
| AD domain        | `noad.local` (NetBIOS `NOAD`)                                    |

### Machines (NOAD Core - 4 VMs, ~10 GB RAM)

| Name            | IP             | Vagrant box                                     | RAM     | vCPU | Role                                                       |
|-----------------|----------------|---------------------------------------------------|---------|------|--------------------------------------------------------------|
| hokage-dc01     | 192.168.56.10  | `gusztavvargadr/windows-server-2022-standard`     | 3 GB    | 2    | Domain controller (Tier 0), DNS, first DC of the forest       |
| anbu-srv01      | 192.168.56.11  | `gusztavvargadr/windows-server-2022-standard`     | 2.5 GB  | 2    | IT server (IIS + file share), unconstrained Kerberos delegation |
| mission-srv01   | 192.168.56.12  | `gusztavvargadr/windows-server-2022-standard`     | 2.5 GB  | 2    | Business server (simulated SQL/backup/applications)            |
| academy-ws01    | 192.168.56.13  | `gusztavvargadr/windows-10`                       | 2 GB    | 2    | User workstation, entry point of the attack chain              |

RAM reduced compared to the original plan (4/4/4/3 GB) to fit a host with
15 GB total, leaving headroom for the host itself + VirtualBox.

### Why two member servers?

Splitting IT (`anbu-srv01`) and business (`mission-srv01`) roles instead of
a single generic server allows for several distinct service accounts,
several SPNs, several ACLs and several Kerberoasting scenarios - without
adding a VM. It's closer to a real SME than a 2-machine lab.

### Deployment

```bash
cd ~/naruto-lab
vagrant up                                    # creates and starts the 4 VMs
cd ansible
ansible-playbook site.yml -i inventory/hosts.yml
```

Play order in `site.yml`: DC promotion -> OU/group/user/service account
creation -> domain join for member servers/workstations -> applying the
deliberate vulnerabilities -> deploying realistic services and artifacts
(IIS, shares, GPP, local escalation).

## Active Directory organization

```
NOAD.LOCAL
├── OU=Village
│   ├── OU=Administration   (Hokage's office)
│   ├── OU=Academy          (students, team senseis)
│   ├── OU=ANBU             (elite IT/operations)
│   ├── OU=Medical          (medical corps)
│   ├── OU=Mission          (mission assignment desk)
│   ├── OU=Finance          (advisors/budget)
│   ├── OU=Research         (scientific ninja tools)
│   └── OU=IT               (IT support)
├── OU=Servers
├── OU=Workstations
├── OU=Service Accounts
└── OU=Privileged           (sensitive accounts/groups isolated)
```

## Groups

| Group              | Type       | Role                                                       |
|----------------------|------------|-----------------------------------------------------------------|
| GG_Hokage          | Tier 0     | **Legitimate** nesting into Domain Admins (Tsunade)               |
| GG_ANBU            | Tier 1     | Local administrators of `anbu-srv01`                              |
| GG_Jonin           | Tier 1     | Workstation administrators                                        |
| GG_Chunin          | Standard   | Intermediate users                                                 |
| GG_Genin           | Standard   | Standard users                                                     |
| GG_Medical / GG_Mission / GG_Finance / GG_Research / GG_IT | Functional | Departments |
| GG_Backup / GG_SQL / GG_Helpdesk | Functional | Technical operations |
| GG_Team7 / GG_Team8 / GG_Team10 / GG_TeamGai | Cosmetic | Teams (realistic BloodHound noise) |
| GG_LegacyAdmins    | **Trap**   | **Forgotten** nesting into Domain Admins (never-cleaned-up delegation) |
| GG_TempAdmins      | **Trap**   | Local administrator of `mission-srv01`, "temporary" access never revoked |
| GG_ProjectPhoenix  | **Trap**   | Deliberate dead end (target of a GenericAll leading nowhere)       |
| GG_Archive         | **Trap**   | Dead group, pure BloodHound noise                                  |

## Users (37 accounts)

`sAMAccountName` convention: **firstname.lastname** in lowercase - see the
OSINT section below, this is intentional.

| Account                 | OU              | Notable groups                    | Notable trait |
|-------------------------|------------------|--------------------------------------|-------------------|
| naruto.uzumaki          | Administration   | GG_Genin, GG_Team7                    | `DONT_REQ_PREAUTH` -> AS-REP roastable |
| shikamaru.nara          | Administration   | GG_Jonin, GG_Team10                   | - |
| tsunade.senju           | Administration   | GG_Hokage, GG_Medical                  | De facto Domain Admin (GG_Hokage) |
| temari.sabaku           | Administration   | GG_Genin                               | **Disabled** account (former Suna liaison) |
| jiraiya.sannin          | Administration   | GG_Jonin                               | - |
| kakashi.hatake          | ANBU             | GG_Jonin, GG_ANBU, GG_Team7             | Local administrator of `academy-ws01` |
| yamato.tenzo            | ANBU             | GG_ANBU                                | - |
| sai.shimura             | ANBU             | GG_ANBU                                | - |
| itachi.uchiha           | ANBU             | GG_ANBU                                | Password reused by `svc_anbu` |
| asuma.sarutobi          | Academy          | GG_Jonin, GG_Team10                      | Team 10 sensei |
| kurenai.yuhi            | Academy          | GG_Jonin, GG_Team8                       | Team 8 sensei |
| gai.maito               | Academy          | GG_Jonin, GG_TeamGai                     | Team Gai sensei |
| iruka.umino             | Academy          | GG_Chunin                               | - |
| konohamaru.sarutobi     | Academy          | GG_Genin                                | Target of `mizuki.touji`'s forgotten GenericWrite |
| ebisu.tokuma            | Academy          | GG_Chunin                               | - |
| sasuke.uchiha           | Academy          | GG_Genin, GG_Team7                      | `GenericAll` over `sakura.haruno` |
| sakura.haruno           | Academy          | GG_Genin, GG_Team7, GG_Medical           | Target of sasuke's GenericAll |
| hinata.hyuga            | Academy          | GG_Genin, GG_Team8                      | - |
| kiba.inuzuka            | Academy          | GG_Genin, GG_Team8                      | - |
| shino.aburame           | Academy          | GG_Genin, GG_Team8                      | - |
| ino.yamanaka            | Academy          | GG_Genin, GG_Team10                     | - |
| choji.akimichi          | Academy          | GG_Genin, GG_Team10                     | - |
| rock.lee                | Academy          | GG_Genin, GG_TeamGai                    | - |
| neji.hyuga              | Academy          | GG_Genin, GG_TeamGai                    | - |
| tenten.mitsu            | Academy          | GG_Genin, GG_TeamGai                    | - |
| mizuki.touji            | Academy          | GG_Chunin                               | Forgotten `GenericWrite` over `konohamaru.sarutobi` |
| shizune.kato            | Medical          | GG_Medical, GG_Chunin                    | - |
| kotetsu.hagane          | Mission          | GG_Mission, GG_Chunin                    | Same password as `izumo.kamizuki` |
| izumo.kamizuki          | Mission          | GG_Mission, GG_Chunin                    | Same password as `kotetsu.hagane` |
| zabuza.momochi          | Mission          | GG_Genin                                | Mercenary retained for Wave Country bridge security, killed in action, local admin on `mission-srv01` never revoked |
| homura.mitokado         | Finance          | GG_Finance, GG_Chunin                    | - |
| koharu.utatane          | Finance          | GG_Finance, GG_Chunin                    | - |
| kabuto.yakushi          | Research         | GG_Research, GG_Genin                    | `GenericAll` (dead end) over `GG_ProjectPhoenix` |
| ibiki.morino            | IT               | GG_IT, GG_Helpdesk                       | Human account, distinct from the `svc_ibiki` service account |
| anko.mitarashi          | IT               | GG_IT, GG_Helpdesk                       | - |
| hiruzen.sarutobi        | Privileged       | GG_LegacyAdmins                          | **Disabled account** (died in battle) - Domain Admin via forgotten nesting, "resurrection" possible via Edo Tensei (see Arc 5) |
| danzo.shimura           | Privileged       | GG_IT                                    | `GenericWrite` (Shadow Credentials) over `hiruzen.sarutobi` |

## Service accounts (OU=Service Accounts)

| Account           | SPN                                                 | Group       | Notable trait |
|-------------------|------------------------------------------------------|-------------|-------------------|
| svc_ibiki         | `HTTP/anbu-srv01.noad.local`                          | GG_ANBU     | Kerberoastable but a **dead end** - strong random password |
| svc_sql           | `MSSQLSvc/mission-srv01.noad.local:1433`              | GG_SQL      | Kerberoastable, password reused by `svc_mission` |
| svc_mission       | `MissionApp/mission-srv01.noad.local`                 | GG_Mission  | Kerberoastable, same password as `svc_sql` |
| svc_backup        | `BackupExec/mission-srv01.noad.local`                 | GG_Backup   | Kerberoastable, local admin of `mission-srv01` |
| svc_iis           | `HTTP/anbu-srv01.noad.local:8080`                     | GG_IT       | IIS application pool identity - Kerberoastable but a **dead end** (strong random password); the real password still leaks in plaintext via the `\\anbu-srv01\IT` share |
| svc_print         | -                                                      | GG_IT       | Tied to the print spooler (PrinterBug) |
| svc_inventory     | -                                                      | GG_IT       | Forgotten `GenericWrite` over the `mission-srv01$` computer object -> RBCD |
| svc_anbu          | `ANBUTools/anbu-srv01.noad.local`                     | GG_ANBU     | Kerberoastable, same password as `itachi.uchiha` |
| svc_monitoring    | -                                                      | GG_IT       | Over-privileged: DCSync rights (`DS-Replication-Get-Changes[-All]`) |

All passwords (users and service accounts) are defined in
`ansible/group_vars/all/naruto_universe.yml`.

## Infrastructure credentials

| Account                   | Password          | Scope                                             |
|---------------------------|-------------------|--------------------------------------------------|
| `vagrant`                 | `vagrant`         | Local admin on every VM; becomes Domain Admin once the DC is promoted |
| Administrator (RID 500)   | `NarutoLab2026!`  | Built-in local admin of hokage-dc01                |
| Safe Mode Password (DSRM) | `NarutoDSRM2026!` | Directory Services Restore Mode                    |

## Remote access (WireGuard)

`192.168.56.0/24` is a VirtualBox private network, only reachable from the
deployment host itself. A WireGuard tunnel on the host exposes it to an
attack machine on the same LAN, without any network changes on the
Windows VMs (the host masquerades the tunnel traffic as its own
`192.168.56.1` address, so the VMs just see and answer local traffic).

| Item              | Value                          |
|--------------------|--------------------------------|
| Server             | `192.168.1.21:51820` (UDP)      |
| Tunnel subnet       | `10.66.66.0/24`                 |
| Server config       | `/etc/wireguard/wg0.conf` (host) |
| Service             | `wg-quick@wg0` (enabled, starts on boot) |
| Split tunnel        | Only `192.168.56.0/24` (the lab) is routed through the tunnel |

Client config (attack machine), `/etc/wireguard/noad.conf`:

```ini
[Interface]
PrivateKey = <client private key, see ~/.wg-noad/client_private.key on the host>
Address = 10.66.66.2/24

[Peer]
PublicKey = M1pj9tvatDFm4w1l5k6uf0lcYv8qf9GfSC+xbvqSj3Y=
Endpoint = 192.168.1.21:51820
AllowedIPs = 192.168.56.0/24
PersistentKeepalive = 25
```

```bash
sudo wg-quick up noad     # bring the tunnel up
sudo wg-quick down noad   # tear it down
```

## Attack progression (arcs)

See `docs/attack-path.md` for the full technical detail.

- **Arc 1 - Academy**: reconnaissance (SMB/LDAP/DNS, the IT wiki on
  `anbu-srv01`, shares).
- **Arc 2 - Genin**: initial access (AS-REP roasting on `naruto.uzumaki`).
- **Arc 3 - Chunin**: local escalation on `academy-ws01`
  (AlwaysInstallElevated, an unquoted service path, a GPP cpassword in
  SYSVOL, plaintext credentials on the shares).
- **Arc 4 - Jonin**: AD escalation (Kerberoasting, ACL abuse, reused
  passwords, unconstrained delegation, RBCD).
- **Arc 5 - ANBU**: DCSync via `svc_monitoring`, Shadow Credentials
  (`danzo.shimura` -> `hiruzen.sarutobi`), the forgotten nesting of
  `GG_LegacyAdmins` into Domain Admins.

## OSINT: enumerating usernames

Thanks to the `firstname.lastname` convention, a publicly available list
of Naruto character names directly yields plausible account names -
worth documenting as-is in the writeup:

```bash
curl -s https://en.wikipedia.org/wiki/List_of_Naruto_characters \
  | grep -oP 'title="\K[^"]+(?=")' \
  | grep -E '^[A-Z][a-z]+ [A-Z][a-z]+' \
  | awk '{print tolower($1 "." $2)}' \
  | sort -u > naruto_users.txt
```

## Scope of this version (what is NOT implemented)

To keep a reliable single-pass deployment, the following items are
**deliberately** out of scope and documented as future extensions:

- **Real SQL Server** on `mission-srv01`: the "SQL" role is simulated via
  an SPN (`MSSQLSvc`), file shares and configuration files with plaintext
  credentials - no SQL Server instance is actually installed.
- **ADCS (ESC1/ESC4/ESC8)**: requires a full PKI, deliberately deferred to
  a future extension.
- **Windows LAPS** (schema extension + GPO): same, deferred.

## Future extensions (not built)

- **Suna**: second domain/forest with a trust relationship, trust abuse.
- **Kiri**: hardened environment (restrictive GPOs, tiered administration).
- **Kumo**: ADCS/PKI infrastructure with certificate abuse.
- **Akatsuki**: "rogue" infrastructure (C2, Shadow IT) simulating an
  adversary already established in the environment.
- **Orochimaru Labs**: advanced experimental techniques (ESC13+, complex
  delegations).

The passwords in this repo are deliberately stored in plaintext - local
isolated lab only, never reproduce this outside a dedicated environment.
