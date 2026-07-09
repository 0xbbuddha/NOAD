# Architecture - NOAD (Naruto Oriented Active Directory)

Thematic mapping on names/hierarchy - the underlying mechanisms are those
of a classic Active Directory lab (inspired by GOAD). For the full list of
users/groups/service accounts and credentials, see `docs/NOAD.md`.

## Domain

| Naruto                | Active Directory                                    |
|------------------------|----------------------------------------------------|
| Konohagakure            | `noad.local` forest/domain (NetBIOS `NOAD`)         |
| Hokage                  | `GG_Hokage` - Tier 0, legitimate nesting into Domain Admins |
| ANBU                    | `GG_ANBU` - Tier 1, administrators of `anbu-srv01`  |
| Jonin                   | `GG_Jonin` - Tier 1, workstation administrators      |
| Chunin                  | `GG_Chunin` - intermediate users                     |
| Genin                   | `GG_Genin` - standard users                          |

Real department -> Naruto "flavor" mapping: Administration = Hokage's
office, IT = ANBU, HR = Academy, SOC = Intelligence Division
(Ibiki/Anko), R&D = Scientific ninja tools, Finance = Village advisors,
Support = Medical corps.

## Machines (NOAD Core - 4 VMs)

| Name            | Role                                                             | Vagrant box                                     |
|-----------------|----------------------------------------------------------------------|------------------------------------------------|
| hokage-dc01     | Domain controller (Tier 0), DNS                                       | gusztavvargadr/windows-server-2022-standard     |
| anbu-srv01      | IT server (IIS + file share), unconstrained Kerberos delegation        | gusztavvargadr/windows-server-2022-standard     |
| mission-srv01   | Business server (simulated SQL/backup/applications)                    | gusztavvargadr/windows-server-2022-standard     |
| academy-ws01    | Client workstation, attacker entry point                                | gusztavvargadr/windows-10                        |

## OU organization

```
NOAD.LOCAL
├── OU=Village
│   ├── OU=Administration   OU=Academy   OU=ANBU   OU=Medical
│   └── OU=Mission          OU=Finance   OU=Research   OU=IT
├── OU=Servers
├── OU=Workstations
├── OU=Service Accounts
└── OU=Privileged
```

The local `vagrant` account (provided by the `gusztavvargadr/*` boxes)
serves as a technical thread throughout: it already exists on every VM
before the domain join even happens, and automatically becomes Domain
Admin once hokage-dc01 becomes the first domain controller of the forest.
It's the account used by Ansible end to end (see
`ansible/inventory/hosts.yml`), not a character from the universe.

## "Trap" groups

`GG_LegacyAdmins`, `GG_TempAdmins`, `GG_ProjectPhoenix` and `GG_Archive`
exist to produce realistic BloodHound noise (not every path leads
somewhere) while hiding a real weakness in some of them (a forgotten
nesting into Domain Admins for `GG_LegacyAdmins`, a forgotten local
administrator on `mission-srv01` for `GG_TempAdmins`). See `docs/NOAD.md`
for the full detail.

## Planned extensions (not built)

- **Sunagakure** (`suna.local`): second domain, trust relationship with
  Konoha -> cross-domain trust abuse (SID history, etc.).
- **Kiri**: hardened environment (restrictive GPOs, tiered administration).
- **Kumo**: vulnerable ADCS/PKI infrastructure (ESC1/ESC4/ESC8).
- **Akatsuki**: isolated, off-domain network hosting the attacker
  infrastructure (C2, phishing relay) rather than a target.
- **Orochimaru Labs**: advanced experimental techniques.

The `microsoft.ad.domain_trust` Ansible module already exists in the
installed collection and will allow implementing Suna without any
tooling change.
