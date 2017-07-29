---
published: true
---



## Using a forked ansible galaxy role

Occasionally there may be a need to fix something in an [ansible galaxy](https://galaxy.ansible.com/) role, and use your fixed fork instead of the original role. Luckily since ansible galaxy introduced [using YAML files to configure dependencies](http://docs.ansible.com/ansible/galaxy.html#advanced-control-over-role-requirements-files), this is simple enough to do.

### Updating your requirements.yml
Lets say we're customising one of [Geerlingguy](https://galaxy.ansible.com/detail#/user/219)'s roles, usually we'd have the following in our `requirements.yml`:

```yaml
- src: geerlingguy.php
  version: 1.8.1
```

To use our fork, all we need to do is change the `src` over to our fork, and set the `name` to match the original role:
 
```yaml
- src: https://github.com/lewisw/ansible-role-php
  name: geerlingguy.php
```

### Using a specific branch
I personally like to keep all my fixes on a separate branch, so I can submit them as pull requests, so to use a specific branch just use `version`:

```yaml
- src: https://github.com/lewisw/ansible-role-php
  name: geerlingguy.php
  version: remotes/origin/configurable-pool-path
```

### Updating ansible galaxy
Then you'll need to re-run the ansible galaxy install command like so:

```bash
ansible-galaxy install -f -r requirements.yml
```

**Note:** The `-f` will force the reinstallation of your roles. It probably isn't necessary for ansible galaxy to pick-up your changes, but just to be on the safe side I always include it.

### Done
That's all, happy forking!
