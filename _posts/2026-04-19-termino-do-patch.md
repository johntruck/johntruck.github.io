---
title: Patch finished
date: 2026-04-19
categories: [Patch (Kernel), Patch Development]
tags: [patch, kernel]     # TAG names should always be lowercase
---

# Finding which file to make changes to:

The process of finding a file to make changes to was quite simple: I looked at the directories the other students were working in and did some greps. For example:

```bash
grep "<<" *.c
```

After doing that for about an hour we (me and my duo) found that /sound/soc/atmel/mchp-spdifrx.c was a good target for replacing bitfield manipulations with macros. Because of these two spots:

* Sidenote: FIELD_GET ~= ((register & mask) >> shift) and FIELD_PREP ~= ((register << shift) & mask) (there are other things like safety measures, but these are good simplifications). To learn more about FIELD_GET() and FIELD_PREP() check [section 2.1](https://flusp.ime.usp.br/courses/linux-kernel-patch-suggestions/).

```text
-#define SPDIFRX_RSR_IFS(reg)                   \
-       (((reg) & SPDIFRX_RSR_IFS_MASK) >> 16)

-#define SPDIFRX_VERSION_MFN(reg)       (((reg) & SPDIFRX_VERSION_MFN_MASK) >> 16)
```

Which is basically what FIELD_GET() was meant to replace.

Also the following lines are the type of bit manipulation FIELD_PREP() tries to replace.

```
-#define SPDIFRX_MR_DATAWIDTH(width) \
-      (((6 - (width) / 4) << 4) & SPDIFRX_MR_DATAWIDTH_MASK)
```

Even though these operations are basically the definitions of FIELD_GET() and FIELD_PREP(), I wonder if the fact that these operations are being made on "#define" lines may pose a problem (I checked and there are some FIELD_GET() and FIELD_PREP() on "#define" lines of other files in the kernel, but I am not sure if the maintainers of these modules specifically are also willing to use that type of practice).

# A typo!

Surprisingly I found a typo.

```
-#define SPDIFRX_MR_VBMODE_MASK         GENAMSK(1, 1)
+#define SPDIFRX_MR_VBMODE_MASK         GENMASK(1, 1)
```

As you can see, someone wrote GENAMSK instead of GENMASK. I even checked if the GENAMSK macro exists and it doesn't. So I am pretty sure it's a typo.

# Controversial change:

```
+
+/* 32-bit word byte masks */
+#define SPDIFRX_BYTE_0_MASK         GENMASK(7, 0)
+#define SPDIFRX_BYTE_1_MASK         GENMASK(15, 8)
+#define SPDIFRX_BYTE_2_MASK         GENMASK(23, 16)
+#define SPDIFRX_BYTE_3_MASK         GENMASK(31, 24)
+

-               *user_data++ = val & 0xFF;
-               *user_data++ = (val >> 8) & 0xFF;
-               *user_data++ = (val >> 16) & 0xFF;
-               *user_data++ = (val >> 24) & 0xFF;
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_0_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_1_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_2_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_3_MASK, val);

-               *ch_stat++ = val & 0xFF;
-               *ch_stat++ = (val >> 8) & 0xFF;
-               *ch_stat++ = (val >> 16) & 0xFF;
-               *ch_stat++ = (val >> 24) & 0xFF;
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_0_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_1_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_2_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_3_MASK, val);
```

In order to replace the user_data and ch_stat lines we had to define some byte masks, which I am really not sure if follows the code practices of the maintainers. So I think this may cause some problems.

# Some questions before sending the patch:
I still have some questions to ask David (the monitor), so I will wait to ask him before we actually send the patch to the maintainers.

# Full patch:

```text
---
 sound/soc/atmel/mchp-spdifrx.c | 33 ++++++++++++++++++++-------------
 1 file changed, 20 insertions(+), 13 deletions(-)

diff --git a/sound/soc/atmel/mchp-spdifrx.c b/sound/soc/atmel/mchp-spdifrx.c
index 521bee499..2c47aabdc 100644
--- a/sound/soc/atmel/mchp-spdifrx.c
+++ b/sound/soc/atmel/mchp-spdifrx.c
@@ -6,6 +6,7 @@
 //
 // Author: Codrin Ciubotariu <codrin.ciubotariu@microchip.com>

+#include <linux/bitfield.h>
 #include <linux/clk.h>
 #include <linux/io.h>
 #include <linux/module.h>
@@ -41,6 +42,13 @@

 #define SPDIFRX_VERSION                        0xFC    /* Version Register */

+
+/* 32-bit word byte masks */
+#define SPDIFRX_BYTE_0_MASK         GENMASK(7, 0)
+#define SPDIFRX_BYTE_1_MASK         GENMASK(15, 8)
+#define SPDIFRX_BYTE_2_MASK         GENMASK(23, 16)
+#define SPDIFRX_BYTE_3_MASK         GENMASK(31, 24)
+
 /*
  * ---- Control Register (Write-only) ----
  */
@@ -55,7 +63,7 @@
 #define SPDIFRX_MR_RXEN_ENABLE         (1 << 0)        /* SPDIF Receiver Enabled */

 /* Validity Bit Mode */
-#define SPDIFRX_MR_VBMODE_MASK         GENAMSK(1, 1)
+#define SPDIFRX_MR_VBMODE_MASK         GENMASK(1, 1)
 #define SPDIFRX_MR_VBMODE_ALWAYS_LOAD \
        (0 << 1)        /* Load sample regardless of validity bit value */
 #define SPDIFRX_MR_VBMODE_DISCARD_IF_VB1 \
@@ -74,7 +82,7 @@
 /* Sample Data Width */
 #define SPDIFRX_MR_DATAWIDTH_MASK      GENMASK(5, 4)
 #define SPDIFRX_MR_DATAWIDTH(width) \
-       (((6 - (width) / 4) << 4) & SPDIFRX_MR_DATAWIDTH_MASK)
+       FIELD_PREP(SPDIFRX_MR_DATAWIDTH_MASK, 6 - ((width) / 4))

 /* Packed Data Mode in Receive Holding Register */
 #define SPDIFRX_MR_PACK_MASK           GENMASK(7, 7)
@@ -118,15 +126,14 @@
 #define SPDIFRX_RSR_LOWF                       BIT(2)
 #define SPDIFRX_RSR_NOSIGNAL                   BIT(3)
 #define SPDIFRX_RSR_IFS_MASK                   GENMASK(27, 16)
-#define SPDIFRX_RSR_IFS(reg)                   \
-       (((reg) & SPDIFRX_RSR_IFS_MASK) >> 16)
+#define SPDIFRX_RSR_IFS(reg)                   FIELD_GET(SPDIFRX_RSR_IFS_MASK, reg)

 /*
  *  ---- Version Register (Read-only) ----
  */
 #define SPDIFRX_VERSION_MASK           GENMASK(11, 0)
 #define SPDIFRX_VERSION_MFN_MASK       GENMASK(18, 16)
-#define SPDIFRX_VERSION_MFN(reg)       (((reg) & SPDIFRX_VERSION_MFN_MASK) >> 16)
+#define SPDIFRX_VERSION_MFN(reg)       FIELD_GET(SPDIFRX_VERSION_MFN_MASK, reg)

 static bool mchp_spdifrx_readable_reg(struct device *dev, unsigned int reg)
 {
@@ -317,10 +324,10 @@ static void mchp_spdifrx_channel_status_read(struct mchp_spdifrx_dev *dev,

        for (i = 0; i < ARRAY_SIZE(ctrl->ch_stat[channel].data) / 4; i++) {
                regmap_read(dev->regmap, SPDIFRX_CHSR(channel, i), &val);
-               *ch_stat++ = val & 0xFF;
-               *ch_stat++ = (val >> 8) & 0xFF;
-               *ch_stat++ = (val >> 16) & 0xFF;
-               *ch_stat++ = (val >> 24) & 0xFF;
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_0_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_1_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_2_MASK, val);
+               *ch_stat++ = FIELD_GET(SPDIFRX_BYTE_3_MASK, val);
        }
 }

@@ -334,10 +341,10 @@ static void mchp_spdifrx_channel_user_data_read(struct mchp_spdifrx_dev *dev,

        for (i = 0; i < ARRAY_SIZE(ctrl->user_data[channel].data) / 4; i++) {
                regmap_read(dev->regmap, SPDIFRX_CHUD(channel, i), &val);
-               *user_data++ = val & 0xFF;
-               *user_data++ = (val >> 8) & 0xFF;
-               *user_data++ = (val >> 16) & 0xFF;
-               *user_data++ = (val >> 24) & 0xFF;
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_0_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_1_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_2_MASK, val);
+               *user_data++ = FIELD_GET(SPDIFRX_BYTE_3_MASK, val);
        }
 }

-- 
2.43.0
```