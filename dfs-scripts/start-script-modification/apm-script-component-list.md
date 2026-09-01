## Component list for App Core

```python

COMPONENTS = [
    {"name": "cms", "base_dir": Path("/home/cms/bin"), "service_prefix": ""},
    {"name": "cp", "base_dir": Path("/home/cp/bin"), "service_prefix": ""},
    {"name": "cs", "base_dir": Path("/home/cs/bin"), "service_prefix": ""},
    {"name": "davs", "base_dir": Path("/home/davs/bin"), "service_prefix": ""},
    {"name": "dfs", "base_dir": Path("/home/dfs/bin"), "service_prefix": ""},
    {"name": "drs", "base_dir": Path("/home/drs/bin"), "service_prefix": ""},
    {"name": "dmscore", "base_dir": Path("/home/dmscore/bin"), "service_prefix": ""},
    {"name": "extch", "base_dir": Path("/home/extch/bin"), "service_prefix": ""},
    {"name": "extchSMS", "base_dir": Path("/home/extchSMS/bin"), "service_prefix": ""},
    {"name": "ias", "base_dir": Path("/home/ias/bin"), "service_prefix": ""},
    #{"name": "kms", "base_dir": Path("/home/kms/bin"), "service_prefix": ""},
    {"name": "knotify", "base_dir": Path("/home/knotify/bin"), "service_prefix": ""},
    {"name": "kod", "base_dir": Path("/home/kod/bin"), "service_prefix": ""},
    {"name": "map", "base_dir": Path("/home/map/bin"), "service_prefix": ""},
    {"name": "pcs", "base_dir": Path("/home/pcs/bin"), "service_prefix": ""},
    {"name": "spg", "base_dir": Path("/home/spg/bin"), "service_prefix": ""},
    {"name": "tms", "base_dir": Path("/home/tms/bin"), "service_prefix": ""},
    {"name": "tsp", "base_dir": Path("/home/tsp/bin"), "service_prefix": ""},
    {"name": "bds", "base_dir": Path("/home/bds/bin"), "service_prefix": ""},
    {"name": "ecs", "base_dir": Path("/home/ecs/bin"), "service_prefix": ""},
    {"name": "rms", "base_dir": Path("/home/rms/bin"), "service_prefix": ""},
    {"name": "rpg", "base_dir": Path("/home/rpg/bin"), "service_prefix": ""},
    {"name": "mps", "base_dir": Path("/home/mps/bin"), "service_prefix": ""},
    {"name": "bkofc", "base_dir": Path("/home/bkofc/bin"), "service_prefix": ""},
    {"name": "utilityservice", "base_dir": Path("/home/utilityservice/bin"), "service_prefix": ""},
    {"name": "npsb_recon", "base_dir": Path("/home/npsb_recon/bin"), "service_prefix": ""},
    # NOTE: "davs", "dfs", "extch", "bkofc-summary" mutli instance are NOT here --
    # davs/dfs/portal_davs/portal_dfs are multi-instance (use patch_apm.py),
    # extch and bkofc-summary were already added to patch_apm.py earlier.
    # Add any further single-instance components below.
]

```

---


## Component List for Txn-history

```python

COMPONENTS = [
    {
        "name": "davs",
        "base_dir": Path("/home/davs/bin"),
        "instances": ["davs1", "davs2", "davs3"],
        "service_prefix": "",
    },
    {
        "name": "dfs",
        "base_dir": Path("/home/dfs/bin"),
        "instances": ["dfs1", "dfs2", "dfs3"],
        "service_prefix": "",
    },
    {
        "name": "portal_davs",
        "base_dir": Path("/home/portal_davs/bin"),
        "instances": ["davs1", "davs2", "davs3"],
        "service_prefix": "portal-",
    },
    {
        "name": "portal_dfs",
        "base_dir": Path("/home/portal_dfs/bin"),
        "instances": ["dfs1", "dfs2", "dfs3"],
        "service_prefix": "portal-",
    },
    {
        # No instance subfolder -- start file sits directly in bin/
        "name": "extch",
        "base_dir": Path("/home/extch/bin"),
        "instances": [],
        "service_prefix": "",
    },
    {
        "name": "bkofc-summary",
        "base_dir": Path("/home/bkofc-summary/bin"),
        "instances": ["bkofc-summary1", "bkofc-summary2"],
        "service_prefix": "",
    },
]

```

---

## Component list for USSD:

```python

COMPONENTS = [
    {"name": "ussdgwgp", "base_dir": Path("/home/ussdgwgp/bin"), "service_prefix": ""},
    {"name": "ussdgwblink", "base_dir": Path("/home/ussdgwblink/bin"), "service_prefix": ""},
    {"name": "ussdgwrobi", "base_dir": Path("/home/ussdgwrobi/bin"), "service_prefix": ""},
    {"name": "ussdgwttalk", "base_dir": Path("/home/ussdgwttalk/bin"), "service_prefix": ""},
    {"name": "knotifydmz", "base_dir": Path("/home/knotifydmz/bin"), "service_prefix": ""},
    {"name": "outboundproxy", "base_dir": Path("/home/outboundproxy/bin"), "service_prefix": ""},
    # Add any further single-instance components below.
]

```

---

