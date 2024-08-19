# Q3 Explanation 

First of all i clone the kernel repo and download the patchset. When first applied, some hunks are failed.

### [02/12] iwlwifi: yoyo: support for ROM usniffer

````c
[asmolniakova@dev1 kernel-5.15.0-83.92]$ patch -F0 -p1 < ../copy_patch 
patching file drivers/net/wireless/intel/iwlwifi/fw/uefi.h
Reversed (or previously applied) patch detected!  Assume -R? [n] n
Apply anyway? [n] y
Hunk #1 FAILED at 2.
Hunk #2 FAILED at 40.
2 out of 2 hunks FAILED -- saving rejects to file drivers/net/wireless/intel/iwlwifi/fw/uefi.h.rej

CHECK WITHOUT REVERSE
-----------------------------------------------
CHECK WITH REVERSE

[asmolniakova@dev1 kernel-5.15.0-83.92]$ patch -F0 -p1 < ../patchset.patch 
patching file drivers/net/wireless/intel/iwlwifi/fw/uefi.h
Reversed (or previously applied) patch detected!  Assume -R? [n] y
patching file drivers/net/wireless/intel/iwlwifi/fw/api/debug.h
Hunk #1 FAILED at 379.
1 out of 1 hunk FAILED -- saving rejects to file drivers/net/wireless/intel/iwlwifi/fw/api/debug.h.rej
patching file drivers/net/wireless/intel/iwlwifi/iwl-dbg-tlv.c
Hunk #1 FAILED at 284.
Hunk #2 FAILED at 812.
Hunk #3 succeeded at 1075 (offset -154 lines).
Hunk #4 succeeded at 1086 (offset -154 lines).
2 out of 4 hunks FAILED -- saving rejects to file drivers/net/wireless/intel/iwlwifi/iwl-dbg-tlv.c.rej
````

The first thing I see is that first two hunks are already applied. So I delete them from the patch since everything is already fixed. Then I open every .rej file one by one to see what is wrong.

### [03/12] iwlwifi: mvm: read 6E enablement flags from DSM and pass to FW

I see structure but the elements DSM_FUNC_QUERY = 0 and DSM_FUNC_ACTIVATE_CHANNEL = 8 are missing and I find them in this commit 1f578d4f2d52f64133c4b53346451b1116242e3d. I create a patch out of this commit.
```c
enum iwl_dsm_funcs_rev_0 {
        DSM_FUNC_DISABLE_SRD = 1,
        DSM_FUNC_ENABLE_INDONESIA_5G2 = 2,
        DSM_FUNC_11AX_ENABLEMENT = 6,
       DSM_FUNC_ENABLE_UNII4_CHAN = 7
}

KERNEL
----------------------------
PATCH

@@ -109,6 +109,7 @@ enum iwl_dsm_funcs_rev_0 {
 372         DSM_FUNC_QUERY = 0,
 373         DSM_FUNC_DISABLE_SRD = 1,
 374         DSM_FUNC_ENABLE_INDONESIA_5G2 = 2,
 375 +       DSM_FUNC_ENABLE_6E = 3,
 376         DSM_FUNC_11AX_ENABLEMENT = 6,
 377         DSM_FUNC_ENABLE_UNII4_CHAN = 7,
 378         DSM_FUNC_ACTIVATE_CHANNEL = 8
```
> git log -S'DSM_FUNC_ACTIVATE_CHANNEL = 8' --source --all

> git format-patch --filename-max-length=128 -n -o ~/my 1f578d4f2d52^..1f578d4f2d52

### [05/12] iwlwifi: mvm: update RFI TLV

Looking at this hunk and I see that "IWL_UCODE_TLV_CAPA_COEX_HIGH_PRIO = (__force iwl_ucode_tlv_capa_t)61," is absent. I find it in this commit 8b75858c2e218570a61bb7ee21b17bdd19c49485 but 2 hunks are failing because of extra tabs. So I manualy remove extra tabs from the dependency patch, it applies and the patch from original patchset now does too.
```c

IWL_UCODE_TLV_CAPA_FW_RESET_HANDSHAKE           = (__force iwl_ucode_tlv_capa_t)57,
        IWL_UCODE_TLV_CAPA_PASSIVE_6GHZ_SCAN            = (__force iwl_ucode_tlv_capa_t)58,
        IWL_UCODE_TLV_CAPA_HIDDEN_6GHZ_SCAN             = (__force iwl_ucode_tlv_capa_t)59,
        IWL_UCODE_TLV_CAPA_BROADCAST_TWT                = (__force iwl_ucode_tlv_capa_t)60,

        /* set 2 */
        IWL_UCODE_TLV_CAPA_EXTENDED_DTS_MEASURE         = (__force iwl_ucode_tlv_capa_t)64,
        IWL_UCODE_TLV_CAPA_SHORT_PM_TIMEOUTS            = (__force iwl_ucode_tlv_capa_t)65,

 KERNEL
 ----------------------------------
 PATCH

  --- drivers/net/wireless/intel/iwlwifi/fw/file.h
  2 +++ drivers/net/wireless/intel/iwlwifi/fw/file.h
  3 @@ -421,6 +421,7 @@ enum iwl_ucode_tlv_capa {
  4         IWL_UCODE_TLV_CAPA_HIDDEN_6GHZ_SCAN             = (__force iwl_ucode_tlv_ca    pa_t)59,
  5         IWL_UCODE_TLV_CAPA_BROADCAST_TWT                = (__force iwl_ucode_tlv_ca    pa_t)60,
  6         IWL_UCODE_TLV_CAPA_COEX_HIGH_PRIO               = (__force iwl_ucode_tlv_ca    pa_t)61,
  7 +       IWL_UCODE_TLV_CAPA_RFIM_SUPPORT                 = (__force iwl_ucode_tlv_ca    pa_t)62,
  8 
  9         /* set 2 */
 10         IWL_UCODE_TLV_CAPA_EXTENDED_DTS_MEASURE         = (__force iwl_ucode_tlv_ca    pa_t)64,
```

### [07/12] iwlwifi: rename GEO_TX_POWER_LIMIT to PER_CHAIN_LIMIT_OFFSET_CMD
I want to apply this change but in older kernel there is a structure in between and a little difference in a line being deleted. So I look up the commit where the structure was added and find it in 97f8a3d1610b1f50013de1e632b007cbfd7459d8 .
````c
struct iwl_geo_tx_power_profiles_cmd_v3 {
        __le32 ops;
        struct iwl_per_chain_offset table[IWL_NUM_GEO_PROFILES][IWL_NUM_BANDS_PER_CHAIN_V2];
        __le32 table_revision;
} __packed; /* GEO_TX_POWER_LIMIT_VER_3 */

union iwl_geo_tx_power_profiles_cmd {
        struct iwl_geo_tx_power_profiles_cmd_v1 v1; 
        struct iwl_geo_tx_power_profiles_cmd_v2 v2; 
        struct iwl_geo_tx_power_profiles_cmd_v3 v3; 
};

/**
 * struct iwl_geo_tx_power_profiles_resp -  response to GEO_TX_POWER_LIMIT cmd
 * @profile_idx: current geo profile in use
 */

KERNEL
--------------------
PATCH

@@ -437,10 +437,10 @@ struct iwl_geo_tx_power_profiles_cmd_v3 {
        __le32 ops;
        struct iwl_per_chain_offset table[IWL_NUM_GEO_PROFILES][IWL_NUM_BANDS_PER_CHAIN_V2];
        __le32 table_revision;
-} __packed; /* GEO_TX_POWER_LIMIT_VER_3 */
+} __packed; /* PER_CHAIN_LIMIT_OFFSET_CMD_VER_3 */

 /**
- * struct iwl_geo_tx_power_profile_cmd_v4 - struct for GEO_TX_POWER_LIMIT cmd.
+ * struct iwl_geo_tx_power_profile_cmd_v4 - struct for PER_CHAIN_LIMIT_OFFSET_CMD cmd.
  * @ops: operations, value from &enum iwl_geo_per_chain_offset_operation
  * @table: offset profile per band.
  * @table_revision: BIOS table revision.
  ````

### Making the final patch

I restore all the files in the repo and apply dependency pathches that I made one by one. Then apply the original patch. Then I use git diff to create a solid pathc with all the changes

> git diff > final.patch

### Some troubles

When I try to build this kernel I get such an error. My patches don't alter this variable, though. I didn't have enough time to investigate this error but I guess it should be something with naming.
```c
"drivers/net/wireless/intel/iwlwifi/iwl-dbg-tlv.s: FATAL: Variables dbg_tlv_alloc (line 3430) and       dbg_tlv_alloc (line 7072) don't match^M"
```