## 1. What the product depends on today (measured)

| dependency | size of the tie | where |
|---|---|---|
| OpenSSL libcrypto | **103 imported functions** | `crypto-seal` (59), `snmp-crypto` (23), `crypto-aead` (10), `tls-cert_reader` (12), licence, TOTP, KDF |
| OpenSSL libssl | 15 | `tls.adb` (the console's TLS) |
| SQLite | 18 | `db-sqlite` |
| libargon2 | 2 | `crypto-argon2` (D27: **absent from RHEL's repositories**; `ARGON2=openssl` switches to OpenSSL's KDF) |
| zlib | 4 | `compress.adb` (backups, D81) |
| libc / system calls | ~43 | netlink, sockets, epoll, ioctl, prctl, memfd, Landlock ... |
| spawned programs | `ssh-keygen`, `scp`, `ssh` (NETCONF, row 78b), `ssh-keyscan`, `systemctl`, `openssl` (self-signed certificate; `s_client` as gNMI's TLS carrier, D44), `/usr/sbin/sendmail` (not declared in `tools/spawned-programs.py`) | every entry says "no row owns its removal" |
| web console | first-party, except the Roboto font (OFL) | |
| cryptography | every primitive comes from OpenSSL except Argon2id (`docs/export-control.md:42-68`) | |

No document plans to replace any of these. Each one is a place where this
list can remove code we must trust without having proved it.

## 2. Where the roadmap is going (from the survey)

- **Civil, now** (1.0 by row 278): install image and updates (225, 226,
  258), SBOM and its vulnerability judgement (229), the OUI registry (233),
  the claim code (276), self-repair (273), progress (274), console rows.
- **Enterprise (level 2)**: PostgreSQL behind a storage abstraction (46,
  D9), active/standby HA (87), FIDO2/WebAuthn (47, D7), SSO SAML/OIDC/LDAP,
  TPM 2.0 key sealing, Syslog-TLS to a SIEM with NTS timestamps (D10),
  integrations (SIEM, GLPI, Zabbix, Prometheus, Netbox, ServiceNow), the
  remote encrypted backup (50), the CRA's essential requirements and CE
  marking (53, December 2027), the Remote Controller (213), the Radio
  Probe device (129-135).
- **Military (level 3)**: non-exportable PKCS#11 HSM, local and network
  (D8), FIPS 140-2/3 or CC; mTLS with X.509 on a smart card, no password
  fallback; a named algorithm profile (RGS ANSSI, CNSA, NATO) with
  OCSP/CRL mandatory; a sealed audit trail (hash chain or HSM signature),
  WORM optional; a validated air-gap; CC/EUCC, ANSSI qualification, AQAP
  2210; a native GtkAda client (D15).
- **Signing keys in hardware**: Nitrokey HSM 2 (259, D79).
- **Not in the roadmap at all** (the survey's grep): post-quantum
  cryptography, cryptographic agility. Both matter for a level that names
  CNSA and the RGS (§4, item 17).

---

## 3. The ranked list

Ranked by what each gives the product: work taken off the team, attack
surface removed, and whether it serves a level the product will actually
ship. **Kind**: *adopt* (exists, use it), *extend* (exists, needs proof or
glue), *create* (a gap, measured), *system* (runs beside the product on
the appliance's OS).

| # | project | kind | what it gives | rows / decisions served | dependency removed |
|---|---|---|---|---|---|
| 1 | **Device fingerprint databases**: Rapid7 *recog* (banners), *p0f* signatures (TCP passive OS), *Fingerbank* (DHCP), **JA4** (TLS ClientHello; the vault holds `ja4`) **[licences to verify; JA4+ beyond JA4 has its own licence]** | adopt (data) | Device identification is the product's core, and building fingerprint sets alone is endless. The readers already exist (Wire.DHCP, Wire.TLS, Wire.TCP, SSH_Ident); these datasets turn their output into "HP LaserJet, firmware 3.x". **Nmap's databases are under the NPSL [to verify], probably unusable in a proprietary product**: checked before anything else. | inventory quality everywhere; 233's sibling | — |
| 2 | **SPARK cryptography for everything that needs no FIPS certificate**: `sparknacl` (X25519, Ed25519, XSalsa20-Poly1305), `hmac_ada` and `hkdf_ada` (SPARK-proved), `sha2`, `blake2b`/`blake3` (SPARK), `libkeccak` | extend | The seal (59 OpenSSL imports), the licence check (Ed25519), TOTP (HMAC), HKDF: internal constructions that no outside party verifies against a certificate. Moving them to proved Ada removes most of the 103 libcrypto imports and puts the product's own cryptography inside its proof. **Decision first**: the export classification, the crypto inventory, and which levels keep OpenSSL's FIPS provider (item 16). | D27, D42, D81, the seal, licence, TOTP | OpenSSL libcrypto, mostly |
| 3 | **A SPARK Argon2id** on the `blake2b` crate | create (gap: no Argon2 crate, measured) | D27 says libargon2 is absent from RHEL's repositories: today it is a build dependency carried by hand, or OpenSSL's KDF. Argon2 is BLAKE2b plus a memory-hard fill, a natural SPARK target. Publishable. | D27, the console's passwords, backup key derivation | libargon2 |
| 4 | **PostgreSQL through an existing layer**: `gnatcoll_sql` + `gnatcoll_postgres` (one query API over SQLite and PostgreSQL), or `ado_postgresql`/`ado_sqlite`; `libpq` | adopt | Row 46 asks for a storage abstraction so that SQLite becomes PostgreSQL at level 2. GNATCOLL.SQL is that abstraction already, with typed queries. The cost is moving 18 SQLite imports and their SQL; the gain is not writing a second backend. | 46, D9, 87 | the hand-written SQLite binding |
| 5 | **HA for PostgreSQL and the appliance**: Patroni or pg_auto_failover, keepalived for the virtual address **[to verify, licences]** | system | Row 87 asks for an active/standby pair with a single write holder. That is exactly what these do, proven in production; writing a failover protocol in the product would be months, and a source of split-brain defects. | 87, D9 | — |
| 6 | **WebAuthn verifier** on `cbor_ada` (SPARK) and `cose_ada` (RFC 9052/9053) | create, small | FIDO2 (row 47, D7) needs: CBOR attestation parsing, COSE keys, signature check, counters. The two hard formats exist as crates; the glue is a few hundred lines plus test vectors. Publishable; the Ada ecosystem has none. | 47, D7, the military smart-card path later | — |
| 7 | **SSO**: the `security` crate (OAuth2/OpenID, the vault screened `ada-security`), `jwt`; Keycloak as the test identity provider **[to verify]** | adopt | OIDC covers most enterprise SSO. SAML has no Ada implementation (none in the index): OIDC first, SAML through the customer's IdP bridge. LDAP has no Ada crate: a thin binding to libldap, or LDAP through the IdP. | level 2 auth | — |
| 8 | **PKCS#11 and smart cards**: OpenSC and p11-kit on the OS, SoftHSM2 for tests **[to verify]**; an Ada PKCS#11 binding | create (binding) + system | The military level (D8: non-exportable keys in an HSM; mTLS with a card) and row 259 (Nitrokey HSM 2, which OpenSC drives **[to verify]**) both go through PKCS#11. One typed binding serves the signing machine, the appliance's keys and smart-card login. SoftHSM2 makes it testable in CI with no hardware. | 259, D8, D79, level 3 auth | — |
| 9 | **TPM 2.0 sealing**: tpm2-tss (the vault holds a `tpm` corpus), Keylime for attestation **[to verify]** | extend (binding) + system | Level 2 seals the appliance's keys to its TPM. A binding to tpm2-tss's ESAPI, or tpm2-tools from a root helper (the pattern of D86), avoids writing TPM command marshalling. | level 2 keys | key files on disk |
| 10 | **Update framework references**: TUF and Uptane **[to verify]** | adopt (design) | Rows 226 and 258 already carry their own design (D61, D79, D86). TUF's threat model (rollback, freeze, mix-and-match, key compromise) is the checklist to hold them against before 1.0, and the vocabulary an auditor expects. No code adopted, a review. | 226, 258, the CRA (53) | — |
| 11 | **SBOM and vulnerability cross-checks in CI**: Syft, Grype, OSV-Scanner, the CycloneDX tools **[to verify]** | adopt (CI) | Row 229 builds the appliance's SBOM and judges it. Running an independent tool on the same image in CI is a differential test of our own judgement, at no cost to the product, and evidence for the CRA file. | 229, 53 | — |
| 12 | **Harrow**: coverage-guided fuzzing for Ada (`harrow-fuzzing-project.md`) | create | Proof covers no run-time error, not correctness; every parser defect the pipeline has met was the second kind. Serves every reader the product has and will have. | every Wire row, 53 (secure development evidence) | — |
| 13 | **Deflate in Ada**: `z_compression` (standalone Deflate) or `zipada`'s codecs (the vault holds `zip-ada`) | adopt | zlib is 4 functions behind the backup (D81). The archive is authenticated before it is decompressed, so the risk is low, but a C library less is a C library less, and zipada also brings LZMA and BZip2 if a better ratio is wanted. | D81, 50 | zlib |
| 14 | **The spawned OpenSSH tools**: `ssh_protocol` (SSH core, RFC 4251) **[maturity to read]** | extend | Four spawned programs (`ssh-keygen`, `scp`, `ssh`, `ssh-keyscan`) carry the backup deposit (264c, 227d) and NETCONF (78b). In-process SSH would give typed errors, progress (274) and no argument parsing across a process boundary, if the crate is mature. Otherwise they stay: OpenSSH is the most reviewed code here. | 264c, 227d, 50, 78b, 274 | 4 spawned programs |
| 15 | **The remaining `openssl` CLI spawns, through the library we already link** | (no new project) | The self-signed certificate (Keystore) and gNMI's TLS carrier (`s_client`, D44) can use libcrypto's X.509 builder and libssl, both already linked: a spawned process and its text parsing gone. Listed because it is a dependency reduction found by this survey. | D44, 147 | `openssl` CLI |
| 16 | **OpenSSL's FIPS provider** as the validated path | system | The military level asks for FIPS 140-2/3 or CC. RHEL's OpenSSL FIPS provider is the realistic validated module **[to verify the RHEL 10 certificate status]**. Item 2 moves the product's own constructions to SPARK; this keeps OpenSSL as the certified path for TLS and for the levels that require a certificate. The two together are the crypto-agility answer: a profile picks the provider. | level 3 algorithms, D8 | — |
| 17 | **Post-quantum**: the ANSSI transition guide (in the vault: `anssi-guides/Transition-post-quantique-protocole-TLS-1-3`), OpenSSL's ML-KEM/ML-DSA **[to verify the version]**, `adare_dilithium5` / `adare_hqc256` (Ada bindings, measured) | adopt (later) | **Absent from the roadmap.** A level that names CNSA and the RGS will be asked for hybrid key exchange in TLS and PQ signatures for updates and licences. Not urgent for 1.0; worth a §6 decision so that formats (the bundle's signature, the licence) leave room for a second algorithm now. | level 3; 226/258 formats; licences | — |
| 18 | **Syslog-TLS and NTS on the OS**: rsyslog (or syslog-ng) forwarding the journal over TLS, chrony with NTS (chrony is already in the image) **[NTS support to verify]**; CEF formatting (the vault holds `cef`) | system | Level 2's SIEM forwarding and D10's trusted time are OS services. The product writes structured lines; the OS forwards them. Nothing to write in Ada. | level 2 audit, D10 | — |
| 19 | **Integrations**: `prometheus` (client library, measured), Netbox, GLPI and Zabbix through their REST APIs | adopt + small glue | Level 2's integration list. The Prometheus exposition is a crate away; the others are REST clients. An HTTP client in Ada is needed (AWS, or libcurl through `utilada_curl`); the product writes its own HTTP server today. | level 2 integrations | — |
| 20 | **RecordFlux**: generates SPARK parsers and their proofs from message specifications **[existence, licence and maturity to verify; not in the Alire index]** | adopt (tool) | Every new protocol (the Radio Probe's 8443 session, LLDP-MED, mDNS, MQTT ...) is a hand-written proved reader today. A generator from a specification would turn weeks into days, and make each reader's correctness a property of its specification. To evaluate on one existing reader before anything else. | 129a, every future Wire row | — |
| 21 | **Proof and coding-standard tooling for certification**: `spat` (SPARK proof analysis, measured), `libadalang` and `adalang_analyzer` (measured), AdaControl **[to verify]** | adopt | CC/EUCC and the ANSSI qualification want evidence: proof statistics, coding rules enforced. These turn the existing proof into reports an evaluator reads. | 53, level 3 compliance | — |
| 22 | **Vellum**: proved X.509 and DER reader (`vellum-x509-project.md`) | create | Today only the console's own certificate is read. Becomes valuable with a device certificate inventory (a §7 decision) and with the military level's mTLS cards, OCSP and CRLs. Thumper's DER code (the vault holds `code-thumper`) is a reference. | level 3 mTLS, a possible inventory row | 12 libcrypto imports (Cert_Reader) |
| 23 | **Ordinal**: version ordering and OSV matching (`ordinal-versions-project.md`) | create/extract | Row 229b's matching rests on it. The product already has the comparators. Publishing them adds outside review and differential testing. | 229 | — |
| 24 | **linux-sys**: typed confinement and network bindings (`linux-sys-project.md`) | create/extract | Takes Landlock, seccomp, AF_PACKET and netlink constants out of the product's review load, with a CI check against kernel headers. Valuable, but the product's current bindings work. | the confinement rows, the military "full Landlock" | hand-written constants |
| 25 | **spark-wire**: the readers published (`spark-wire-project.md`) | publish | Outside review and contributed protocols. Waits on the licence and export decisions. | every Wire row | — |
| 26 | **GtkAda** for the level-3 native client (D15), measured in the index | adopt (later) | Exists; no work until level 3. | D15 | — |

---

## 4. What to do first

The ones that need no decision and pay soonest:

1. **Check the fingerprint datasets' licences (item 1)**: a day of reading
   for the largest gain in what the operator sees.
2. **Run Syft, Grype and OSV-Scanner on the image in CI (item 11)** as a
   differential of row 229, before 1.0.
3. **Review rows 226 and 258 against TUF's threat model (item 10)** before
   the beta of row 278.
4. **Replace the `openssl` CLI spawns with the linked library (item 15)**:
   a normal row, no new dependency.

The ones that need the operator first:

5. **SPARK cryptography for internal constructions (items 2, 3, 16)**: a
   §6 decision on which constructions leave OpenSSL, what the export
   classification and the crypto inventory then say, and which levels keep
   the FIPS provider. Largest attack-surface reduction in the list.
6. **The storage layer for level 2 (items 4, 5)**: GNATCOLL.SQL over both
   engines, and HA by an existing manager rather than our own protocol.
7. **Post-quantum (item 17)**: a §6 decision now, so that the signature
   fields of bundles and licences can carry a second algorithm later.

The ones that wait for their level: 6, 7, 8, 9 (Enterprise), 16 to 22
(military).
