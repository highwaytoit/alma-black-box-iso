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
5. Leave the default bootc image unchanged unless you intentionally want to install another image.
6. Wait for the workflow to finish successfully.
7. Open the completed workflow run. The **Summary** contains a **Download installer artifact** link.
8. Download `alma-black-box-installer`, extract the ZIP, and use `alma-black-box-installer.iso`.

The artifact also contains `SHA256SUMS`.

Artifacts are intentionally retained for only **1 day**. This repository is a builder, not an ISO archive. If the artifact expires, simply run the workflow again.

## Default installation image

The default workflow input is:

```text
ghcr.io/highwaytoit/alma-black-box:10
```

The workflow resolves that moving tag to an immutable digest at build time and uses that exact digest for the installer payload.

The default Alma Black Box image is verified with the included `cosign.pub` before the ISO is built.

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

The partitioning and account configuration live in:

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

You may edit `installer/iso.toml` before running the workflow if your machine needs a different partition layout, timezone, hostname, network behavior, or local account.

## Default account

The supplied installer creates this temporary administrator:

```text
username: bbox
password: bbox
```

The account is a member of `wheel`. Root is locked.

The public temporary password is intentional: **do not put your real password into a public template repository or workflow file.** Change it locally immediately after the first successful boot.

After logging in as `bbox`, run:

```bash
passwd
```

Enter `bbox` once as the current password, then enter your new private password twice.

If you change the username in `installer/iso.toml`, use that account instead.

## What the workflow does

A manual build performs these steps:

1. Resolve the selected bootc image tag to its current amd64 digest.
2. Verify the exact digest with Cosign when signature verification is enabled.
3. Build a disposable AlmaLinux installer container containing Anaconda, Lorax, GRUB and EFI tooling.
4. Pull the verified bootc payload and `bootc-image-builder`.
5. Build a `bootc-installer` ISO using the Kickstart embedded in `installer/iso.toml`.
6. Create `alma-black-box-installer.iso` and `SHA256SUMS`.
7. Upload them as a short-lived GitHub Actions artifact.
8. Put the direct artifact download link in the workflow Summary.

The Anaconda/Lorax packages used to construct the ISO are **installer-only**. They are not added to the Alma Black Box production image being installed.

## Repository layout

```text
.github/workflows/build-installer.yml  Manual ISO build workflow
installer/Containerfile                Disposable AlmaLinux installer environment
installer/iso.toml                     Kickstart, partition layout and temporary account
cosign.pub                              Alma Black Box image verification key
```

## Related project

- Alma Black Box: https://github.com/highwaytoit/alma-black-box
- AlmaLinux bootc images: https://github.com/AlmaLinux/bootc-images
- bootc: https://github.com/bootc-dev/bootc
- osbuild / image-builder: https://github.com/osbuild

## License

Apache-2.0. See [LICENSE](LICENSE).
