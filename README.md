# Stenopos package registry

The signed index that `tagma add` and `tagma publish` use (ADR-0039). It
records where each package version's source lives and a hash of its files.
It does not host any code.

| Path | What it is |
|------|------------|
| `root.pub` | The registry root key. It is kept offline and only signs key statements. tagma has it built in |
| `allowed_signers` | `root.pub` in the form `ssh-keygen -Y verify` reads |
| `keys/<name>.toml` + `.sig` | A key the root approves, with its role and expiry, signed by the root |
| `index/<namespace>/<name>.toml` | A package: its owners and one entry per version |
| `snapshot.toml` + `.sig` | The hash of every index file, with an expiry, signed daily by the online key |

## Verify a key statement

```bash
ssh-keygen -Y verify -f allowed_signers -I registry-root -n stenopos-registry \
    -s keys/online.toml.sig < keys/online.toml
```

## Publishing

Packages are published by CI when a Stenopos-org package repository is
tagged; entries are never edited by hand. Open publishing comes later.

This repository is developed on the Stenopos Gitea server; the GitHub copy is
published automatically after every build and test passes.

*Ἀσφάλεια περὶ πλείστου: security above all.*
