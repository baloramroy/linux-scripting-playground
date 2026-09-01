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


## Component list for USSD:

```python

COMPONENTS = [
    {"name": "cms", "base_dir": Path("/home/cms/bin"), "service_prefix": ""},
    {"name": "npsb_recon", "base_dir": Path("/home/npsb_recon/bin"), "service_prefix": ""},
    # NOTE: "davs", "dfs", "extch", "bkofc-summary" mutli instance are NOT here --
    # davs/dfs/portal_davs/portal_dfs are multi-instance (use patch_apm.py),
    # extch and bkofc-summary were already added to patch_apm.py earlier.
    # Add any further single-instance components below.
]

```