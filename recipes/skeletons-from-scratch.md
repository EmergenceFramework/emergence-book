# Bootstrap a Site from Git

1. Create a new site with no parent
2. Clone the repository you want to bootstrap from to disk on the same machine hosting the new site:
```bash
git clone https://github.com/JarvusInnovations/emergence-skeleton.git /tmp/emergence-skeleton-v1
```

3. Change your working directory to the newly-cloned repository:
```bash
cd /tmp/emergence-skeleton-v1/
```

4. Import the contents of the repository into your newly created site, excluding root files and the `.git` tree:
```bash
echo "Emergence_FS::importTree('.', '/', ['exclude' => ['#^/[^/]+\$#', '#^/\.git(/|\$)#'] ]);" | sudo emergence-shell my-site-handle
```
*Be sure to substitute `my-site-handle` at the end*

5. Check if the imported site offers a `php-config/Git.config.d` file mapping the source repository into the site tree. It may be wrapped in an `if` condition checking the site handle that must be made to match the current site to activate it. Visit `/git/status` to initialize the repository link to enable pushing/pulling ongoing changes to the git repository