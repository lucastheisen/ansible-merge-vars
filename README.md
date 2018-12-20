# Ansible `merge_vars` Action Plugin

This plugin is similar to `include_vars`, but designed to merge all the variables in the files specified by `from` rather than replacing.

_Note, this implimentation is not necessarily safe as it has to [pretend to be `include_vars`](https://github.com/lucastheisen/ansible-merge-vars/blob/master/lib/plugins/action/merge_vars.py#L23) to get interpolation when the facts are set._

## Background

This plugin is designed to support the inclusion of _DRY_ application configuration within ansible vars.  For example, if you have multiple porjects (foo, bar), and multiple environments (dev, qa, prod), and some vars are shared at various levels (project, or environment), and you want to keep your configuration _DRY_, then you can:
```
- name: Merge in application configuration
  merge_vars:
    from:
    - "vars/app/{{ app_project }}.yml"
    - "vars/app/{{ app_environment }}.yml"
    - "vars/app/{{ app_project }}-{{ app_environment }}.yml"
```

When paired with [inventory/hosts](https://github.com/lucastheisen/ansible-merge-vars/blob/master/inventory/hosts):
```
all:
  hosts:
    bar-dev:
    bar-qa:
    foo-dev:
    foo-qa:
  children:
    bar:
      hosts:
        bar-dev:
        bar-qa:
      vars:
        app_project: bar
    dev:
      hosts:
        bar-dev:
        foo-dev:
      vars:
        app_environment: dev
        k8s_cluster: dev-cluster.pastdev.com
        k8s_verify_ssl: False
    foo:
      hosts:
        foo-dev:
        foo-qa:
      vars:
        app_project: foo
    k8s_namespaces:
      hosts:
        bar-dev:
        bar-qa:
        foo-dev:
        foo-qa:
      vars:
        ansible_connection: local
        k8s:
          cluster: "{{ k8s_cluster }}"
          namespace: "{{ app_project }}-{{ app_environment }}"
          verify_ssl: "{{ k8s_verify_ssl }}"
    qa:
      hosts:
        bar-qa:
        foo-qa:
      vars:
        app_environment: qa
        k8s_cluster: qa-cluster.pastdev.com
        k8s_verify_ssl: true
```

And app vars:

_vars/app/foo/vars.yml_
```
app:
  db:
    hostname: foo_db
```

_vars/app/dev/vars.yml_
```
app:
  debug: true
```

_vars/app/foo-dev/vars.yml_
```
app:
  db:
    username: "{{ vault_app.db.username }}"
    password: "{{ vault_app.db.password }}"
```

_vars/app/foo-dev/vault.yml_
```
vault_app:
  db:
    username: foo_username
    password: foo_password
```

Would result in:
```
ok: [foo-dev] => (item=app) => {
    "app": {
        "db": {
            "hostname": "foo_db",
            "password": "foo_password",
            "username": "foo_username"
        },
        "debug": true
    },
    "item": "app"
}
ok: [foo-dev] => (item=k8s) => {
    "item": "k8s",
    "k8s": {
        "cluster": "dev-cluster.pastdev.com",
        "namespace": "foo-dev",
        "verify_ssl": false
    }
}
```

## Alternatives

1. Per [this conversation](https://meetbot.fedoraproject.org/ansible-meeting/2018-12-20/ansible_core_irc_meeting.2018-12-20-15.07.log.html) during an ansible-meeting, you could use the `hash_behavior: merge` configuration option.  The downside to this would be that:

  * This setting applies globally which means:
    * The modules/plays/roles are not portable (other consumers would have to also have that global setting)
  * This setting is ignored by `include_vars` when using `dir` feature ([_15:48:05 <bcoca> there is one thing in i.nclude_vars that does NOT follow this and that is the 'directory' feature, buyt that is internal to the plubing_](https://meetbot.fedoraproject.org/ansible-meeting/2018-12-20/ansible_core_irc_meeting.2018-12-20-15.07.log.html))
    * This could be resolved by using inline vault vars and a single vars file (instead of the `sensitive_var: {{ vault_sensitive_var }}` idiom suggested by the [_Best Practices documentation_](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#variables-and-vaults)).
      * _The single vars file would not allow the _DRY_ approach of `foo`, `dev`, and `foo-dev` variables_
      * _Inline vars are more difficult to edit, you have to `ansible-vault encrypt` then copy/paste_
      * _Inline vars add giant blobs of incomprehensible text to what is otherwise a human readable configuration file_

2. Add a `merge: yes` option to `include_vars` itself to scope the merge at a single task rather than the entire project via `hash_behavior: merge`

3. Add this plugin as a peer of `include_vars` to the ansible core where it could be _trusted_ in the same fashion as `include_vars`
  * This would be simple enough to do as it is a very simple plugin.  It just requires buy in from ansible core.
