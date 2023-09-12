# Provision Nodes

## Authorized keys

Add SSH public keys to `common_authorized_keys` in `group_vars/node.yml` to be
able to SSH into hosts with the `common_user` (which defaults to `ansible`).
This is used by the `common` Role.

## Initialize/provision host for the first time

Use `--limit node.002.012` to filter only matching nodes you want to init.

Set the initial SSH user with `-u user` and `-e 'ansible_user=user` where
the username is `user`.

`-k` signals to ask for SSH passwords and `--ask-become-pass` is for the `sudo`
password.

`-t` will only run the tagged `init` tasks which are needed for initial setup.

Ensure at least one SSH key on the host running the playbook is in the
`common_authorized_keys` variable (see [Authorized keys](#authorized-keys)).

```shell
ansible-playbook -k --ask-become-pass -i inventory site.yml -t init --limit node.002.012 -u user -e 'ansible_user=user'
```

## Regular run

```shell
ansible-playbook -i inventory site.yml -v
```

## Debugging

Add the `-vvvv` flag for maximum verbosity.
