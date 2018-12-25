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
   * Would this work with the `dirs` feature, or not for the same reason `dirs` doesn't work with `hash_behavior: merge`?
   ```
   ltheisen@ltp52s:~/git/pastdev-ansible$ diff -u ../ansible/lib/ansible/modules/utilities/logic/include_vars.py lib/modules/utilities/logic/include_vars.py
   --- ../ansible/lib/ansible/modules/utilities/logic/include_vars.py      2018-12-16 15:10:23.054969700 -0500
   +++ lib/modules/utilities/logic/include_vars.py 2018-12-25 13:20:37.319859200 -0500
   @@ -39,6 +39,10 @@
        version_added: "2.2"
        description:
          - The name of a variable into which assign the included vars. If omitted (null) they will be made top level vars.
   +  hash_behaviour:
   +    version_added: "2.8"
   +    description:
   +      - Either 'replace' or 'merge', determines the behavior when var already exists.
      depth:
        version_added: "2.2"
        description:
   ```
   ```
   ltheisen@ltp52s:~/git/pastdev-ansible$ diff -u ../ansible/lib/ansible/plugins/action/include_vars.py lib/plugins/action/include_vars.py
   --- ../ansible/lib/ansible/plugins/action/include_vars.py       2018-12-16 15:10:25.148728700 -0500
   +++ lib/plugins/action/include_vars.py  2018-12-24 16:27:49.483592500 -0500
   @@ -25,6 +25,7 @@
    from ansible.module_utils.six import string_types
    from ansible.module_utils._text import to_native, to_text
    from ansible.plugins.action import ActionBase
   +from ansible.utils.vars import merge_hash
   
   
    class ActionModule(ActionBase):
   @@ -34,7 +35,7 @@
        VALID_FILE_EXTENSIONS = ['yaml', 'yml', 'json']
        VALID_DIR_ARGUMENTS = ['dir', 'depth', 'files_matching', 'ignore_files', 'extensions', 'ignore_unknown_extensions']
        VALID_FILE_ARGUMENTS = ['file', '_raw_params']
   -    VALID_ALL = ['name']
   +    VALID_ALL = ['name', 'hash_behaviour']
   
        def _set_dir_defaults(self):
            if not self.depth:
   @@ -72,6 +73,8 @@
            self.ignore_files = self._task.args.get('ignore_files', None)
            self.valid_extensions = self._task.args.get('extensions', self.VALID_FILE_EXTENSIONS)
   
   +        self.hash_behaviour = self._task.args.get('hash_behaviour', 'replace')
   +
            # convert/validate extensions list
            if isinstance(self.valid_extensions, string_types):
                self.valid_extensions = list(self.valid_extensions)
   @@ -148,12 +151,25 @@
                result['failed'] = failed
                result['message'] = err_msg
   
   +        result['ansible_facts_hash_behaviour'] = self.hash_behaviour
   +        if (self.hash_behaviour == 'merge'):
   +            results = self._merge_here_rather_than_task_executor_cause_i_dont_wanna_modify_core(task_vars, results)
   +
            result['ansible_included_var_files'] = self.included_files
            result['ansible_facts'] = results
            result['_ansible_no_log'] = not self.show_content
   
            return result
   
   +    def _merge_here_rather_than_task_executor_cause_i_dont_wanna_modify_core(self, task_vars, results):
   +        data = {}
   +
   +        for key in results:
   +            if key in task_vars:
   +                data[key] = task_vars[key]
   +
   +        return merge_hash(data, results)
   +
        def _set_root_dir(self):
            if self._task._role:
                if self.source_dir.split('/')[0] == 'vars':
   ```

3. Add this plugin as a peer of `include_vars` to the ansible core where it could be _trusted_ in the same fashion as `include_vars`
   * This would be simple enough to do as it is a very simple plugin.  It just requires buy in from ansible core.
