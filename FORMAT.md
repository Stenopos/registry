# The registry format

Version 1. This is the exact format of this repository, which tagma reads and
`tagma registry check` validates. The design and its reasons are ADR-0039
(`adr/0039-package-registry.md` in the Stenopos design repository); this
document is the contract. Every file is TOML 1.0 in UTF-8, and only the
subset `std.toml` reads: no multi-line strings, floats or dates (dates and
times are strings).

## Layout

```
root.pub                        the root key (one ssh-ed25519 line)
allowed_signers                 root.pub as `ssh-keygen -Y verify` reads it
keys/<key>.toml                 a key statement
keys/<key>.toml.sig             the root's signature over it
keys/<key>.pub                  the key itself, for ssh-keygen
index/<namespace>/<name>.toml   a package
snapshot.toml                   every file's hash, an expiry and a counter
snapshot.toml.sig               the online key's signature over it
```

Anything else at the top level is limited to `README.md`, `FORMAT.md`,
`SECURITY.md`, the licences and CI files (`.drone.yml`, `.gitignore`, and
`syntagma.lock`, the Syntagma commit whose tagma CI runs); in a git checkout
only tracked files count; the checker rejects other files in
`keys/` and `index/`, except `index/.keep`, which keeps an empty index in git.

## Names

- A **namespace** and a package **name** are 1–64 characters of lowercase
  ASCII letters, digits and `-`, starting with a letter and not ending with
  `-`. A package is written `<namespace>/<name>` (`stenopos/json`).
- A **key name** follows the same rule (`online`, `stenopos-ci`).
- A **version** is `MAJOR.MINOR.PATCH`, each a decimal number without
  leading zeros (`1.10.0`). Pre-release and build suffixes are not accepted in
  version 1 of the format. Versions compare numerically, field by field.
- A **version requirement** (in dependencies) is `MAJOR`, `MAJOR.MINOR` or
  `MAJOR.MINOR.PATCH` and means "at least that, within that major version"
  (`"1.2"` is `>= 1.2.0, < 2.0.0`). The major version is part of a package's
  identity: `json` 1.x and 2.x are resolved as different packages.
- A **capability** is one of `File`, `Network`, `Process`, `Memory`,
  `System`. A list of them is sorted and has no repeats.
- A **hash** is `sha256:` and 64 lowercase hex digits.
- A **commit** is 40 lowercase hex digits (a git SHA-1 commit id).
- A **date** is a string `YYYY-MM-DD`; a **time** is a string
  `YYYY-MM-DDTHH:MM:SSZ` (UTC).

## Key statements: `keys/<key>.toml`

```toml
role = "publisher"
purpose = "Signs version entries published by Stenopos CI"
key = "ssh-ed25519 AAAA... Stenopos CI"
created = "2026-11-05"
expires = "2027-11-05"
namespaces = ["stenopos"]
```

| Field | Rule |
|---|---|
| `role` | `"online"` (signs the snapshot) or `"publisher"` (signs version entries) |
| `purpose` | Free text |
| `key` | One `ssh-ed25519` public key line; the same line is `keys/<key>.pub` |
| `created`, `expires` | Dates; the key is valid on days `created` ≤ d < `expires` |
| `namespaces` | Publishers only: the namespaces it may publish to |
| `revoked` | Optional: a date, not before `created`, from which the key is compromised (below) |

`keys/<key>.toml.sig` is an armored SSH signature (`ssh-keygen -Y sign`) by
the root key over the file's exact bytes, namespace `stenopos-registry`. The
root key signs nothing else. There is exactly one `online` key that is not
revoked at a time.

### Revoking a key

When a key is compromised, the root re-signs its statement with
`revoked = "<date>"`: the date from which it must not be trusted (it may be
in the past). Then:

- A version the key signed with `published` on or after that date is
  **revoked**. Its entry stays (the index is append-only), but tagma never
  uses it: not for a new resolution, and, unlike a retracted version, not for a
  lock that names it either. A build that has a revoked version locked
  fails once its copy of the index is updated, naming the version.
- A retraction the key signed with `retracted_on` on or after that date is void.
- The key can't sign anything new: the registry's CI rejects a new entry or
  retraction signed by it.
- A revocation can't be removed or moved later.
- A revoked `online` key is replaced by a new one under another name; the
  snapshot is signed by the one that is not revoked.

## Packages: `index/<namespace>/<name>.toml`

```toml
name = "stenopos/json"
repository = "https://github.com/Stenopos/json"

[[version]]
version = "1.2.0"
edition = "2026"
commit = "0123456789abcdef0123456789abcdef01234567"
hash = "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
caps = ["File"]
published = "2026-11-05"
publisher = "stenopos-ci"
signature = "U1NIU0lH..."

[version.dependencies]
"stenopos/text" = "1.0"
```

| Field | Rule |
|---|---|
| `name` | The package; must match the file's path |
| `repository` | An `https://` URL of the package's git repository, made only of letters, digits and `- . _ /` (no `..`), at most 200 characters. A GitHub URL `https://github.com/<owner>/<repo>` has a mirror at `https://gitea.tuxsmash.com/<owner>/<repo>`, which tagma tries when GitHub fails |
| `version` (array) | One table per published version, in increasing version order, no repeats |
| `version.version` | The version |
| `version.edition` | The language edition the package declares (`Tagma.toml`) |
| `version.commit` | The commit the version was published from |
| `version.hash` | The content hash of the package's files at that commit (below) |
| `version.caps` | The capabilities its code uses: the union of its `@cap` annotations |
| `version.dependencies` | Optional table: package → version requirement; a package can't depend on itself |
| `version.published` | The date it was published |
| `version.publisher` | The key that signed it: a `publisher` key valid on `published` whose `namespaces` include this package's |
| `version.signature` | The publisher's signature over the entry statement (below) |
| `version.retracted` | Optional; when present, `true`, with `retracted_on` and `retract_signature` |
| `version.retracted_on` | The date it was retracted |
| `version.retract_signature` | The publisher's signature over the retraction statement (below) |

A published entry never changes, except that it can be retracted once.
Versions are only ever added. The registry's CI rejects a commit that
changes or removes an entry (it compares against the previous commit), and
a new entry or retraction that is not dated today or yesterday (UTC): dates can't
be set back, so a key can't be used to slip a version in before its
revocation.

A **retracted** version is never chosen for a new resolution, but a lock file
that already names it still builds. Publishers retract with `tagma retract`, which
the package's CI runs on a signed `retract/<version>` tag.

### The entry statement

A version's `signature` covers this text, built from the entry, one field per
line, each line ending in `\n`:

```
stenopos-registry-entry 1
package stenopos/json
repository https://github.com/Stenopos/json
version 1.2.0
edition 2026
commit 0123456789abcdef0123456789abcdef01234567
hash sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
caps File
dependency stenopos/text 1.0
published 2026-11-05
```

`caps` lists the capabilities separated by `,` (the line is `caps ` with
nothing after it when there are none). There is one `dependency` line per
dependency, sorted by package name. The signature is an SSH signature with
namespace `stenopos-registry-entry`, stored as the base64 of its binary form
(the armored text's body, joined into one line).

### The retraction statement

```
stenopos-registry-retract 1
package stenopos/json
version 1.2.0
retracted_on 2026-12-01
```

Signed the same way, with namespace `stenopos-registry-retract`, by a publisher
key for the package's namespace valid on `retracted_on`.

## The content hash

The hash of a package version is computed from the files in its repository at
the recorded commit (the files `git archive` would include):

1. For each file, take its path relative to the repository root, with `/`
   separators. A path containing a line break is an error. Symbolic links and
   empty directories are not part of a package and are an error.
2. Write one line per file: the lowercase hex SHA-256 of the file's bytes,
   two spaces, the path, `\n`.
3. Sort the lines by path, comparing bytes.
4. The hash is `sha256:` followed by the hex SHA-256 of the concatenated
   lines.

This is the scheme of Go's `dirhash` (`h1`), so it doesn't depend on how the
archive was compressed or ordered.

## The snapshot: `snapshot.toml`

```toml
counter = 42
created = "2026-11-05T03:00:00Z"
expires = "2026-11-12T03:00:00Z"

[files]
"keys/online.toml" = "sha256:..."
"index/stenopos/json.toml" = "sha256:..."
```

| Field | Rule |
|---|---|
| `counter` | Increases by exactly 1 each time the snapshot is signed |
| `created`, `expires` | Times; `expires` is 7 days after `created` |
| `files` | The SHA-256 of every file under `keys/` and `index/`, and of `root.pub` |

`snapshot.toml.sig` is an armored SSH signature over the file's exact bytes
by the `online` key, namespace `stenopos-registry-snapshot`. CI signs a new
snapshot after every change to the index and at least once a day.

## What a client checks

Before using the index, tagma:

1. verifies each key statement against the root key built into tagma (and
   that `root.pub` is that key);
2. verifies the snapshot's signature with the online key, that the online key
   was valid when the snapshot was created, that the snapshot has not
   expired, and that `counter` is not lower than the last one it saw (so a
   replayed, older index is refused);
3. checks every file's hash against the snapshot;
4. verifies each version entry it uses: its signature, its publisher's key and
   namespaces, and then, after fetching the source, the content hash.

A failure stops the command and names the file and the check that failed.

## What the checker checks

`tagma registry check [dir]` runs every rule in this document over a checkout,
including all the signatures, and with `--previous <dir>` also checks that
the change from that checkout only adds versions and retractions. The registry's
CI runs it on every push.
