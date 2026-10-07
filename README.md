# Kali NetHunter Pro for the OnePlus 7 Pro (guacamole)

[Kali NetHunter Pro](https://www.kali.org/docs/nethunter-pro/) (Kali Linux + Phosh,
on a mainline Linux kernel instead of Android) for the **OnePlus 7 Pro**
(`guacamole`, GM1910/GM1911/GM1913/GM1917, Snapdragon 855 / SM8150).

Built on the [sm8150-mainline](https://gitlab.com/sm8150-mainline/linux) kernel
used by postmarketOS, and the official
[kali-nethunter-pro build scripts](https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-pro).
Developed and tested on a GM1911 (India). Also boots on a GM1913 (EU, 8 GB, Samsung KLUEG8UHDB UFS); see [GM1913 notes](#gm1913-notes).

> [!WARNING]
> This is a community port and experimental. It **replaces Android** on your
> phone. Back up everything first (see [Before you start](#before-you-start)).

## Status

| Feature | Status | Notes |
| --- | --- | --- |
| Boot, Kali + Phosh | Works | |
| Display | Works | Bootloader framebuffer (simpledrm); no display driver for the DSC command-mode panel |
| GPU (Adreno 640) | Works (v0.2) | Render-only: Phosh and apps draw on the GPU (Mesa freedreno, OpenGL ES 3.2 / OpenGL 4.6), frames are shown through simpledrm. Falls back to CPU rendering if the GPU doesn't come up |
| Touchscreen | Works | |
| Wi-Fi (built-in) | Works | Scanning and connecting on 2.4 and 5 GHz; needs the Android 12 firmware |
| Battery level | Works | PM8150B fuel gauge; charges over USB |
| USB networking + SSH | Works | Phone is `10.66.0.1` |
| App store (Software + Flathub) | Works (v0.2) | |
| USB OTG (host mode) | Separate boot image | Swap boot image to use USB Wi-Fi adapters etc. |
| Modem firmware | Loads | Calls/SMS/mobile data untested |
| Bluetooth | Works (after v0.2) | WCN3990 over UART: scanning and pairing tested. Bluetooth audio needs audio support, which is missing. Suspend with Bluetooth on is untested |
| Audio, camera, sensors, fingerprint | Not working | |

Performance: v0.1 has no GPU driver, so the CPU draws every frame of a
1440×3120 panel with pixman/cairo software renderers (animations off). Later
builds render on the Adreno 640 GPU instead and only fall back to software
rendering if the GPU driver doesn't load. (v0.1 also pinned the CPU
at full speed; that drained the battery faster than a PC's USB port can
charge it, so it was dropped. To remove it on v0.1:
`sudo systemctl disable --now cpu-performance`.)

## Before you start

You need:

* A OnePlus 7 Pro with an **unlocked bootloader**.
* A Linux PC with `fastboot` and `adb` (`android-tools`).
* **The final Android 12 (OxygenOS 12) firmware on the active slot**, the
  one you flash Kali to. The Wi-Fi firmware in older OxygenOS versions refuses
  to connect with mainline Linux, and the device tree uses the Android 12
  memory layout. On first boot, firmware is extracted from the slot the phone
  booted from.

### Back up your phone's unique partitions

`persist`, `modemst1`, `modemst2`, `fsg`, `fsc` hold your IMEI and radio
calibration. They are never touched by this port, but back them up anyway
before experimenting, e.g. from a recovery with a root adb shell:

```sh
for p in persist modemst1 modemst2 fsg fsc; do
    adb exec-out "dd if=/dev/block/by-name/$p 2>/dev/null" > $p.img
done
```

Keep these files private: they contain your IMEI.

### Get the Android 12 firmware

The easiest way is [LineageOS](https://wiki.lineageos.org/devices/guacamole/),
which ships the Android 12 firmware:

1. Follow the LineageOS install guide for guacamole (flash its `dtbo`,
   `vbmeta` and `boot`, boot into Lineage Recovery, sideload
   `copy-partitions`, format data, sideload the LineageOS zip). The zip
   installs to the other slot and makes it active, so the active slot now
   has the Android 12 firmware.
2. Optionally boot LineageOS once to check the phone works.

## Install

Download the images from the [latest release](../../releases/latest), or
[build them](#building-from-source).

1. Reboot the phone into **fastboot mode**: power off, then hold
   **Power + Volume Up**.
2. Check the downloads and unpack the rootfs (it needs ~5.4 GB):

   ```sh
   sha256sum -c SHA256SUMS
   xz -d nethunter-pro-guacamole-*.rootfs.simg.xz
   ```

3. Flash (all commands act on the **active slot**):

   ```sh
   fastboot flash boot nethunter-pro-guacamole-*.boot.img
   fastboot erase dtbo
   fastboot flash userdata nethunter-pro-guacamole-*.rootfs.simg
   fastboot reboot
   ```

   (Self-built images are named `kali-nethunterpro-*.boot-guacamole.img` and
   `kali-nethunterpro-*.rootfs.simg`.)

   `fastboot erase dtbo` is **required**: the bootloader would otherwise apply
   Android's device-tree overlays to the mainline kernel and it would crash
   instantly. Flashing `userdata` erases Android's data.
4. The first boot extracts the device's firmware (modem, Wi-Fi, DSPs) from the
   phone's own partitions and **reboots once by itself**. No proprietary
   firmware is distributed with these images.
5. Log in with user **`kali`**, password **`1234`**. Change it: `passwd`.
6. **v0.1 only:** turn on boot marking, or after about 7 boots the bootloader
   gives up on the slot and shows *"The current image (boot/recovery) have
   been destroyed"* (see [Recovery](#recovery)):

   ```sh
   sudo systemctl enable --now qbootctl
   ```

### SSH over USB

The phone brings up a USB network and runs an SSH server:

```sh
# on the PC (NetworkManager may also do this for you)
sudo ip addr add 10.66.0.2/24 dev usb0   # interface name may differ
ssh kali@10.66.0.1
```

### USB OTG (external Wi-Fi adapters, etc.)

The USB controller can't switch roles automatically yet. To use USB host
mode, flash the OTG boot image instead (this disables USB networking):

```sh
fastboot flash boot nethunter-pro-guacamole-*.boot-otg.img
```

Flash the normal boot image again to get USB networking back.

### Recovery

* The phone can always be put into fastboot mode with **Power + Volume Up**.
* **"The current image (boot/recovery) have been destroyed and can not
  boot"**: the bootloader gives each slot 7 tries and only resets the count
  when Linux marks the boot successful (`qbootctl.service`, not enabled in
  v0.1). Nothing is actually destroyed. Go to fastboot mode, re-arm the slot
  and boot, then enable the service (step 6 of [Install](#install)):

  ```sh
  fastboot getvar current-slot     # e.g. "current-slot: b"
  fastboot set_active b            # the slot from above
  fastboot reboot
  ```
* If Kali really fails to boot several times, the bootloader marks the slot
  unbootable and switches slots. Fix with
  `fastboot set_active a` (or `b`) after re-flashing.
* Do **not** run `reboot bootloader` from Linux: on this device it lands in
  Qualcomm EDL mode (`05c6:9008`). Hold Power + Volume Up for ~15 s to leave it.

## Building from source

Requirements: a Linux x86_64 host with `git`, `curl`, `patch`, `img2simg`
(`android-tools`), and **podman** (or docker) with access to `/dev/kvm`
(the image build runs in a VM via debos/fakemachine).

```sh
git clone https://github.com/haintrainn/nethunter-pro-guacamole
cd nethunter-pro-guacamole
./build.sh
```

`build.sh`:

1. clones the upstream kali-nethunter-pro build scripts at the commit in
   `nethunter-pro/UPSTREAM_COMMIT` and applies `nethunter-pro/kali-nethunter-pro.patch`;
2. builds the kernel packages with `kernel/build-kernel.sh` (downloads and
   verifies the sm8150-mainline v6.17.0 tarball, applies postmarketOS' and
   this project's patches, builds with LLVM in a Kali container);
3. builds Kali's phoc with this port's patch (`phoc/build-phoc.sh`,
   cross-compiled for arm64 in a Kali container) so it is installed over
   the stock package;
4. builds the image (`build-in-container.sh -v wip -D phosh`);
5. converts the rootfs to an Android sparse image for fastboot.

Output lands in `kali-nethunter-pro/output/`. The first build takes about an
hour (most of it is installing packages under arm64 emulation).

## What this port changes

### Kernel (`kernel/patches/`)

| Patch | What and why |
| --- | --- |
| `0001` | Modem firmware name `.mdt` → `.mbn` (as extracted by droid-juicer); adds `sm8150-oneplus-guacamole-otg.dts` (USB host mode) |
| `0002` | guacamole device tree: `qcom,msm-id`/`board-id` so the bootloader accepts the DTB; disable `dispcc` (it reprograms the display PLL and freezes the bootloader framebuffer); framebuffer interconnect paths and physical size; enable QUP2 + GPI DMA and release the touchscreen reset GPIO; Wi-Fi supplies; PM8150B charger/fuel gauge + battery; Android 12 firmware carve-outs (160 MiB modem region) |
| `0003` | simpledrm: keep the framebuffer's `interconnects` voted so scanout isn't starved |
| `0004` | ath10k: skip the WMI quiet-mode command on WCN3990 (crashes WLAN.HL.3.x firmware) and force passive 5 GHz scans, after [this linux-wireless series](https://ratatoskr.run/linux-wireless/2026/03/15793732/t) |
| `0005` | simpledrm: report a DSI connector when the framebuffer has a `panel` node, so phoc/phosh treat the display as the built-in panel (touch mapping, rounded-corner margins in the top bar) |
| `0006` | guacamole: enable the Adreno 640 GPU and its GMU (zap shader extracted by droid-juicer) and reserve a ramoops region for crash logs |
| `0007` | msm: don't drop a reference on the *exporter's* GEM object when freeing an imported dma-buf. phoc renders on the GPU into buffers imported from simpledrm; the bad reference drop underflowed simpledrm's refcount and crashed the phone |
| `0008` | guacamole: reserve all the memory the Android 12 firmware owns (full 85 MiB TrustZone region, XBL/AOP, secure CDSP, rmtfs guard pages), taken from the downstream device tree. Linux handing out those pages is an XPU violation: the phone reset into Qualcomm crash-dump mode with no kernel log once memory filled up — within seconds during `apt install`, at random otherwise |
| `0009` | guacamole: enable WCN3990 Bluetooth on UART13 (GPIO 43-46) with the downstream supplies, select the revision 21 firmware extracted by droid-juicer, and give the UART the `serial1` alias the serial driver needs to probe. After the [OnePlus 7T Pro port](https://github.com/Sr-0w/hotdog-linux-bringup), which has the same wiring |
| `0010` | ufs-qcom: on SM8150 never negotiate above HS-G3. The controller reports HW version 4.x and defaults to HS-G4, but on the GM1913 the lanes fail to train at HS-G4: `UIC Lane error` right after the power mode change, then the UFS error handler resets the link every ~0.4 s, the root filesystem disappears and the initramfs drops to a shell |
| `0011` | fts_touch: silence `-Wstrict-prototypes` for the downstream FTS driver so the kernel builds with the clang in current Kali rolling (clang 21), which turns the warning into an error |

`kernel/nethunter.config` switches the display to simpledrm, builds a plain
`Image.gz` for the Android bootloader, and enables common USB Wi-Fi/serial
adapters. `kernel/pmos/` holds postmarketOS' kernel config and patches.

### phoc (`phoc/patches/`)

| Patch | What and why |
| --- | --- |
| `0001` | Allocate offscreen buffers (the app thumbnails in Phosh's overview) on the renderer's device. phoc's allocator lives on the display device, simpledrm, whose Mesa backend rejects many thumbnail sizes (`gbm_bo_create failed: Invalid argument`). With GPU rendering, thumbnails failed and app cards in the overview couldn't be swiped away |

The package is versioned `0.57.0-1+guacamole1`; a newer phoc from Kali
replaces it (and brings the bug back) until the fix is upstream.

### Image (`nethunter-pro/kali-nethunter-pro.patch`)

* `wip` device config for guacamole (boot image offsets, OTG variant, kernel
  command line `clk_ignore_unused pd_ignore_unused`).
* droid-juicer config to extract firmware on first boot, from the slot the
  phone booted (`qbootctl`). On its own droid-juicer reads unsuffixed
  partitions from slot a, and this bootloader doesn't tell Linux its slot.
* Enables USB networking; installs Settings, Files, Text Editor, Calculator.
* App store: GNOME Software with Flathub (Kali publishes no AppStream
  metadata, so Kali packages only show up as updates; install them with
  `apt`). Updates are not downloaded in the background.
* GPU: load `msm` at boot (it has no module alias for a GPU without MDSS)
  with `separate_gpu_kms=1`, and pick phoc's renderer when Phosh starts:
  GLES on the Adreno render node, or pixman/cairo if it isn't there.
  Keep the GPU powered while its driver is bound (udev rule): letting it
  runtime-suspend is suspected of resetting the phone.
* Phosh tuning for the framebuffer: output scale 3.
* Don't dump remote processor memory when the modem crashes (reading it can
  force the whole phone into Qualcomm crash-dump mode); just restart it.
* Display panel description (rounded corners) for phosh, which has none for
  guacamole, loaded through `G_RESOURCE_OVERLAYS` so the status bar icons
  aren't cut off by the screen corners.
* Fixes to the build scripts (`wip` variant, rootfs partition extraction).

## GM1913 notes

Tested on a GM1913 (EU, 8 GB RAM, UFS `SAMSUNG KLUEG8UHDB-C2D1` rev 0400) with this repository's v0.2 userspace and a kernel built with patches `0010` and `0011`.

What works: boot to systemd, UFS stable (all six LUNs, full-size root filesystem), USB networking and SSH at `10.66.0.1`. Wi-Fi, charging behaviour and long-term stability were not yet checked on this unit.

**Without patch `0010` the GM1913 does not boot.** The kernel log shows the UFS link going to HS-G4, two lanes (`gear=[4,4], lane[2,2], FAST MODE`), then `UIC Lane error reported`, `ufshcd_err_handler started ... saved_uic_err = 0x4/0x44`, and a reset loop (hundreds of `host_reset`s). The initramfs then stops with its rescue shell, and the `Synchronize Cache(10) failed` lines you see are the driver removing the storage units after a reset. Forcing HS-G3 (`ufs_qcom` host gear 3) gives `gear=[3,3], lane[2,2]` and a clean boot. The release v0.2 images contain the kernel without this patch, so on a GM1913 you need to rebuild the kernel (`kernel/build-kernel.sh`) or splice a patched kernel into the boot image.

Android 12 firmware prerequisite on a GM1913:

* On stock OxygenOS 12 `fastboot oem unlock` fails with `Device cannot be unlocked for technical reason`. This is a OnePlus bug. Roll back to OxygenOS 11 with the official downgrade package, unlock the bootloader there, then update to the final Android 12 again. The unlock survives the update.
* The LineageOS install guide for guacamole requires the stock Android 12 firmware to be installed first. LineageOS itself does not ship the firmware. The final Android 12 firmware is identical in both slots except for the signature of `abl`.
* Lineage Recovery has no ADB authorisation prompt. Use **Advanced > Enable ADB** in the recovery to get a root shell, for example to back up `persist`, `modemst1`, `modemst2`, `fsg` and `fsc`.

Debugging tip: the kernel's early log is lost when the UFS error loop floods the log buffer. Boot with `log_buf_len=16M` on the kernel command line (a 511-character limit applies to the boot image header), and add a USB serial gadget (configfs ACM function) in the initramfs to stream `/proc/kmsg` to the host. A kernel built with `#define DEBUG` in `drivers/ufs/core/ufshcd.c`, `drivers/ufs/host/ufs-qcom.c` and `drivers/phy/qualcomm/phy-qcom-qmp-ufs.c` shows the exact UFS bring-up sequence.

## Known issues / TODO

* No proper display driver (the panel is a DSC command-mode panel), so the
  GPU renders and simpledrm shows the result.
* Audio, camera, sensors and fingerprint are not enabled.
* The USB port doesn't switch between device and host mode automatically.
* v0.1 extracts firmware from slot a even when booted from slot b: if slot a
  has older firmware, Wi-Fi doesn't connect. Fixed after v0.1.
* v0.1 resets into Qualcomm crash-dump mode (`05c6:900e`) under memory
  pressure (e.g. `apt install`); fixed by patch `0008`. Hold Power + Volume Up
  for ~15 s to get out.
* With the GPU allowed to runtime-suspend, the phone reset into crash-dump
  mode a few minutes after Phosh switched to GPU rendering. The GPU is now
  kept powered while its driver is loaded; stability of this is still being
  tested.

Contributions welcome.

## Credits

* [sm8150-mainline](https://gitlab.com/sm8150-mainline/linux) and the
  [postmarketOS](https://postmarketos.org) guacamole port.
* [Mobian](https://mobian.org) and the
  [Kali NetHunter Pro](https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-pro) build scripts.
* The ath10k WCN3990 HL3.x workarounds by the author of the linked
  linux-wireless series (tested on the OnePlus 7T).
* [LineageOS](https://lineageos.org) for the firmware packaging and downstream
  device trees used as reference.
* Robin Snyders' [hotdog-linux-bringup](https://github.com/Sr-0w/hotdog-linux-bringup)
  (OnePlus 7T Pro), the reference for the Bluetooth device-tree node.

## License

Scripts, configuration and documentation in this repository:
[GPL-3.0-or-later](LICENSE), like the kali-nethunter-pro build scripts they
modify. Kernel patches (`kernel/patches/`, `kernel/pmos/`) are GPL-2.0, like
the Linux kernel.
