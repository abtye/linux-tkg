## linux-tkg

此存储库提供了从[Linux官方git仓库](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)自动下载、修补和编译Linux内核的脚本，并提供了一系列旨在获得更好桌面或游戏体验的补丁。通过编辑`customization.cfg`文件和遵循交互式安装脚本，可以启用或禁用提供的补丁。您可以使用外部配置文件（默认为`$HOME/.config/frogminer/linux-tkg.cfg`，可用`customization.cfg`中的`_EXT_CONFIG_PATH`变量进行调整）。您还可以使用自己的补丁（更多信息请参阅`customization.cfg`文件）。

### 重要信息

- **非 pacman 发行版的支持是实验性的。我们邀请您报告可能遇到的问题。**
- **如果您的发行版不使用 systemd ，请在`customization.cfg`中设置`_configfile="running-kernel"`，否则你会获得一个无法启动的内核**
- 请记住，使用 GCC 构建最新的 Linux 内核将需要约 20-25GB 的磁盘空间。使用 llvm/clang、LTO、ccache 或在 defconfig 中启用更多驱动程序将提高这一要求，因此请确保您正在使用的磁盘上有足够的可用空间来构建
- In `intel_pstate` driver, frequency scaling aggressiveness has been changed with kernel 5.5 which results in stutters and poor performance in low/medium load scenarios (for higher power savings). As a workaround for our gaming needs, we are setting it to passive mode to make use of the `acpi_cpufreq` governor passthrough, keeping full support for turbo frequencies. It's combined with our aggressive ondemand governor by default for good performance on most CPUs while keeping frequency scaling for power savings. In a typical low/medium load scenario (Core i7 9700k, playing Mario Galaxy on Dolphin emulator) intel_pstate in performance mode gives a stuttery 45-50 fps experience, while passive mode + aggressive ondemand offers a locked 60 fps.
- 如果 Nvidia 的专有驱动程序不支持您选择的内核OOTB，则可能需要进行修补：[Frogging Family Nvidia-all](https://github.com/Frogging-Family/nvidia-all) 可以自动为您完成此操作。
- 关于 Arch Linux 上 5.9 之前的内核的注意事项：由于默认情况下切换到`zstd`压缩`initramfs`，您将在压缩开始时遇到`invalid magic at start of compress`（压缩开始时魔法无效）错误。您可以通过编辑`/etc/mkinitcpio.conf`来取消注释`COMPRESSION="lz4"`行（只是举例，因为这是除zstd之外的最佳选项），并使用`sudo mkinitpcio -P`为所有内核重新生成`initramfs`，从而解决此问题


### 自定义选项
#### 替换CPU调度器

[CFS](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler) 
是 ≤ 6.5 的原始内核源代码中唯一可用的CPU调度器。
[EEVDF](https://lwn.net/Articles/925371/)
是 ≥ 6.6 的原始内核源代码中唯一可用的CPU调度器。

它目前的实现不允许在内核级别注入额外的调度器，需要替换它。一次只能修补一个调度器。

然而，使用[Sched-ext](https://github.com/sched-ext/scx)，可以在运行时注入CPU调度器。默认情况下，我们在 ≥ 6.8 时提供支持。

多亏了@sirlucja，Arch 用户可以从[AUR](https://aur.archlinux.org/packages/scx-scheds-git)的`scx-scheds`包获得scx调度器（对于持久化，在`/etc/default/scx`中设置调度器并启用`scx`服务）

在构建内核时，linux-tkg中可以选择使用替代调度器：

- Project C / PDS & BMQ by Alfred Chen: [blog](http://cchalpha.blogspot.com/ ), [code repository](https://gitlab.com/alfredchen/projectc)
- MuQSS by Con Kolivas : [blog](http://ck-hack.blogspot.com/), [code repository](https://github.com/ckolivas/linux)
- CacULE by Hamad Marri - 基于 CFS : [code repository](https://github.com/hamadmarri/cacule-cpu-scheduler)
- Task Type (TT) by Hamad Marri - 基于 CFS : [code repository](https://github.com/hamadmarri/TT-CPU-Scheduler)
- BORE (Burst-Oriented Response Enhancer) by Masahito Suzuki - 基于 CFS/EEVDF : [code repository](https://github.com/firelzrd/bore-scheduler)
- Undead PDS : TkG's port of the pre-Project C "PDS-mq" scheduler by Alfred Chen. While PDS-mq got dropped with kernel 5.1 in favor of its BMQ evolution/rework, it wasn't on par with PDS-mq in gaming. "U" PDS still performed better in some cases than other schedulers, so it's been kept undead for a while.

在某些情况下，这些替代调度器可能会提供更好的性能/延迟比。每个调度程序的可用性取决于所选的内核版本：脚本将显示每个版本的可用内容。

#### 默认优化
- 内存管理和交换调整
- 调度优化
- `CFS/EEVDF` 优化
- 使用["Cake"](https://www.bufferbloat.net/projects/codel/wiki/CakeTechnical/)网络队列管理系统
- 默认设置 `vm.max_map_count=16777216`
- 从 [Clear Linux 的补丁组](https://github.com/clearlinux-pkgs/linux) 中精选的补丁

#### 可选优化
`customization.cfg`文件提供了许多额外调整的开关：

- [NTsync](https://repo.or.cz/linux/zf.git/shortlog/refs/heads/ntsync5)、`Fsync` 和 `Futex2`（已弃用）支持：可以提高游戏性能，需要打了补丁的 wine，比如 [wine-tkg](https://github.com/Frogging-Family/wine-tkg-git)
- [Graysky 的每个CPU架构原生优化](https://github.com/graysky2/kernel_compiler_patch)：将编译后的代码调整到指定的CPU
- 使用`O2`/`O3`和`LTO`（仅Clang）优化。
  - **关于v3.0.2（2021-11-21）和Clang之前的DKMS模块的警告:** DKMS v3.0.1及更早版本将默认使用GCC，这将无法针对Clang构建的内核构建模块。例如，这将破坏Nvidia驱动程序。可以强制较旧的DKMS使用Clang，但不建议这样做。
- 使用[Modprobed-db](https://github.com/graysky2/modprobed-db)的数据库可以减少编译时间，并生成一个只包含其中列出的模块的较小内核。**不推荐**
  - **警告**：一定要先仔细阅读[文档](https://wiki.archlinux.org/index.php/Modprobed-db)，因为它附带了可能导致内核无法启动的警告。
- "Zenify" patchset using core blk, mm and scheduler tweaks from Zen
- "Zenify" 补丁组使用来自 Zen 的 core blk, mm 和 调度器优化
- `ZFS` FPU 标志 (<5.9)
- 覆盖缺失的ACS功能
- [Waydroid](https://wiki.archlinux.org/title/Waydroid) 支持
- [OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB) 支持
- 提供自己的内核`.config`文件
- ...

#### 用户补丁

To apply your own patch files using the provided scripts, you will need to put them in a `linux<VERSION><PATCHLEVEL>-tkg-userpatches` folder -- where _VERSION_ and _PATCHLEVEL_ are the kernel version and patch level, as specified in [linux Makefile](https://github.com/torvalds/linux/blob/master/Makefile), the patch works on, _e.g_ `linux65-tkg-userpatches` -- at the same level as the `PKGBUILD` file, with the `.mypatch` extension. The script will by default ask if you want to apply them, one by one. The option `_user_patches` should be set to `true` in the `customization.cfg` file for this to work.


### Install procedure

For all the supported linux distributions, `linux-tkg` has to be cloned with `git`. Since it keeps a clone of the kernel's sources within (`linux-src-git`, created during the first build after a fresh clone), it is recommended to keep the cloned `linux-tkg` folder and simply update it with `git pull`, the install script does the necessary cleanup at every run.

#### Arch & derivatives
```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
makepkg -si
```
The script will use a slightly modified Arch config from the `linux-tkg-config` folder, it can be changed through the `_configfile` variable in `customization.cfg`. The options selected at build-time are installed to `/usr/share/doc/$pkgbase/customization.cfg`, where `$pkgbase` is the package name.

**Note:** the `base-devel` package group is expected to be installed, see [here](https://wiki.archlinux.org/title/Makepkg) for more information.

#### DEB (Debian, Ubuntu and derivatives) and RPM (Fedora, SUSE and derivatives) based distributions

**Important notes:**
An issue has been reported for Ubuntu where the stock kernel cannot boot properly any longer, the whereabouts are not entirely clear (only a single user reported that, see https://github.com/Frogging-Family/linux-tkg/issues/436).

The interactive `install.sh` script will create, depending on the selected distro, `.deb` or `.rpm` packages, move them in the the subfolder `DEBS` or `RPMS` then prompts to install them with the distro's package manager.
```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```
Uninstalling custom kernels installed through the script has to be done
manually. `install.sh` can can help out with some useful information:
```shell
cd path/to/linux-tkg
./install.sh uninstall-help
```
The script will use a slightly modified Arch config from the `linux-tkg-config` folder, it can be changed through the `_configfile` variable in `customization.cfg`.

#### Generic install
The interactive `install.sh` script can be used to perform a "Generic" install by choosing `Generic` when prompted. It git clones the kernel tree in the `linux-src-git` folder, patches the code and edits a `.config` file in it. The commands to do are the following:
```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```
The script will compile the kernel then prompt before doing the following:
```shell
sudo cp -R . /usr/src/linux-tkg-${kernel_flavor}
cd /usr/src/linux-tkg-${kernel_flavor}
sudo make modules_install
sudo make install
sudo dracut --force --hostonly --kver $_kernelname $_dracut_options
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
**Notes:**
- All the needed dependencies to patch, configure, compile or install the kernel are expected to be installed by the user beforehand.
- If you only want the script to patch the sources in `linux-src-git`, you can use `./install.sh config`
- `${kernel_flavor}` is a default naming scheme but can be customized with the variable `_kernel_localversion` in `customization.cfg`.
- `_dracut_options` is a variable that can be changed in `customization.cfg`.
- `_libunwind_replace` is a variable that can be changed in `customization.cfg` for replacing `libunwind` with `llvm-libunwind`.
- The script uses Arch's `.config` file as a base. A custom one can be provided through `_configfile` in `customization.cfg`.
- The installed files will not be tracked by your package manager and uninstalling requires manual intervention. `./install.sh uninstall-help` can help with useful information if your install procedure follows the `Generic` approach.

#### Gentoo
The interactive `install.sh` script supports Gentoo by following the same procedure as `Generic`, symlinks the sources folder in `/usr/src/` to `/usr/src/linux`, then offers to do an `emerge @module-rebuild` for convenience
```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```
**Note:** If you're running openrc, you'll want to set `_configfile="running-kernel"` to use your current kernel's defconfig instead of Arch's. Else the resulting kernel won't boot.
