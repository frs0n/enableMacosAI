# 从源码编译 RegionSpoof.kext

仅在**换 macOS 大版本**(prebuilt kext 被拒)、或想自行审计/修改后重编时需要。
前提:装好 Command Line Tools(`xcode-select --install`)。在仓库根目录执行:

```bash
SDK="$(xcrun --show-sdk-path)"
KH="$SDK/System/Library/Frameworks/Kernel.framework/Headers"

# 1) 编译 kext 主体
xcrun clang++ -arch arm64e -isysroot "$SDK" -I"$KH" \
  -DKERNEL -DKERNEL_PRIVATE -DDRIVER_PRIVATE -DAPPLE -DNeXT \
  -fno-builtin -fno-common -fapple-kext -mkernel \
  -std=c++17 -fno-rtti -fno-exceptions -fno-stack-protector -O2 \
  -c src/RegionSpoof.cpp -o /tmp/RegionSpoof.o

# 2) 编译 kmod 入口（提供链接必需的 _kmod_info 符号）
xcrun clang -arch arm64e -isysroot "$SDK" -I"$KH" \
  -DKERNEL -DKERNEL_PRIVATE -DAPPLE \
  -fno-builtin -fno-common -fapple-kext -mkernel -O2 \
  -c src/kmod_info.c -o /tmp/kmod_info.o

# 3) 链接
xcrun clang++ -arch arm64e -isysroot "$SDK" -nostdlib -fapple-kext \
  -Xlinker -kext -lkmod -lkmodc++ -lcc_kext \
  -o /tmp/RegionSpoof /tmp/RegionSpoof.o /tmp/kmod_info.o

# 4) 组装 bundle + ad-hoc 签名 + 改属主
rm -rf RegionSpoof.kext
mkdir -p RegionSpoof.kext/Contents/MacOS
cp src/Info.plist   RegionSpoof.kext/Contents/Info.plist
cp /tmp/RegionSpoof RegionSpoof.kext/Contents/MacOS/RegionSpoof
sudo codesign -f -s - RegionSpoof.kext
sudo chown -R 0:0 RegionSpoof.kext
```

编好后用 `sudo ./install.sh` 安装(安装 / 验证 / 卸载见 [README](README.md))。

> 说明:`-fapple-kext -mkernel` 走 kext 专用 ABI;`-lkmod -lkmodc++ -lcc_kext` + `-Xlinker -kext`
> 是 kext 必需的链接库。`codesign -s -` 是 ad-hoc 签名(因此**必须 Permissive / 完整关 SIP** 才能加载;
> 想在 SIP-on 下用,把 `-s -` 换成你的 Developer-ID,并走 Reduced Security)。
