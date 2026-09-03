# Alma Black Box ISO Builder

Build a personal, unattended UEFI installer ISO for [Alma Black Box](https://github.com/highwaytoit/alma-black-box) with GitHub Actions.

This repository is intended to be used as a **GitHub template**. It is an ISO factory, not the Alma Black Box operating-system source. The installed operating system comes from the selected bootc container image.

> [!IMPORTANT]
> The default image is `ghcr.io/highwaytoit/alma-black-box:10`. The `:10` tag is resolved to its current amd64 digest every time you manually start a build. A repository created from this template months from now will therefore install the current image behind that tag, not the image that existed when the template was copied.

## Build your ISO

1. Click **Use this template** on this repository and create your own repository.
2. Open **Actions** in your new repository.
3. Select **Build Alma Black Box installer ISO**.
4. Click **Run workflow**.
5. Choose the installer access options described below.
6. Leave the default bootc image unchanged unless you intentionally want to install another image.
7. Wait for the workflow to finish successfully.
8. Open the completed workflow run. The **Summary** contains a **Download installer artifact** link.
9. Download `alma-black-box-installer`, extract the ZIP, and use `alma-black-box-installer.iso`.

The artifact also contains `SHA256SUMS`.

Artifacts are intentionally retained for only **1 day**. This repository is a builder, not an ISO archive. If the artifact expires, simply run the workflow again.

## Installer access options

Every ISO keeps the local administrator account:

```text
username: bbox
groups:   wheel
```

Root login remains locked.

The **Run workflow** screen provides two access toggles:

- **Enable bbox password login** — default: ON
- **Install SSH public key from SSH_PUBLIC_KEY secret** — default: OFF

This gives three supported modes:

| Password login | SSH key | Result |
| --- | --- | --- |
| ON | OFF | Default `bbox / bbox` login |
| ON | ON | `bbox / bbox` plus SSH public-key login |
| OFF | ON | SSH-key-only login for `bbox`; the password is locked |

The workflow rejects **OFF / OFF** immediately so it cannot spend hours building an ISO with no usable login method.

### Default password mode

With **Enable bbox password login** ON, the temporary credentials are:

```text
username: bbox
password: bbox
```

The password stored in the installer configuration is a one-way hash, not plaintext.

Change the password immediately after the first successful boot:

```bash
passwd
```

Do not put your real password into a public template repository or workflow file.

### SSH public-key login

Only the **public** SSH key is placed into GitHub. Your private SSH key stays on your computer and must never be committed to the repository or stored in this template.

If you already have an SSH key you want to use, copy its `.pub` contents.

To create a dedicated Ed25519 key on Linux or macOS:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/bbox_ed25519 -C "bbox"
```

This creates:

```text
~/.ssh/bbox_ed25519      private key - keep this private
~/.ssh/bbox_ed25519.pub  public key  - put this in GitHub
```

Show the public key with:

```bash
cat ~/.ssh/bbox_ed25519.pub
```

In the repository you created from this template:

1. Open **Settings**.
2. Open **Secrets and variables** -> **Actions**.
3. Click **New repository secret**.
4. Name it exactly:

   ```text
   SSH_PUBLIC_KEY
   ```

5. Paste the complete public key, for example `ssh-ed25519 AAAA... bbox`.
6. Save the secret.
7. Run the ISO workflow and enable **Install SSH public key from SSH_PUBLIC_KEY secret**.

The workflow validates the key before starting the long ISO build. If the toggle is enabled but the secret is missing or malformed, the workflow fails immediately.

The generated installer adds the key to the `bbox` account using Kickstart's native `sshkey` support.

### SSH-key-only mode

For SSH-key-only access:

- turn **Enable bbox password login** OFF;
- turn **Install SSH public key from SSH_PUBLIC_KEY secret** ON.

The `bbox` user still exists and remains in `wheel`, but its local password is locked. SSH public-key authentication remains available through the injected key.

> [!NOTE]
> A locked password also means you cannot initially sign in to Cockpit with `bbox` using a password. If you later want normal Cockpit password login, SSH into the machine first and set a local password with `sudo passwd bbox`.

## Default installation image

The default workflow input is:

```text
ghcr.io/highwaytoit/alma-black-box:10
```

The workflow resolves that moving tag to an immutable digest at build time and uses that exact digest for the installer payload.

The default Alma Black Box image is verified with the included `cosign.pub` before the ISO is built.

### About `cosign.pub`

> [!IMPORTANT]
> **Do not remove or replace `cosign.pub` when building the default Alma Black Box ISO.**
>
> The ISO Builder uses this public key to verify that `ghcr.io/highwaytoit/alma-black-box:10` was signed by the Alma Black Box project before that image is included in the installer ISO.
>
> This is a public verification key only. It cannot be used to sign images and does not expose the private signing key.
>
> If you use the default Alma Black Box image, leave `cosign.pub` unchanged.

### Building a different bootc image

The **Run workflow** dialog contains a `bootc_image` field. Replace the default value with another public bootc image if required.

Signature verification is enabled by default. If the replacement image uses a different Cosign key, replace `cosign.pub` with the correct public key before building. If the replacement image is unsigned, the workflow also provides a `verify_signature` switch; disabling verification removes this supply-chain check and should be an intentional choice.

The supplied installer environment is AlmaLinux 10 / x86_64 / UEFI oriented. A different distribution or architecture may require changes to `installer/Containerfile` and the workflow.

## DANGER: the installer is intentionally destructive

> [!CAUTION]
> **Disconnect every disk except the target installation disk before booting this ISO on physical hardware.**
>
> The supplied Kickstart uses `clearpart --all`. It is deliberately designed for a machine where the installer can see only the disk that should be erased. Do not rely on the installer to guess which of several attached disks contains data you want to keep.

For a VM, create the VM with **one blank target disk**.

For physical hardware, the safest procedure is:

1. Power the machine off.
2. Disconnect all non-target storage devices.
3. Leave only the OS target disk connected.
4. Boot the installer ISO and let the unattended installation finish.
5. Boot the installed Alma Black Box system and verify it.
6. Power down and reconnect any data disks afterward.

## Default partition layout

The partitioning and base account configuration live in:

```text
installer/iso.toml
```

The default layout is:

| Mount point | Filesystem | Size |
| --- | --- | --- |
| `/boot/efi` | EFI System Partition | 512 MiB |
| `/boot` | XFS | 1 GiB |
| `/` | XFS | minimum 8 GiB, grows to use remaining disk space |
| swap | none | not created |

The disk is initialized as GPT.

You may edit `installer/iso.toml` before running the workflow if your machine needs a different partition layout, timezone, hostname, network behavior, or local account. The workflow makes a temporary runtime copy of this file and applies the selected password/SSH options only to that copy; it does not rewrite the committed `installer/iso.toml`.

## What the workflow does

A manual build performs these steps:

1. Validate the selected login options and, when requested, the `SSH_PUBLIC_KEY` secret.
2. Generate a temporary installer configuration with the selected password/SSH mode.
3. Resolve the selected bootc image tag to its current amd64 digest.
4. Verify the exact digest with Cosign when signature verification is enabled.
5. Build a disposable AlmaLinux installer container containing Anaconda, Lorax, GRUB and EFI tooling.
6. Pull the verified bootc payload and `bootc-image-builder`.
7. Build a `bootc-installer` ISO using the Kickstart from the temporary installer configuration.
8. Create `alma-black-box-installer.iso` and `SHA256SUMS`.
9. Upload them as a short-lived GitHub Actions artifact.
10. Put the direct artifact download link and selected access mode in the workflow Summary.

The Anaconda/Lorax packages used to construct the ISO are **installer-only**. They are not added to the Alma Black Box production image being installed.

## Repository layout

```text
.github/workflows/build-installer.yml  Manual ISO build workflow and access-mode generation
installer/Containerfile                Disposable AlmaLinux installer environment
installer/iso.toml                     Base Kickstart and partition layout
cosign.pub                              Alma Black Box image verification key
```

## Related project

- Alma Black Box: https://github.com/highwaytoit/alma-black-box
- AlmaLinux bootc images: https://github.com/AlmaLinux/bootc-images
- bootc: https://github.com/bootc-dev/bootc
- osbuild / image-builder: https://github.com/osbuild

## License

Apache-2.0. See [LICENSE](LICENSE).
