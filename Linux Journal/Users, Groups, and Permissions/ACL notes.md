To download ACL on a Ubuntu/Debian server, use the following commands:
```bash
sudo apt update
sudo apt install acl
```

- ==`apt`== is the "app store" of ubuntu
- ==`update`== is like install on windows.

To VERIFY:
```bash
which setfacl   # should show /usr/bin/setfacl
setfacl --version
```

- ==`which`== prints out the full path to the `setfacl` binary (for example, `/usr/bin/setfacl`), or nothing if it isn’t installed.

#### ACL

- ==`setfacl`== on its own will return with an error.
- it must be used with an action:
	1. - **`-m`** (modify) → add or change the entry you specify, leaving _all other_ ACL entries and the basic owner/group/other bits intact.
	2. **`-x`** (remove) → delete the entry you specify, again leaving everything else intact.
	3. **`-b`** (remove all) → strip _every_ ACL entry off the file.
	4. **`--set`** (or `-S`) → replace the _entire_ ACL with exactly the list you give it (this _is_ destructive to the ACL).
	