# Chrome OS VM for Chromium developers

This workflow allows developers with a Chromium checkout using [Simple Chrome]
to download and launch a Chrome OS VM on their workstations, update the VM with
locally built Chrome, and run various tests.

[TOC]

## Prerequisites

1.  [depot_tools installed]
2.  [Linux Chromium checkout]
3.  [Virtualization enabled]
4.  [Simple Chrome] set up

### Virtualization check
To check if kvm is already enabled:
```bash
(shell) if [[ -e /dev/kvm ]] && grep '^flags' /proc/cpuinfo | grep -qE 'vmx|svm'; then
echo 'KVM is working'; else echo 'KVM not working'; fi
```
If this fails, check the [Virtualization enabled] doc for instructions.

For Goobuntu HP Zx20, interrupt the BIOS bootup with Esc for Options, F10 for
Computer Setup, in the Security menu, System Security tab,
Enable Virtualization Technology in the overlay.

## Typography conventions

| Label         | Paths, files, and commands                             |
|---------------|--------------------------------------------------------|
|  (shell)      | on your build machine, outside the sdk/chroot          |
|  (sdk)        | inside the `chrome-sdk` [Simple Chrome] shell          |
|  (chroot)     | inside the `cros_sdk` [chroot]                         |
|  (vm)         | inside the VM ssh session                              |


## Download the VM

cd to your Chromium repository, and enter the [Simple Chrome] SDK environment
with `--download-vm`:
```bash
(shell) .../chrome/src $ cros chrome-sdk --board=amd64-generic \
--download-vm --clear-sdk-cache --log-level=info
```

### chrome-sdk options

*   `--download-vm` downloads a pre-packaged VM and QEMU (takes a few minutes).
*   `--clear-sdk-cache` recommended, clears the cache.
*   `--log-level=debug` for additional output (e.g. VM image download details).
*   `--board=betty` will download an ARC-enabled VM (Googler-only).
*   `--internal` will set $GN_ARGS to build and deploy an internal Chrome build.
*   `--version` to download a non-LKGM version, eg 10070.0.0.

Some boards do not generate VM images. `amd64-generic` and `betty` (for ARC,
internal only) are recommended.

## Launch a Chrome OS VM

From within the [Simple Chrome] environment:
```bash
(sdk) .../chrome/src $ cros_vm --start
```

### Viewing the VM
To view the VM in a window, you may need to launch `vncviewer`:
```bash
(shell) vncviewer localhost:5900 &
```

To install `vncviewer`:
```bash
(shell) sudo apt-get install vncviewer
```
If this package is not available on your system, any other VNC Viewer should
work as well.

### Stop the VM

```bash
(sdk) .../chrome/src $ cros_vm --stop
```

## Remotely run a sanity test in the VM

```bash
(sdk) .../chrome/src $ cros_vm --cmd -- /usr/local/autotest/bin/vm_sanity.py
```
The command output on the VM will be output to the console after the command
completes. Other commands run within an ssh session can also run with `--cmd`.

Note that the CrOS test private RSA key cannot be world readable, so you may
need to do:
```bash
(shell) .../chrome/src $ chmod 600 ./third_party/chromite/ssh_keys/testing_rsa
```

## SSH into the VM

```bash
(shell) .../chrome/src $  ssh root@localhost -p 9222
```
> Password is `test0000`

To avoid having to type a password and skip the RSA key warning:
```bash
(shell) .../chrome/src $ ssh -o UserKnownHostsFile=/dev/null -o \
StrictHostKeyChecking=no -i ./third_party/chromite/ssh_keys/testing_rsa \
root@localhost -p 9222
```

### Run a local sanity test in the VM

```
(vm) localhost ~ # /usr/local/autotest/bin/vm_sanity.py
```

## Run telemetry unit tests

To run telemetry unit tests, or perf unit tests:
```bash
(shell) .../chrome/src $ third_party/catapult/telemetry/bin/run_tests \
--browser=cros-chrome --remote=localhost --remote-ssh-port=9222 [test]
(shell) .../chrome/src $ tools/perf/run_tests \
--browser=cros-chrome --remote=localhost --remote-ssh-port=9222 [test]
```

Alternatively, to run these tests in local mode instead of remote mode, SSH into
the VM as above, then invoke `run_tests`:
```bash
(vm) localhost ~ # python \
/usr/local/telemetry/src/third_party/catapult/telemetry/bin/run_tests [test]
(vm) localhost ~ # python /usr/local/telemetry/src/tools/perf/run_tests [test]
```

## Update Chrome in the VM

### Build Chrome

For testing local Chrome changes on Chrome OS, use the [Simple Chrome] flow to
build Chrome (after entering the Simple Chrome SDK environment as described
above):
```bash
(sdk) .../chrome/src $ autoninja -C out_$SDK_BOARD/Release/ \
chromiumos_preflight
```

### Launch the VM

```bash
(sdk) .../chrome/src $ cros_vm --start
```

### Deploy your Chrome to the VM

```bash
(sdk) .../chrome/src $ deploy_chrome --build-dir=out_$SDK_BOARD/Release/ \
--to=localhost --port=9222
```

## Run an autotest in the VM

Autotests only exist inside the chromeos checkout, so they require running
`test_that` directly from the chroot.

From inside your [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ ./build_packages --board=amd64-generic
(chroot) ~/trunk/src/scripts $ test_that --board=amd64-generic localhost:9222 \
login_Cryptohome
```

## Run an ARC++ test in the VM

Download the betty VM:
```bash
(sdk) .../chrome/src $ cros chrome-sdk --board=betty --download-vm
```
vm_sanity will detect and run an ARC++ test:
```bash
(vm) localhost ~ # /usr/local/autotest/bin/vm_sanity.py
```
Run a cheets autotest from within your [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ ./build_packages --board=betty
(chroot) ~/trunk/src/scripts $ test_that --board=betty localhost:9222 \
cheets_ContainerMount
```

## Run a Chrome GTest binary in the VM

The following will create a wrapper script at out_$SDK_BOARD/Release/bin/ that
can be used to launch a VM, push the test dependencies, and run the GTest. See
the [chromeos-amd64-generic-rel] builder on Chromium's main waterfall for the
list of GTests currently running in VMs (eg: base_unittests, ozone_unittests).
```bash
(sdk) .../chrome/src $ autoninja -C out_$SDK_BOARD/Release/ $TEST
(sdk) .../chrome/src $ ./out_$SDK_BOARD/Release/bin/run_$TEST
```

## Launch a VM built by a waterfall bot

Find a waterfall bot of interest, such as
[amd64-generic-tot-chromium-pfq-informational], which is a FYI bot that builds
TOT Chrome with TOT Chrome OS, or [amd64-generic-chromium-pfq], which is
an internal PFQ builder. Pick a build, click on artifacts, and download
`chromiumos_qemu_image.tar.xz` to `~/Downloads/`

Unzip:
```bash
(shell) $ tar xvf ~/Downloads/chromiumos_qemu_image.tar.xz
```
Launch a VM from within the [Simple Chrome] environment:
```bash
(sdk) .../chrome/src $ cros_vm --start \
--image-path  ~/Downloads/chromiumos_qemu_image.bin
```

## Launch a locally built VM from within the chroot

Follow instructions to [build Chromium OS] and a VM image. In the [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ export BOARD=betty
(chroot) ~/trunk/src/scripts $ ./setup_board --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./build_packages --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./build_image \
--noenable_rootfs_verification test --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./image_to_vm.sh --test_image --board=$BOARD
```

You can either specify the image path or the board:
```bash
(chroot) ~/trunk/src/scripts $ cros_vm --start --image-path \
../build/images/$BOARD/latest/chromiumos_qemu_image.bin
(chroot) ~/trunk/src/scripts $ cros_vm --start --board $BOARD
```

You can also launch the VM from anywhere within your chromeos source tree:
```bash
(shell) .../chromeos $ chromite/bin/cros_vm --start --board $BOARD
```

## cros_run_vm_test

`cros_run_vm_test` runs various tests in a VM. It can use an existing VM or
launch a new one.

### In Simple Chrome

To launch a VM and run a sanity test:
```bash
(sdk) .../chrome/src $ cros_run_vm_test
```

To build chrome, deploy chrome, or both, prior to running tests:
```bash
(sdk) .../chrome/src $ cros_run_vm_test --build --deploy --build-dir \
out_$SDK_BOARD/Release
```

To run a Tast test:
```bash
(sdk) .../chrome/src $ cros_run_vm_test \
--cmd -- local_test_runner ui.ChromeLogin
```

To run an arbitrary test (for e.g. base_unittests), you would need to first
build base_unittests, copy the dependencies (the .so files) to the VM, and run
the test in the correct working directory:
```bash
(sdk) .../chrome/src $ autoninja -C out_$SDK_BOARD/Release base_unittests
(sdk) .../chrome/src $ ls out_$SDK_BOARD/Release/base_unittests \
out_$SDK_BOARD/Release/*.so > /tmp/files.txt
(sdk) .../chrome/src $ cros_run_vm_test --files-from /tmp/files.txt \
--cwd out_$SDK_BOARD/Release --cmd -- ./base_unittests
```
The set of files to transfer to the VM is specified via `--files` or
`--files-from`, a working directory via `--cwd` and the test command to run
via `--cmd`.

### In the chroot

These examples require a locally build VM, see
[Launch a locally built VM from within the chroot].

To run an individual autotest from within the chroot:
```bash
(chroot) ~/trunk/src/scripts $ cros_run_vm_test --board $BOARD \
--autotest test_LoginCryptohome
```

To run the autotest smoke suite:
```bash
(chroot) ~/trunk/src/scripts $ mkdir /tmp/results
(chroot) ~/trunk/src/scripts $ cros_run_vm_test --board $BOARD \
--results-dir=/tmp/results --autotest suite:smoke
```

This doc is at [go/cros-vm]. Please send feedback to [achuith@chromium.org].

[depot_tools installed]: https://www.chromium.org/developers/how-tos/install-depot-tools
[go/cros-qemu]: https://storage.cloud.google.com/achuith-cloud.google.com.a.appspot.com/qemu.tar.gz
[Linux Chromium checkout]: https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md
[Virtualization enabled]: https://g3doc.corp.google.com/tools/android/g3doc/development/crow/enable_kvm.md
[Simple Chrome]: https://chromium.googlesource.com/chromiumos/docs/+/master/simple_chrome_workflow.md
[chroot]: https://www.chromium.org/chromium-os/developer-guide
[amd64-generic-tot-chromium-pfq-informational]: https://build.chromium.org/p/chromiumos.chromium/builders/amd64-generic-tot-chromium-pfq-informational
[amd64-generic-chromium-pfq]: https://uberchromegw.corp.google.com/i/chromeos/builders/amd64-generic-chromium-pfq
[build Chromium OS]: https://www.chromium.org/chromium-os/developer-guide
[go/cros-vm]: https://chromium.googlesource.com/chromiumos/docs/+/master/cros_vm.md
[achuith@chromium.org]: mailto:achuith@chromium.org
[chromeos-amd64-generic-rel]: https://ci.chromium.org/p/chromium/builders/luci.chromium.ci/chromeos-amd64-generic-rel
