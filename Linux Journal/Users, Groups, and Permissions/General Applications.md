As part of the first week schedule, i created a training directory to set permissions and learn chmod and ACL.

Creating user:
```bash
sudo useradd -m skyuser
```

- **==`-m`==** is used to create a ==**HOME**== directory for skyuser

Setting password:
```bash
sudo passwd skyuser
```

#### Changing ownership of /var/log/sky_demo directory:

==`chmod`== is used at a basic level, while ==ACL== is used to more intricately alter permissions on files across multiple users, setting exceptions and such...

Using ch:
```bash
sudo chown skyuser:skyuer /var/log/sky_demo
```

- This changed the ownership of the directory to skyuser of group skyuser. The group was created automatically when the user was created.

```bash
sudo chmod 750 /var/log/sky_demo
```

- This sets the basic permissions for the file ==based on the owner and group of the owner==
- _Breakdown of ==`750`_==:

| Digit | Binary | Meaning                                    |     |
| ----- | ------ | ------------------------------------------ | --- |
| **7** | 111    | owner can **r**ead, **w**rite, e**x**ecute |     |
| **5** | 101    | group can read & execute (no write)        |     |
| **0** | 000    | others get nothing                         |     |

To ==CHECK== the permissions on a file:
```bash
namei -l /var/log/sky_demo
```

- ==`namei`== (name interpreter) displays the permissions of a specific file with every user and group that may or may not have permissions to it.
- ==`-l`== means (long form) which displays the information in a more detailed manner.

To assign a new group and give it ACL permissions:
```bash
sudo groupadd ops                # create the group
sudo setfacl -m g:ops:rx /var/log/sky_demo
getfacl /var/log/sky_demo        # shows the ACL entries
```

For detailed ACL usage, view the [[ACL notes]] 