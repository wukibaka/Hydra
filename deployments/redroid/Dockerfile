FROM redroid/redroid:14.0.0-latest

# ============================================
# 1. GApps (MindTheGapps for Android 14 x86_64)
# ============================================
COPY gapps/system/ /system/

# ============================================
# 2. libndk_translation (ARM -> x86 转译)
# ============================================
ADD ndk_translation.tar /

# ============================================
# 3. Mihomo 代理内核 (android-amd64)
# ============================================
COPY --chmod=755 mihomo /system/bin/mihomo
COPY --chmod=644 mihomo.rc /system/etc/init/mihomo.rc
COPY --chmod=644 mihomo-config.yaml /system/etc/mihomo/config.yaml

# ============================================
# 4. 预装应用 (split APK — 所有 .apk 放同一目录)
# ============================================
COPY WhatsApp/ /system/app/WhatsApp/
COPY TikTok/ /system/app/TikTok/
