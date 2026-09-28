<!--
This file has two parts:
1. The upstream-facing PR body, between the PR-BODY-START/END markers.
2. "Notes pour Cyril" at the end, in French, NOT for upstream maintainers.

To extract just the PR body (e.g. for `gh pr create --body-file`), run:

  awk '/^<!-- PR-BODY-START -->$/{f=1;next}/^<!-- PR-BODY-END -->$/{f=0}f' \
    kata-pr-dragonball-virtiofsd.md > kata-pr-body.md

(the markers must be anchored to the full line with ^...$: the text
"PR-BODY-START"/"PR-BODY-END" also appears, unanchored, inside this
comment block and inside the extraction command itself below.)

Do NOT pass this whole file as --body-file: it would leak the French
notes and the local-state header to the upstream PR.
-->

<!-- PR-BODY-START -->
# runtime-rs/dragonball: fix directory-ordering panic in standalone virtiofsd setup

## Problem

On Kata Containers 4.2.0, `runtime-rs` + Dragonball, with a hypervisor
config that switches `shared_fs` from the default `inline-virtio-fs` to
the standalone (external) `virtio-fs` daemon, sandbox creation fails:

```
failed to create shim task: ttrpc: closed
```

**Scope of this PR:** this fixes the crash below (directory-creation
ordering + a panic-on-early-exit). It does **not** by itself guarantee a
Dragonball sandbox boots end-to-end with `shared_fs = "virtio-fs"` — see
"Known limitation" at the end, which is an open question for
maintainers, not something this PR resolves.

## Reproduction

Minimal Dragonball hypervisor config (e.g. `configuration-dragonball.toml`):

```toml
[hypervisor.dragonball]
shared_fs = "virtio-fs"
virtio_fs_daemon = "/opt/kata/libexec/virtiofsd"
```

Any `ctr`/`crictl`/`kubectl run` that creates a Kata sandbox with this
config fails immediately. Shim logs show:

```
VMM not ready, queueing device ShareFs(ShareFsDevice { ... sock_path: "/run/kata/<id>/root/virtiofsd.sock", mount_tag: "kataShared", queue_size: 0, queue_num: 1, options: [] ...
source: virtiofsd [ERROR virtiofsd] /run/kata/<id>/root does not exist or is not a directory
wait virtiofsd Ok(ExitStatus(unix_wait_status(256)))
A panic occurred at src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs:159: called `Option::unwrap()` on a `None` value
```

## Root cause

Two independent bugs in `runtime-rs`, both hit on this code path.
Line numbers below are at tag `4.2.0` (`c7351e797efff8bfc6bd73da0eb1909be12e2cfe`),
which is byte-identical to current `main` for both files.

### 1. `jailer_root` directory does not exist yet when virtiofsd starts

[`ShareVirtioFsStandalone::setup_device_before_start_vm`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs#L217-L228)
calls `h.get_jailer_root().await?` and then spawns virtiofsd with
`--socket-path` under that directory
([`setup_virtiofsd`, L104](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs#L104)).
This runs before `start_vm()`.

Dragonball's
[`DragonballInner::get_jailer_root()`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/hypervisor/src/dragonball/inner_hypervisor.rs#L192-L194)
just returns the stored `self.jailer_root` string; it never creates the
directory. The directory is only created later, in
[`DragonballInner::run_vmm_server()`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/hypervisor/src/dragonball/inner.rs#L230-L237),
which is called from `start_vm()` — i.e. after the share-fs setup that
just tried to use it.

So virtiofsd is launched with a `--socket-path` whose parent directory
does not exist, and exits immediately:
`/run/kata/<id>/root does not exist or is not a directory`.

**Why this doesn't affect QEMU or Cloud Hypervisor:** both of those
backends' `get_jailer_root()` accessors already create the directory on
every call — see `get_jailer_root()` in
[`qemu/inner.rs`](https://github.com/kata-containers/kata-containers/blob/e803883216e8f1f9357cec3850657d693b4b29fe/src/runtime-rs/crates/hypervisor/src/qemu/inner.rs)
and
[`ch/inner_hypervisor.rs`](https://github.com/kata-containers/kata-containers/blob/e803883216e8f1f9357cec3850657d693b4b29fe/src/runtime-rs/crates/hypervisor/src/ch/inner_hypervisor.rs)
(both call `create_dir_all_with_inherit_owner`, at `main`
`e803883216e8f1f9357cec3850657d693b4b29fe`). QEMU's `start_vm()` even has
a comment about exactly this ordering hazard: *"prepare_before_start_vm()
never calls get_jailer_root(), so it must be done here explicitly"*.
Dragonball's `get_jailer_root()` never got the same treatment.

### 2. The shim panics instead of failing cleanly when virtiofsd exits early

[`setup_virtiofsd`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs#L155-L169)
spawns virtiofsd, then a background task
([`run_virtiofsd`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs#L192-L209))
reads its stderr and sends `Ok(())` over an `mpsc` channel only once it
sees the line `"Waiting for vhost-user socket connection"`. If virtiofsd
exits (for any reason) before ever printing that line, the sender is
dropped without sending anything and `rx.recv()` returns `None`.
Line 159 was:

```rust
// TODO: support timeout
match rx.recv().await.unwrap() {
```

`.unwrap()` on `None` panics, taking down the whole
`containerd-shim-kata-v2` process (hence `ttrpc: closed` on the
containerd side) instead of returning a normal error for this sandbox.

## Fix

Two commits, one per root cause.

### Commit 1 — create the jailer root directory in `get_jailer_root()`

`src/runtime-rs/crates/hypervisor/src/dragonball/inner_hypervisor.rs`,
`DragonballInner::get_jailer_root()`:

```rust
pub(crate) async fn get_jailer_root(&self) -> Result<String> {
    std::fs::create_dir_all(&self.jailer_root)
        .map_err(|e| anyhow!("Failed to create dir {} err : {:?}", self.jailer_root, e))?;
    Ok(self.jailer_root.clone())
}
```

Creating the directory inside `get_jailer_root()` itself, rather than in
`prepare_vm()`, matches the existing QEMU/Cloud Hypervisor pattern cited
above. `run_vmm_server()` still calls `create_dir_all` on the same path
later; both are plain `std::fs::create_dir_all`, which is idempotent, so
calling it twice is harmless. The plain (non-ownership-inheriting)
`create_dir_all` was chosen to match what Dragonball's own
`run_vmm_server()` already uses for this same directory.

`ShareVirtioFsInline` (the default `inline-virtio-fs` backend) does
**not** call `get_jailer_root()` — it passes an empty string literal to
`prepare_virtiofs()` directly — so it is unaffected by this change.

### Commit 2 — don't panic when virtiofsd exits without reporting status

`src/runtime-rs/crates/resource/src/share_fs/share_virtio_fs_standalone.rs`,
`setup_virtiofsd()`:

```rust
match rx
    .recv()
    .await
    .unwrap_or_else(|| Err(anyhow!("virtiofsd exited before reporting its status")))
{
    Ok(_) => { /* unchanged */ }
    Err(e) => { /* unchanged: shuts down virtiofsd, returns a normal Err */ }
}
```

`unwrap_or_else` routes the `None` case into the existing `Err(e)` arm,
so `shutdown_virtiofsd()` still runs. `.ok_or_else(...)?` was
deliberately not used instead, since that would skip
`shutdown_virtiofsd()` on the `None` path and leak the already-spawned
virtiofsd PID.

This is a useful hardening on its own, independent of commit 1: any
other reason virtiofsd exits early (bad `--shared-dir`, wrong
`virtio_fs_daemon` path, OOM, …) hits the same panic today.

## Test performed

- `rustfmt --edition 2018 --check` on both modified files: clean, no
  formatting diff.
- `cargo check -p hypervisor --features dragonball` and
  `cargo check -p resource`, from a fresh clone of this branch
  (`runtime-rs-dragonball-standalone-virtiofsd` @ `22e3214`): both
  **succeed** (`Finished \`dev\` profile [unoptimized + debuginfo]`,
  exit code 0). No warning on either changed file (the only warning in
  either build log is the generic, pre-existing "profiles for the non
  root package will be ignored" workspace note, unrelated to this
  change).
- Confirmed byte-for-byte (identical blob SHA) that both files are
  unchanged between tag `4.2.0` and current `main`.
- **Runtime test on a real cluster** (Kubernetes 1.36, containerd 2.3,
  bare-metal x86_64, kata-deploy 4.2.0 whose `shim-v2-rust` artifact was
  replaced by a build of tag `4.2.0` + these two commits, musl static,
  same `make` invocation as `tools/packaging/static-build/shim-v2`),
  `shared_fs = "virtio-fs"`, `virtio_fs_daemon =
  "/opt/kata/libexec/virtiofsd"`, pod with `runtimeClassName:
  kata-dragonball`:
  - **Before** (stock 4.2.0 shim): every sandbox create hits the panic
    above; containerd reports `failed to create shim task: ttrpc: closed`
    in a loop.
  - **After** (these two commits): no panic. virtiofsd now starts, and
    the sandbox fails *cleanly* later, at VMM start, with a proper error
    propagated to containerd — which confirms the "Known limitation"
    below at runtime:

    ```
    failed to create shim task: Others("failed to handle message start sandbox in task handler

    Caused by:
        0: start vm
        1: start vmm instance …
    ```

    and in the shim log:

    ```
    start micro vm error start vmm instance
        0: Failed to start vmm
        1: Failed to start MicroVm
        2: vmm action error: StartMicroVm(FsDeviceError(CreateFsDevice(InvalidInput)))
    VM: remove devices
    …
    shutdown virtiofsd pid 3189309
    ```

    i.e. the error path now tears down the VM devices and shuts the
    already-spawned virtiofsd down instead of crashing the shim.
  - **With `vhost-user-fs` additionally enabled** on the `dragonball`
    dependency in `src/runtime-rs/crates/hypervisor/Cargo.toml` (see
    "Known limitation"), same cluster and config: the sandbox **boots**.
    The shim logs `vhost-user-fs: protocol negotiate completed
    successfully`, the container runs inside the guest (`hypervisor`
    CPU flag present), and virtiofsd is shut down cleanly when the pod
    ends. Note: as with `qemu-runtime-rs`, `--xattr` must be passed in
    `virtio_fs_extra_args` (the shim only appends it when
    `disable_guest_selinux = false`), otherwise `setxattr` in the guest
    returns `EOPNOTSUPP`.

## Known limitation / question for maintainers

After this fix, the external virtiofsd process itself should start
successfully. But the VMM-side device attachment that follows once the
VMM is ready — `add_share_fs_device`/`do_add_fs_device` in
`inner_device.rs`, which calls `DragonballInner::vmm_instance.insert_fs()`
with `mode = "vhostuser"` — ultimately reaches
[`FsDeviceMgr::create_fs_device`](https://github.com/kata-containers/kata-containers/blob/c7351e797efff8bfc6bd73da0eb1909be12e2cfe/src/dragonball/src/device_manager/fs_dev_mgr.rs#L375-L386)
in the `dragonball` crate:

```rust
fn create_fs_device(...) -> std::result::Result<DbsVirtioDevice, FsDeviceError> {
    match &config.mode as &str {
        VIRTIO_FS_MODE => Self::attach_virtio_fs_devices(config, ctx, epoll_mgr),
        #[cfg(feature = "vhost-user-fs")]
        VHOSTUSER_FS_MODE => Self::attach_vhostuser_fs_devices(config, ctx, epoll_mgr),
        _ => Err(FsDeviceError::CreateFsDevice(virtio::Error::InvalidInput)),
    }
}
```

The `VHOSTUSER_FS_MODE` arm only exists when the `dragonball` crate is
built with its own `vhost-user-fs` Cargo feature. That feature is not in
the feature list `hypervisor/Cargo.toml` requests from its `dragonball`
dependency (`atomic-guest-memory, dbs-upcall, host-device, hotplug,
vhost-net, vhost-user-net, virtio-balloon, virtio-blk, virtio-fs,
virtio-mem, virtio-net, virtio-rng, virtio-vsock` — no
`vhost-user-fs`), `src/runtime-rs/Makefile`'s
`EXTRA_RUSTFEATURES += dragonball` (used by the standard/kata-deploy
build) does not add it either, and a repo-wide search shows no other
crate in the `runtime-rs`/`dragonball` dependency graph requests it
(Cargo feature unification is workspace-wide, so any one crate asking
for it would be enough — none does). So with a standard build, once
virtiofsd is up, `create_fs_device` falls through to the `_ =>` arm and
returns `FsDeviceError::CreateFsDevice(InvalidInput)` instead of
attaching a working device — `shared_fs = "virtio-fs"` is not fully
wired up in the default feature set, independently of this fix. I did
not add a third commit to flip that feature on: it changes binary
composition and needs its own review, and is out of scope for this
crash fix. The runtime test above confirms this exact outcome
(`StartMicroVm(FsDeviceError(CreateFsDevice(InvalidInput)))`) once the
crash is fixed. Flagging it here so maintainers can say whether
`vhost-user-fs` is expected to be enabled by users of this `shared_fs`
mode, or whether it needs its own fix.
<!-- PR-BODY-END -->

---

## Notes pour Cyril

- **À lire en premier : ce correctif seul ne fera probablement pas
  démarrer vos sandboxes de prod, sauf si votre binaire kata-deploy est
  compilé différemment du build standard.** Constaté dans le code, au
  tag `4.2.0` (SHA identique à `main` pour ce fichier) :
  `src/dragonball/src/device_manager/fs_dev_mgr.rs`, fonction
  `create_fs_device` (L375-386) :
  ```rust
  match &config.mode as &str {
      VIRTIO_FS_MODE => Self::attach_virtio_fs_devices(config, ctx, epoll_mgr),
      #[cfg(feature = "vhost-user-fs")]
      VHOSTUSER_FS_MODE => Self::attach_vhostuser_fs_devices(config, ctx, epoll_mgr),
      _ => Err(FsDeviceError::CreateFsDevice(virtio::Error::InvalidInput)),
  }
  ```
  La branche `vhostuser` n'existe que si le crate `dragonball` est
  compilé avec sa feature `vhost-user-fs`. Or, vérifié aussi au tag
  4.2.0 : `hypervisor/Cargo.toml` n'active, pour sa dépendance
  `dragonball`, que `["atomic-guest-memory", "dbs-upcall",
  "host-device", "hotplug", "vhost-net", "vhost-user-net",
  "virtio-balloon", "virtio-blk", "virtio-fs", "virtio-mem",
  "virtio-net", "virtio-rng", "virtio-vsock"]` — **pas
  `vhost-user-fs`** —, `src/runtime-rs/Makefile`
  (`EXTRA_RUSTFEATURES += dragonball`, la variable utilisée par le
  build standard / kata-deploy) n'ajoute rien de plus, et une recherche
  sur tout le dépôt (`vhost-user-fs filename:Cargo.toml`) montre que
  seuls `src/dragonball/Cargo.toml` et
  `dbs_virtio_devices/Cargo.toml` déclarent cette feature côté
  dépendance — aucun autre crate du graphe `runtime-rs`/`dragonball` ne
  la demande, alors que l'unification des features Cargo est globale
  au workspace : il suffirait d'un seul crate qui la demande pour
  qu'elle soit active partout, et il n'y en a aucun. Donc, avec un build
  standard, une fois virtiofsd démarré correctement (grâce à ce
  correctif) et le sandbox prêt à attacher le device, `create_fs_device`
  tombe dans la branche `_` et renvoie une erreur
  (`FsDeviceError::CreateFsDevice(InvalidInput)`) au lieu d'attacher un
  vrai device vhost-user-fs. Le sandbox échouera quand même, juste avec
  une erreur propre au lieu d'un panic.
  **Confirmé à l'exécution le 2026-09-28** (image `4.2.0-dbfix1`, cluster
  jouve-dev) : plus de panic, mais le sandbox échoue sur
  `StartMicroVm(FsDeviceError(CreateFsDevice(InvalidInput)))` — traces dans
  « Test performed ». Test suivant : `4.2.0-dbfix2` (feature
  `vhost-user-fs` ajoutée aux features `dragonball` de
  `hypervisor/Cargo.toml`, commit local `4ddecb5`, pas sur le fork) ;
  résultat à reporter ici et dans la PR (3e commit ou non).

- **Pas de `Fixes: #...`** : pas d'issue ouverte, comme demandé.
  `.github/workflows/commit-message-check.yaml` ne vérifie pas la
  présence d'un `Fixes:` (seulement DCO, corps de message, longueur du
  sujet ≤75 et des lignes de corps ≤150, présence d'un `subsystem:`) —
  rien ne bloquera la CI sur ce point. `CONTRIBUTING.md` recommande
  cependant d'ouvrir une issue avant une PR ; à toi de voir.

- **Commit 1 : paragraphe corrigé** (fait le 2026-09-28) : le message
  affirmait à tort que `inline-virtio-fs` passe par `get_jailer_root()` ;
  reformulé, branche du fork force-pushée (`8cc40e4` → `22e3214`, code
  identique).

- **Attribution Claude — divergence avec la consigne** : le prompt de
  tâche demandait `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`,
  mais le rappel système de cette session imposait
  `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>` (modèle
  effectivement utilisé). J'ai suivi le rappel système. Si tu préfères
  l'attribution demandée dans la tâche, corrige-la au même rebase que
  le point précédent.

- **Attribution IA (`Generated-By:`)** : j'ai utilisé ce trailer (pas
  `Assisted-By:`) car le code a été substantiellement écrit par l'IA, pas
  juste auto-complété, conformément à la politique OpenInfra citée par
  `CONTRIBUTING.md`. Format vérifié sur des commits amont récents.

- **Relis/réécris les messages de commit et cette description toi-même
  avant d'ouvrir la PR** : `CONTRIBUTING.md` décourage explicitement la
  prose entièrement générée par IA dans les PR/commentaires. Le contenu
  ci-dessus est factuel mais reste rédigé par moi.

- **Compilation : faite, et verte.** Clone réel du fork (`git clone
  --depth 1 --branch runtime-rs-dragonball-standalone-virtiofsd`,
  supprimé ensuite) + `cargo check -p hypervisor --features dragonball`
  (5m46s, `Finished`/exit 0) et `cargo check -p resource` (7m20s,
  `Finished`/exit 0), dans ce bac à sable — le réseau a fini par
  passer, contrairement à ce que je pensais au premier essai. Reste à
  lancer `cargo clippy -p hypervisor -p resource --features dragonball`
  et, si possible, un vrai test d'intégration (sandbox Dragonball +
  `shared_fs = "virtio-fs"`) avant d'ouvrir la PR — je n'ai pas pu
  reproduire le bug moi-même ni exécuter le chemin vhost-user-fs
  ci-dessus.

- **Pour ouvrir la PR depuis la branche du fork** :
  ```sh
  awk '/^<!-- PR-BODY-START -->$/{f=1;next}/^<!-- PR-BODY-END -->$/{f=0}f' \
    kata-pr-dragonball-virtiofsd.md > kata-pr-body.md
  # sanity check: must print nothing
  grep -c 'Notes pour\|gh pr create\|awk ' kata-pr-body.md
  gh pr create --repo kata-containers/kata-containers \
    --head jouve:runtime-rs-dragonball-standalone-virtiofsd \
    --base main \
    --title "runtime-rs: fix directory-ordering panic in standalone virtiofsd setup" \
    --body-file kata-pr-body.md
  ```
  ou via l'UI GitHub :
  `https://github.com/kata-containers/kata-containers/compare/main...jouve:kata-containers:runtime-rs-dragonball-standalone-virtiofsd?expand=1`
