# Building skeleton sites from scratch

You can create a site with no parent and entirely populate its VFS from disk:

```bash
cd /path/to/source/repository
echo "Emergence_FS::importTree('.', '/', ['exclude' => ['#^/[^/]+\$#', '#^/\.git(/|\$)#'] ]);" | sudo emergence-shell my-site-handle
```

The two exclude filters prevent bare root files (which the VFS does not support) and the `.git` tree from being imported.