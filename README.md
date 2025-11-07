# dnf-rebuild-prototype

This repository is a proof-of-concept for `dnf-rebuild`, a DNF5 plugin that handles local image builds on bootc systems.
The prototype is implemented as a single bash script: `dnf-rebuild.sh`.

See [https://gitlab.com/fedora/bootc/tracker/-/issues/4](https://gitlab.com/fedora/bootc/tracker/-/issues/4) for background.

## Design goals

1. Provide a straightforward, "batteries-included" way to make local persistent changes to the package set on bootc systems. An rpm-ostree alternative.
2. Provide a declarative interface which allows the user to list desired packages in a file.
3. Allow users to layer non-RPM content
4. Make it easy to take a local layer definition managed by `dnf-rebuild` and "eject" it to be built in some non-local pipeline, and/or deployed to multiple machines. All configuration should fit in a single directory which can be tracked in version control.
5. Don't be too slow or waste too much disk space (hard).

## Potential imperative interface

An imperative interface could be supported by adding subcommands/arguments to dnf-rebuild (`dnf rebuild --add`, `dnf rebuild --remove`) that operate on the infile. The infile should remain the source of truth.

Adapting the usual non-namespaced DNF commands (`dnf install`, `dnf remove`) to operate on the infile may sound like a nicer UX, but it would be a bad idea in my opinion. Each of the existing DNF commands performs a separate transaction. If you run `dnf install --best foo` and then `dnf install --no-best foo`, each command will perform its own transaction with its own set of solver flags. With the declarative interface (libpkgmanifest), the entire infile is resolved as one transaction, and then the manifest is executed as one transaction. These are very different paradigms, and I fear trying to mash them together would create more confusion than it would solve.

## Current design

In this PoC, dnf-rebuild drives bootc as opposed to bootc driving dnf-rebuild ([context](https://gitlab.com/fedora/bootc/tracker/-/issues/4#note_2379057178)). This pattern lets us keep bootc as a lower-level tool without much knowledge of build systems, but it means `dnf-rebuild` has to take over much of the interface from bootc. When bootc is switched to the local image built by dnf-rebuild, bootc just sees a local image as the base image and `bootc upgrade` won't work. Instead, users must use `dnf-rebuild --upgrade`. If we continue with this dnf-rebuild-driving-bootc paradigm, we'll probably want dnf-rebuild to keep parity with bootc's interface, which may be difficult to maintain. E.g. `dnf rebuild --apply`, `dnf rebuild --soft-reboot auto` would be expected to work.

### Usage

1. Run `./dnf-rebuild.sh --init` to initialize the configuration template under `/etc/dnf/rebuild`.
    A tree will be created at `/etc/dnf/rebuild` as follows:

    ```
    /etc/dnf/rebuild
    ├── Containerfile.base
    ├── Containerfile.in
    ├── Containerfile.postinstall
    └── rpms.in.yaml
    ```
     
    dnf-rebuild will use the spec image as reported by `bootc status` to initially populate `Containerfile.base`.

    Edit `Containerfile.base`, `Containerfile.postinstall`, and `rpms.in.yaml` to your liking. `Containerfile.in` is the entry point for the container build and may be overwritten by dnf-rebuild.

    - `Containerfile.base`: contains only the line `FROM <base image>`. Edit to change the base image.
    - `Containerfile.postinstall`: run after installing packages with DNF. Can be used to deploy non-RPM content.
    - `rpms.in.yaml`: [libpkgmanifest](https://github.com/rpm-software-management/libpkgmanifest) input file listing the packages to be installed. Currently, this file is treated as an [rpm-lockfile-prototype input file](https://github.com/konflux-ci/rpm-lockfile-prototype?tab=readme-ov-file#whats-the-input_file). DNF will use repositories configured in the base image as well as repositories configured in this infile.

2. Run `./dnf-rebuild.sh` with no arguments to rebuild the image and `bootc switch` to it. The changes will take effect next boot, like `bootc switch`. Follow [bootc-dev/bootc#76](https://github.com/containers/bootc/issues/76) for progress on `bootc switch --apply-live`.

    `./dnf-rebuild.sh --upgrade` will pull the latest version of the base image and then rebuild, similar to `bootc upgrade`.

    Under the hood, `dnf-rebuild.sh` is just running `podman build` on `/etc/dnf/rebuild/Containerfile.in` and mounting `/var/cache/libdnf5` inside the builder so that RPMs are cached between rebuilds.
