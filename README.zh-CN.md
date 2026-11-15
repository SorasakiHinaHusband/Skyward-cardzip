**简体中文** | [English](README.md)

# Skyward cardzip

把逐分区镜像打成 **Skyward 卡刷 zip** 的独立工具，适合那些原本要用 fastboot
刷的移植包（例如 `fastboot flash system system.img`）。它不绑定某一个机型：
分区名可以配置，任何 Skyward 0.3.x recovery 都能运行生成的脚本。

生成的 `META-INF/com/google/android/update-binary` 会把每个镜像写进当前槽位
的逻辑分区或物理分区；写入前先检查镜像是否放得进目标分区，最后关闭 AVB。

## 快速开始

```sh
python3 make-cardzip.py \
  --images /path/to/port/images \
  --extra  /path/to/official/images \
  --fill   system_dlkm --fill vendor_dlkm \
  --skip   my_company --skip my_preload \
  --out    OnePlus12_Port_CardFlash.zip \
  --description "OnePlus 12 ColorOS 17 port"
```

然后把 zip 拷到手机，在 Skyward 里安装：

```
安装 → 选择文件 → OnePlus12_Port_CardFlash.zip
```

## 打包内容

`--images` 是主来源。里面的每个 `.img` 都必须是已知分区名，或者用
`--logical` / `--physical` 明确指定；否则工具会直接报错，不会猜。

`--skip` 用于“包里有，但我不想刷”的分区，保留手机原分区。一加 12 的移植包
里通常是 `my_company` 和 `my_preload`。

`--extra` 用于从官方包补缺失分区。它必须配合明确的 `--fill NAME` 使用，
这样即使官方目录是一整套镜像，也不会悄悄覆盖你不想动的分区。被补的分区必须
在 `--images` 里缺失；官方目录里的 `boot.img`、`dtbo.img`、`super.img` 等会
被忽略。

已知逻辑分区：

```
system system_ext product vendor odm
my_bigball my_carrier my_company my_engineering my_heytap
my_manifest my_preload my_product my_region my_stock
system_dlkm vendor_dlkm odm_dlkm
```

已知物理分区：`vbmeta`、`vbmeta_system`、`vbmeta_vendor`。
`recovery` 默认忽略；只有确实想覆盖 Skyward 时才加 `--include-recovery`
（Skyward 自己的“保留 recovery”选项之后可能又把它写回来）。

镜像必须是**原始分区镜像**（EROFS、ext4 等），不能是 Android sparse image。
如果 `file image.img` 显示 “Android sparse image”，先用 `simg2img` 转换。
`super.img` 不处理：这个工具只写手机 super 里已经存在的各个逻辑分区。

## 脚本做了什么

1. 用 `getprop ro.boot.slot_suffix` 读当前槽位（读不到就用 `_a`）。
2. 对每个逻辑镜像：
   - 检查 `/dev/block/mapper/<name><slot>` 是否存在；
   - 读取镜像大小和目标分区大小（`blockdev --getsize64`）；
   - 如果 `image > partition`，拒绝写入；
   - 用 `unzip -p "$ZIP" images/<name>.img > "$dev"` 写入。
3. 对每个 vbmeta 物理镜像，写入 `/dev/block/by-name/<name><slot>`。
4. 执行 `avbctl --force disable-verity`、
   `avbctl --force disable-verification`，然后 `sync`。

它不会使用 `unzip -p … | dd`。测试用的 recovery 上，`dd` 在大文件管道复制
结束后会以 134/139 退出，虽然字节已经写完；直接重定向可以完全避开这个问题。
它也不会在运行时用 `unzip -l | awk … exit`，因为提前退出会关掉管道，让
recovery 的 `unzip` 异常终止。

## 参数

| 参数 | 含义 |
| --- | --- |
| `--images DIR` | 主镜像目录，文件名为 `<分区>.img` |
| `--extra DIR` | 额外目录，可重复；只配合 `--fill` 使用 |
| `--fill NAME` | 从 `--extra` 取这个缺失分区，可重复 |
| `--skip NAME` | 不刷这个镜像，保留手机原分区，可重复 |
| `--out FILE` | 输出 zip（ZIP64，stored） |
| `--logical NAME` | 额外指定一个逻辑分区名 |
| `--physical NAME` | 额外指定一个物理分区名 |
| `--include-recovery` | 同时刷 `recovery.img` |
| `--no-avb` | 不执行 `avbctl` |
| `--description TEXT` | 显示在安装日志和 `cardflash.txt` 里 |
| `--dry-run` | 只打印会打包哪些镜像，不生成 zip |

## 安装前注意

- 电量 50% 以上，插着线，写入过程中不要拔。
- zip 没有签名。Skyward 里要关闭 **安装前校验签名**，否则会拒绝安装。
- 如果 Skyward 停在 `unmount /mnt/system_ext: Device or resource busy`，
  在 `adb shell` 里执行 `umount -l /mnt/system_ext` 再重试。
- 移植包必须和手机的 super 布局、槽位兼容。这个工具只检查镜像放不放得下，
  不能把不兼容的移植包变成能开机的包。
- 保留原厂 boot/recovery/super 镜像，或者 EDL 包，以便回滚。

## 实测

用一加 12 的 `OnePlus12_17.0.1.202_Port_From_FindX10ProMax&OnePlus16` 镜像
组合验证过：移植包的 `system`/`vendor`/`odm`/`my_*`，官方 16.0.10.501 的
`system_dlkm`/`vendor_dlkm`，以及移植包的 `vbmeta*`。通过 Skyward 安装、
关闭 AVB 后，手机正常开机进入移植系统。

## 许可证

GPL-3.0-or-later，与 Skyward 一致。见 `LICENSE`。
