---
title: 为 OnePlus 6 构建 LineageOS 22.2 并在使用 KernelSU 的情况下锁定 Bootloader
date: 2025-05-14 12:24:28
tags: LineageOS
categories: 技术
---
# 为 OnePlus 6 构建 LineageOS 22.2 并在使用 KernelSU 的情况下锁定 Bootloader

本文最后修改于 2025 年 5 月 14 日，某些内容随着时间的推移可能不再适用。

## 准备工作

- 一台 OnePlus 6 （代号 enchilada）；
- 一台搭载 Ubuntu 24.02 LTS 的具备超过 400GB 硬盘的主机；
- 能够访问 GitHub 的网络连接，我们需要下载超过 100GB 的文件。

## 参考链接

1. [LineageOS 适用于 OnePlus 6 的自行构建指南](https://wiki.lineageos.org/devices/enchilada/build/)
2. [LineageOS 的自签名指南](https://wiki.lineageos.org/signing_builds)
3. [XDA 上的使用自签名的 LOS 重新锁定 OnePlus 6T 上的引导加载程序](https://xdaforums.com/t/4113743/)
4. [XDA 上的使用 LOS 18.1 自签名版本重新锁定 OnePlus 8T 上的引导加载程序](https://xdaforums.com/t/4259409/)
5. [XDA 上的使用 LOS 19.1 自签名版本重新锁定 Google Pixel 5 上的引导加载程序](https://xdaforums.com/t/4500903/)
6. [XDA 上 @DevelLevel 的经验](https://xdaforums.com/t/4113743/post-88081535)

## 开始动手！

### 1. 下载 LineageOS 的源码

根据[**链接1**](https://wiki.lineageos.org/devices/enchilada/build/)的步骤一直操作，直到命令 `repo sync` 运行完成为止。

### 2. 集成 MindTheGapps（可选）

这一步不是必须的，但如果想使用 GApps，则必须在源码中集成，直接安装可刷入的 GApps 包会导致锁定 Bootloader 后无法启动。

至于为什么使用 MindTheGapps，是因为只有这个一直更新，并且能够方便的集成到 AOSP。

编辑 `~/android/lineage/.repo/manifests/default.xml` 文件，加入以下内容：

```xml
<remote name="MindTheGapps" fetch="https://gitlab.com/MindTheGapps/" />
<project path="vendor/gapps" name="vendor_gapps" remote="MindTheGapps" revision="vic" />
```

其中，`vic` 分支是适用于 Android 15 的 GApps，如果需要其他版本的 GApps，则需要修改为对应分支。分支名称可以在 [MindTheGapps 的 GitLab 仓库](https://gitlab.com/MindTheGapps/vendor_gapps)找到。

文件修改完成后，使用 `repo sync` 同步代码。代码同步完成后，将以下内容添加到 `~/android/lineage/device/oneplus/enchilada/device.mk` 文件的末尾：
```
include vendor/gapps/arm64/arm64-vendor.mk
```

### 3. 下载 OnePlus 6 的内核等特定设备文件

这个步骤可以参考 [LineageOS 的教程](https://wiki.lineageos.org/extracting_blobs_from_zips.html)，也可以选择 TheMuppets 整理好的文件。为了方便，我们选择后者。

运行以下命令从 GitHub 下载文件：
```bash
cd ~/android/lineage/
git clone -b lineage-22.2 --depth=1 https://github.com/TheMuppets/proprietary_vendor_oneplus_enchilada.git vendor/oneplus/enchilada
git clone -b lineage-22.2 --depth=1 https://github.com/TheMuppets/proprietary_vendor_oneplus_sdm845-common.git vendor/oneplus/sdm845-common
```

### 4. 生成签名密钥

要锁定 Bootloader，设备必须支持自定义信任根，而自定义信任根的证书必须由用户——也就是我们自己——生成并刷入手机。所以，我们首先需要生成用于签名的密钥。

密钥只需要生成一次，之后可以一直使用。

根据[**链接2**](https://wiki.lineageos.org/signing_builds)的步骤操作，以下所有命令中 `subject` 命令代表个人信息，按照自己的需求更改。

首先设置证书信息：
```bash
cd ~/android/lineage
subject='/C=CN/ST=Beijing/L=Beijing/O=Individual/OU=Personal/CN=DreamVoid/emailAddress=i@dreamvoid.me'
mkdir ~/.android-certs
for cert in bluetooth cyngn-app media networkstack nfc platform releasekey sdk_sandbox shared testcert testkey verity; do \
    ./development/tools/make_key ~/.android-certs/$cert "$subject"; \
done
```

将 RSA 位数从 2048 改为 4096：
```bash
cp ./development/tools/make_key ~/.android-certs/
sed -i 's|2048|4096|g' ~/.android-certs/make_key
```

生成密钥：
```bash
for apex in com.android.adbd com.android.adservices com.android.adservices.api com.android.appsearch com.android.appsearch.apk com.android.art com.android.bluetooth com.android.btservices com.android.cellbroadcast com.android.compos com.android.configinfrastructure com.android.connectivity.resources com.android.conscrypt com.android.devicelock com.android.extservices com.android.graphics.pdf com.android.hardware.authsecret com.android.hardware.biometrics.face.virtual com.android.hardware.biometrics.fingerprint.virtual com.android.hardware.boot com.android.hardware.cas com.android.hardware.neuralnetworks com.android.hardware.rebootescrow com.android.hardware.wifi com.android.healthfitness com.android.hotspot2.osulogin com.android.i18n com.android.ipsec com.android.media com.android.media.swcodec com.android.mediaprovider com.android.nearby.halfsheet com.android.networkstack.tethering com.android.neuralnetworks com.android.nfcservices com.android.ondevicepersonalization com.android.os.statsd com.android.permission com.android.profiling com.android.resolv com.android.rkpd com.android.runtime com.android.safetycenter.resources com.android.scheduling com.android.sdkext com.android.support.apexer com.android.telephony com.android.telephonymodules com.android.tethering com.android.tzdata com.android.uwb com.android.uwb.resources com.android.virt com.android.vndk.current com.android.vndk.current.on_vendor com.android.wifi com.android.wifi.dialog com.android.wifi.resources com.google.pixel.camera.hal com.google.pixel.vibrator.hal com.qorvo.uwb; do \
    subject='/C=CN/ST=Beijing/L=Beijing/O=Individual/OU=Personal/CN='$apex'/emailAddress=i@dreamvoid.me'; \
    ~/.android-certs/make_key ~/.android-certs/$apex "$subject"; \
    openssl pkcs8 -in ~/.android-certs/$apex.pk8 -inform DER -nocrypt -out ~/.android-certs/$apex.pem; \
done
```

将 `.pk8` 格式的私钥转换为 `.key` 格式：
```bash
cd ~/.android-certs
openssl pkcs8 -in releasekey.pk8 -inform DER -out releasekey.key -nocrypt
```

### 5. 进行一次 LineageOS 的 userdebug 版本的签名构建

我们直接根据 [LineageOS 的教程](https://wiki.lineageos.org/signing_builds#generating-an-install-package)操作即可。

现在，我们可以在 `~/android/lineage` 中找到 `signed-ota_update.zip`，这就是我们自签名的刷机包。现在就可以试试这个包，也可以直接继续接下来的步骤。

### 6. 修改 LineageOS 中的代码

本节提到的目录和文件除使用绝对路径或另有说明外，均基于 LineageOS 根目录，即 `~/android/lineage`。

修改任何代码之前，先创建我们需要的工作目录：
```bash
mkdir ~/android/enchilada
mkdir ~/android/enchilada/patches
mkdir ~/android/enchilada/pkmd
```

#### (1) 修改 enchilada 的 `BroadConfig.mk`

LineageOS 22.2 构建时 Soong 默认不信任工作目录之外的文件，这就导致我们的签名密钥无法直接使用，所以我们把密钥目录挂载到工作目录内：
```bash
mkdir ~/android/lineage/device/oneplus/enchilada/android-keys
mount --bind ~/.android-certs ~/android/lineage/device/oneplus/enchilada/android-keys
```

在 `device/oneplus/enchilada/BroadConfig.mk` 文件中，添加以下行（来自[此处](https://github.com/Wunderment/build_devices/blob/master/enchilada_20/build/board-config-additions.txt)）：

```
# Enable RADIO files so we can add the firmware IMGs to the OTA.
ADD_RADIO_FILES := true

# Set the AVB key and hash algorithm.
BOARD_AVB_KEY_PATH := $(DEVICE_PATH)/android-keys/releasekey.key
BOARD_AVB_ALGORITHM := SHA256_RSA2048

# Include the rest of the prebuilt partitions.
# The following three images are exclude as lineage recovery doesn't seem to be able to flash them: india.img, reserve.img
AB_OTA_PARTITIONS += abl aop bluetooth cmnlib cmnlib64 devcfg dsp fw_4j1ed fw_4u1ea hyp keymaster LOGO modem oem_stanvbk qupfw storsec tz xbl xbl_config
```

上述修改内容中提到的 `$(DEVICE_PATH)/android-keys` 代表挂载之后的密钥目录，任何在 `~/.android-certs` 进行的修改都会同步到此目录中。每次重新启动系统时都需要挂载一次。

#### (2) 修改 sdm845-common 的 `BoardConfigCommon.mk`（可选）

修改这个的作用是启用 LineageOS 默认禁用的分区验证，我们已经集成了 KernelSU 和 GApps，所以我们可以开启分区验证。

在 `device/oneplus/sdm845-common/BoardConfigCommon.mk` 文件中，找到并注释以下行：
```
#BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS += --set_hashtree_disabled_flag
#BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS += --set_verification_disabled_flag
```

原始文件没有前导 `#` 号，我们加上 `#` 号代表注释这两行。

#### (3) 修改 AOSP 的 Makefile

下载[这个文件](https://github.com/Wunderment/build_tasks/blob/master/source/core_Makefile-22.2.patch)到 `~/android/enchilada/patches` 中。

前往 `build/core` 目录应用这个文件：
```bash
cd ~/android/lineage/build/core
patch Makefile ~/android/enchilada/patches/core_Makefile-22.2.patch
```

#### (4) 将 vendor 镜像加入签名列表

提取 OxygenOS 11 的最新版本的 payload.bin 文件并解压出 img 镜像，找到 `oem_stanvbk.img` 文件，将其复制到 `vendor/oneplus/enchilada/radio` 目录。

找到 `vendor/oneplus/enchilada/Android.mk` 文件，将以下内容添加到 `endif` 行之前，其他行之后（请无视文件顶部的`DO NOT MODIFY`，没什么好担心的）：
```
$(call add-radio-file,radio/oem_stanvbk.img)
```

### 7. 安装 KernelSU-Next（可选）

这一步不是必须的，但如果想在锁定 Bootloader 后使用 root 权限，则此时安装 KernelSU-Next 是最好的选择。

前往内核目录：
```bash
cd ~/android/lineage/kernel/oneplus/sdm845
```

安装 KernelSU-Next 最新发行版：

```bash
curl -LSs "https://raw.githubusercontent.com/rifsxd/KernelSU-Next/next/kernel/setup.sh" | bash -
```

### 8. 构建 LineageOS 的 user 版本

要构建 user 版本而不是 userdebug 版本，首先需要修改默认的构建配置。

构建配置默认存放在 Lineage 根目录中的 build 文件夹内，我们首先复制一份默认配置：
```bash
cd ~/android/lineage/build
cp buildspec.mk.default buildspec.mk
```

之后，修改 `buildspec.mk` 文件，找到 `TARGET_BUILD_VARIANT:=user` 删除前面的 `#` 号。

现在，按照正常流程构建 LineageOS：
```bassh
cd ~/android/lineage
source build/envsetup.sh
breakfast enchilada user
croot
mka target-files-package otatools
```

构建完成后，签名：
```bash
croot
sign_target_files_apks -o -d ~/.android-certs \
    --extra_apks AdServicesApk.apk=$HOME/.android-certs/releasekey \
    --extra_apks FederatedCompute.apk=$HOME/.android-certs/releasekey \
    --extra_apks HalfSheetUX.apk=$HOME/.android-certs/releasekey \
    --extra_apks HealthConnectBackupRestore.apk=$HOME/.android-certs/releasekey \
    --extra_apks HealthConnectController.apk=$HOME/.android-certs/releasekey \
    --extra_apks OsuLogin.apk=$HOME/.android-certs/releasekey \
    --extra_apks SafetyCenterResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceConnectivityResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceUwbResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceWifiResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks WifiDialog.apk=$HOME/.android-certs/releasekey \
    --extra_apks com.android.adbd.apex=$HOME/.android-certs/com.android.adbd \
    --extra_apks com.android.adservices.apex=$HOME/.android-certs/com.android.adservices \
    --extra_apks com.android.adservices.api.apex=$HOME/.android-certs/com.android.adservices.api \
    --extra_apks com.android.appsearch.apex=$HOME/.android-certs/com.android.appsearch \
    --extra_apks com.android.appsearch.apk.apex=$HOME/.android-certs/com.android.appsearch.apk \
    --extra_apks com.android.art.apex=$HOME/.android-certs/com.android.art \
    --extra_apks com.android.bluetooth.apex=$HOME/.android-certs/com.android.bluetooth \
    --extra_apks com.android.btservices.apex=$HOME/.android-certs/com.android.btservices \
    --extra_apks com.android.cellbroadcast.apex=$HOME/.android-certs/com.android.cellbroadcast \
    --extra_apks com.android.compos.apex=$HOME/.android-certs/com.android.compos \
    --extra_apks com.android.configinfrastructure.apex=$HOME/.android-certs/com.android.configinfrastructure \
    --extra_apks com.android.connectivity.resources.apex=$HOME/.android-certs/com.android.connectivity.resources \
    --extra_apks com.android.conscrypt.apex=$HOME/.android-certs/com.android.conscrypt \
    --extra_apks com.android.devicelock.apex=$HOME/.android-certs/com.android.devicelock \
    --extra_apks com.android.extservices.apex=$HOME/.android-certs/com.android.extservices \
    --extra_apks com.android.graphics.pdf.apex=$HOME/.android-certs/com.android.graphics.pdf \
    --extra_apks com.android.hardware.authsecret.apex=$HOME/.android-certs/com.android.hardware.authsecret \
    --extra_apks com.android.hardware.biometrics.face.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.face.virtual \
    --extra_apks com.android.hardware.biometrics.fingerprint.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.fingerprint.virtual \
    --extra_apks com.android.hardware.boot.apex=$HOME/.android-certs/com.android.hardware.boot \
    --extra_apks com.android.hardware.cas.apex=$HOME/.android-certs/com.android.hardware.cas \
    --extra_apks com.android.hardware.neuralnetworks.apex=$HOME/.android-certs/com.android.hardware.neuralnetworks \
    --extra_apks com.android.hardware.rebootescrow.apex=$HOME/.android-certs/com.android.hardware.rebootescrow \
    --extra_apks com.android.hardware.wifi.apex=$HOME/.android-certs/com.android.hardware.wifi \
    --extra_apks com.android.healthfitness.apex=$HOME/.android-certs/com.android.healthfitness \
    --extra_apks com.android.hotspot2.osulogin.apex=$HOME/.android-certs/com.android.hotspot2.osulogin \
    --extra_apks com.android.i18n.apex=$HOME/.android-certs/com.android.i18n \
    --extra_apks com.android.ipsec.apex=$HOME/.android-certs/com.android.ipsec \
    --extra_apks com.android.media.apex=$HOME/.android-certs/com.android.media \
    --extra_apks com.android.media.swcodec.apex=$HOME/.android-certs/com.android.media.swcodec \
    --extra_apks com.android.mediaprovider.apex=$HOME/.android-certs/com.android.mediaprovider \
    --extra_apks com.android.nearby.halfsheet.apex=$HOME/.android-certs/com.android.nearby.halfsheet \
    --extra_apks com.android.networkstack.tethering.apex=$HOME/.android-certs/com.android.networkstack.tethering \
    --extra_apks com.android.neuralnetworks.apex=$HOME/.android-certs/com.android.neuralnetworks \
    --extra_apks com.android.nfcservices.apex=$HOME/.android-certs/com.android.nfcservices \
    --extra_apks com.android.ondevicepersonalization.apex=$HOME/.android-certs/com.android.ondevicepersonalization \
    --extra_apks com.android.os.statsd.apex=$HOME/.android-certs/com.android.os.statsd \
    --extra_apks com.android.permission.apex=$HOME/.android-certs/com.android.permission \
    --extra_apks com.android.profiling.apex=$HOME/.android-certs/com.android.profiling \
    --extra_apks com.android.resolv.apex=$HOME/.android-certs/com.android.resolv \
    --extra_apks com.android.rkpd.apex=$HOME/.android-certs/com.android.rkpd \
    --extra_apks com.android.runtime.apex=$HOME/.android-certs/com.android.runtime \
    --extra_apks com.android.safetycenter.resources.apex=$HOME/.android-certs/com.android.safetycenter.resources \
    --extra_apks com.android.scheduling.apex=$HOME/.android-certs/com.android.scheduling \
    --extra_apks com.android.sdkext.apex=$HOME/.android-certs/com.android.sdkext \
    --extra_apks com.android.support.apexer.apex=$HOME/.android-certs/com.android.support.apexer \
    --extra_apks com.android.telephony.apex=$HOME/.android-certs/com.android.telephony \
    --extra_apks com.android.telephonymodules.apex=$HOME/.android-certs/com.android.telephonymodules \
    --extra_apks com.android.tethering.apex=$HOME/.android-certs/com.android.tethering \
    --extra_apks com.android.tzdata.apex=$HOME/.android-certs/com.android.tzdata \
    --extra_apks com.android.uwb.apex=$HOME/.android-certs/com.android.uwb \
    --extra_apks com.android.uwb.resources.apex=$HOME/.android-certs/com.android.uwb.resources \
    --extra_apks com.android.virt.apex=$HOME/.android-certs/com.android.virt \
    --extra_apks com.android.vndk.current.apex=$HOME/.android-certs/com.android.vndk.current \
    --extra_apks com.android.vndk.current.on_vendor.apex=$HOME/.android-certs/com.android.vndk.current.on_vendor \
    --extra_apks com.android.wifi.apex=$HOME/.android-certs/com.android.wifi \
    --extra_apks com.android.wifi.dialog.apex=$HOME/.android-certs/com.android.wifi.dialog \
    --extra_apks com.android.wifi.resources.apex=$HOME/.android-certs/com.android.wifi.resources \
    --extra_apks com.google.pixel.camera.hal.apex=$HOME/.android-certs/com.google.pixel.camera.hal \
    --extra_apks com.google.pixel.vibrator.hal.apex=$HOME/.android-certs/com.google.pixel.vibrator.hal \
    --extra_apks com.qorvo.uwb.apex=$HOME/.android-certs/com.qorvo.uwb \
    --extra_apex_payload_key com.android.adbd.apex=$HOME/.android-certs/com.android.adbd.pem \
    --extra_apex_payload_key com.android.adservices.apex=$HOME/.android-certs/com.android.adservices.pem \
    --extra_apex_payload_key com.android.adservices.api.apex=$HOME/.android-certs/com.android.adservices.api.pem \
    --extra_apex_payload_key com.android.appsearch.apex=$HOME/.android-certs/com.android.appsearch.pem \
    --extra_apex_payload_key com.android.appsearch.apk.apex=$HOME/.android-certs/com.android.appsearch.apk.pem \
    --extra_apex_payload_key com.android.art.apex=$HOME/.android-certs/com.android.art.pem \
    --extra_apex_payload_key com.android.bluetooth.apex=$HOME/.android-certs/com.android.bluetooth.pem \
    --extra_apex_payload_key com.android.btservices.apex=$HOME/.android-certs/com.android.btservices.pem \
    --extra_apex_payload_key com.android.cellbroadcast.apex=$HOME/.android-certs/com.android.cellbroadcast.pem \
    --extra_apex_payload_key com.android.compos.apex=$HOME/.android-certs/com.android.compos.pem \
    --extra_apex_payload_key com.android.configinfrastructure.apex=$HOME/.android-certs/com.android.configinfrastructure.pem \
    --extra_apex_payload_key com.android.connectivity.resources.apex=$HOME/.android-certs/com.android.connectivity.resources.pem \
    --extra_apex_payload_key com.android.conscrypt.apex=$HOME/.android-certs/com.android.conscrypt.pem \
    --extra_apex_payload_key com.android.devicelock.apex=$HOME/.android-certs/com.android.devicelock.pem \
    --extra_apex_payload_key com.android.extservices.apex=$HOME/.android-certs/com.android.extservices.pem \
    --extra_apex_payload_key com.android.graphics.pdf.apex=$HOME/.android-certs/com.android.graphics.pdf.pem \
    --extra_apex_payload_key com.android.hardware.authsecret.apex=$HOME/.android-certs/com.android.hardware.authsecret.pem \
    --extra_apex_payload_key com.android.hardware.biometrics.face.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.face.virtual.pem \
    --extra_apex_payload_key com.android.hardware.biometrics.fingerprint.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.fingerprint.virtual.pem \
    --extra_apex_payload_key com.android.hardware.boot.apex=$HOME/.android-certs/com.android.hardware.boot.pem \
    --extra_apex_payload_key com.android.hardware.cas.apex=$HOME/.android-certs/com.android.hardware.cas.pem \
    --extra_apex_payload_key com.android.hardware.neuralnetworks.apex=$HOME/.android-certs/com.android.hardware.neuralnetworks.pem \
    --extra_apex_payload_key com.android.hardware.rebootescrow.apex=$HOME/.android-certs/com.android.hardware.rebootescrow.pem \
    --extra_apex_payload_key com.android.hardware.wifi.apex=$HOME/.android-certs/com.android.hardware.wifi.pem \
    --extra_apex_payload_key com.android.healthfitness.apex=$HOME/.android-certs/com.android.healthfitness.pem \
    --extra_apex_payload_key com.android.hotspot2.osulogin.apex=$HOME/.android-certs/com.android.hotspot2.osulogin.pem \
    --extra_apex_payload_key com.android.i18n.apex=$HOME/.android-certs/com.android.i18n.pem \
    --extra_apex_payload_key com.android.ipsec.apex=$HOME/.android-certs/com.android.ipsec.pem \
    --extra_apex_payload_key com.android.media.apex=$HOME/.android-certs/com.android.media.pem \
    --extra_apex_payload_key com.android.media.swcodec.apex=$HOME/.android-certs/com.android.media.swcodec.pem \
    --extra_apex_payload_key com.android.mediaprovider.apex=$HOME/.android-certs/com.android.mediaprovider.pem \
    --extra_apex_payload_key com.android.nearby.halfsheet.apex=$HOME/.android-certs/com.android.nearby.halfsheet.pem \
    --extra_apex_payload_key com.android.networkstack.tethering.apex=$HOME/.android-certs/com.android.networkstack.tethering.pem \
    --extra_apex_payload_key com.android.neuralnetworks.apex=$HOME/.android-certs/com.android.neuralnetworks.pem \
    --extra_apex_payload_key com.android.nfcservices.apex=$HOME/.android-certs/com.android.nfcservices.pem \
    --extra_apex_payload_key com.android.ondevicepersonalization.apex=$HOME/.android-certs/com.android.ondevicepersonalization.pem \
    --extra_apex_payload_key com.android.os.statsd.apex=$HOME/.android-certs/com.android.os.statsd.pem \
    --extra_apex_payload_key com.android.permission.apex=$HOME/.android-certs/com.android.permission.pem \
    --extra_apex_payload_key com.android.profiling.apex=$HOME/.android-certs/com.android.profiling.pem \
    --extra_apex_payload_key com.android.resolv.apex=$HOME/.android-certs/com.android.resolv.pem \
    --extra_apex_payload_key com.android.rkpd.apex=$HOME/.android-certs/com.android.rkpd.pem \
    --extra_apex_payload_key com.android.runtime.apex=$HOME/.android-certs/com.android.runtime.pem \
    --extra_apex_payload_key com.android.safetycenter.resources.apex=$HOME/.android-certs/com.android.safetycenter.resources.pem \
    --extra_apex_payload_key com.android.scheduling.apex=$HOME/.android-certs/com.android.scheduling.pem \
    --extra_apex_payload_key com.android.sdkext.apex=$HOME/.android-certs/com.android.sdkext.pem \
    --extra_apex_payload_key com.android.support.apexer.apex=$HOME/.android-certs/com.android.support.apexer.pem \
    --extra_apex_payload_key com.android.telephony.apex=$HOME/.android-certs/com.android.telephony.pem \
    --extra_apex_payload_key com.android.telephonymodules.apex=$HOME/.android-certs/com.android.telephonymodules.pem \
    --extra_apex_payload_key com.android.tethering.apex=$HOME/.android-certs/com.android.tethering.pem \
    --extra_apex_payload_key com.android.tzdata.apex=$HOME/.android-certs/com.android.tzdata.pem \
    --extra_apex_payload_key com.android.uwb.apex=$HOME/.android-certs/com.android.uwb.pem \
    --extra_apex_payload_key com.android.uwb.resources.apex=$HOME/.android-certs/com.android.uwb.resources.pem \
    --extra_apex_payload_key com.android.virt.apex=$HOME/.android-certs/com.android.virt.pem \
    --extra_apex_payload_key com.android.vndk.current.apex=$HOME/.android-certs/com.android.vndk.current.pem \
    --extra_apex_payload_key com.android.vndk.current.on_vendor.apex=$HOME/.android-certs/com.android.vndk.current.on_vendor.pem \
    --extra_apex_payload_key com.android.wifi.apex=$HOME/.android-certs/com.android.wifi.pem \
    --extra_apex_payload_key com.android.wifi.dialog.apex=$HOME/.android-certs/com.android.wifi.dialog.pem \
    --extra_apex_payload_key com.android.wifi.resources.apex=$HOME/.android-certs/com.android.wifi.resources.pem \
    --extra_apex_payload_key com.google.pixel.camera.hal.apex=$HOME/.android-certs/com.google.pixel.camera.hal.pem \
    --extra_apex_payload_key com.google.pixel.vibrator.hal.apex=$HOME/.android-certs/com.google.pixel.vibrator.hal.pem \
    --extra_apex_payload_key com.qorvo.uwb.apex=$HOME/.android-certs/com.qorvo.uwb.pem \
    $OUT/obj/PACKAGING/target_files_intermediates/*-target_files*.zip \
    signed-target_files.zip
```

签名完成后，打包：
```bash
ota_from_target_files -k ~/.android-certs/releasekey \
    --block --backup=true \
    signed-target_files.zip \
    signed-ota_update.zip
```

现在，就可以将 `signed-ota_update.zip` 刷入手机，任意方法均可使用。（TWRP 永远的神）

刷入完成后，最好先启动一次系统。如果哪里出现了问题，而未进入系统就直接锁定 Bootloader，会导致手机再也无法启动到系统，也无法解锁，从而变砖，只能通过 9008 线刷救砖。

### 9. 生成并刷入 `avb_custom_key`

生成 `avb_custom_key`：
```bash
avbtool extract_public_key --key ~/.android-certs/releasekey.key --output ~/pkmd.bin
```

可以在用户主目录找到 `pkmd.bin` 文件。现在将手机重启到 Bootloader 模式准备刷入此文件。

如果事先刷过别的 key，使用以下命令清除：
```bash
fastboot erase avb_custom_key
```

之后，使用以下命令刷入我们的 Key：
```bash
fastboot flash avb_custom_key pkmd.bin
```

最后，重启一次 Bootloader，然后锁定 Bootloader：
```bash
fastboot reboot bootloader
fastboot oem lock
```

在手机上确认锁定后，手机会自动重启，经过一次 Lineage Recovery 的 Wipe Data 后，我们就可以开始使用锁定了 Bootloader 并且带有 KernelSU-Next 和 GApps 的 LineageOS 了。

## 已知问题

- 经过测试，集成 KernelSU-Next 后，KSUN 管理器`无法获取 root 权限`，查阅 logcat 并尝试将 SELinux 设为宽容模式后得知为 SELinux 规则问题，且就算设为宽容模式也无法授予 root 给其他软件。
- 经过测试，TEE 仍然不可用，密钥认证演示提示错误代码 `-10003`。此项尚未经过更广泛的测试（即非 LineageOS 是否也是如此）