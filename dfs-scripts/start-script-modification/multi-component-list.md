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