# Extra SaltStack Formulas
This is a collection of formulas for systems which are not covered by the official SaltStack formula collection. The formulas here do not adhere to the [official conventions](https://docs.saltproject.io/en/latest/topics/development/conventions/formulas.html#writing-formulas) but makes a new set of those:
- Avoid breaking up states into sub-functions, like `foo.install`, `foo.config`, `foo.service`. Control such operation via pillar values instead.
- Sub-modules are still a thing. Say your `foo` app has a `bar` sub component, then `foo.bar` would be the state to use.
- There's a unified method to load the values from the defaults and the pillar, which are flattened/merged into a single object. It's done using the `load_config.jinja` script, instead of dedicated `map.jinja` ones, which is maintained and documented [here](extra-formulas-common/)
- Some shared content is hosted in the [extra-formulas-common](extra-formulas-common/) repo, which should be added to your SaltStack `fileroot` along the actual formula.

The approach here is slightly different than upstream SaltStack:
1. by adding a formula to the `fileroots` in your master you're claiming that a formula exists (akin to a registry)
2. applying a state to certain hosts, via state `top.sls` file, would be the equivalent of "installing" the formula's "code" to those hosts
3. by leveraging the `use` pillar flag you can enable/disable the formula and install or uninstall the software itself.

## The `use` flag

With the `use` pillar flag you can disable the formula altogether. Just remove the entry from your pillar (which is the default on an empty pillar) and the formula won't do anything at all. It would be the same effect as removing the entry from state's `top.sls` file in upstream SaltStack formulas.

By setting the value to `true` you will trigger the `install` effect, the good 'ol regular formula application. By setting it to `false` you'd trigger the `uninstall` effect (which a lot of formulas lack, btw).

To achieve this, your custom formula should have a basic structure like:

```
{# remember to set your "default_sources" value #}
...
{% from "extra_formulas_common/load_config.jinja" import config as spam with context -%}

{% if spam.use is defined -%}

{% if spam.use | to_bool -%}

{# All your installation, config, start states go here #}

{%- else -%}

{# All your stop, uninstall, cleanup states go here #}

{%- endif %}

{%- else -%}

formula-spam-is-disabled:
  test.show_notification:
    - text: |
        The spam formula is disabled for this host
        You could enable it by setting the "use" flag in the pillar

{%- endif %}
```

## The textbook case

For a lot of software in mainstream linux distributions the formula would look like

```
{%- set default_sources = {'module' : 'spam', 'defaults' : True, 'pillar' : True, 'grains' : []} %}
{% from "extra_formulas_common/load_config.jinja" import config as spam with context -%}

{% if spam.use is defined -%}

{% if spam.use | to_bool -%}

spam_package_installation:
  pkg.installed:
    - name: {{ spam.package_name }}

spam_config_file:
  file.managed:
    - name: {{ spam.config_file_path }}
    - source: salt://{{ slspath }}/files/config.jinja
    - template: jinja
    - context: {{ spam | json }}
    - require:
      - pkg: {{ spam.package_name }}

spam_service_running:
  service.running:
    - name: {{ spam.service_name }}
    - enable: True
    - require:
      - file: {{ spam.config_file_path }}

{%- else -%}

spam_service_stopped:
  service.dead:
    - name: {{ spam.service_name }}
    - enable: False

spam_config_file_removal:
  file.absent:
    - name: {{ spam.config_file_path }}
    - require:
      - service: {{ spam.service_name }}

spam_package_removal:
  pkg.removed:
    - name: {{ spam.package_name }}
    - require:
      - file: {{ spam.config_file_path }}

{%- endif %}

{%- else -%}

formula-spam-is-disabled:
  test.show_notification:
    - text: |
        The spam formula is disabled for this host
        You could enable it by setting the "use" flag in the pillar

{%- endif %}
```

Remember to replace `spam` with the actual name of your software. Populate your `defaults/defaults.yaml` file with values for `package_name`, `config_file_path`, and `service_name`. The content of the `files/config.jinja` will depend on the software itself, the format expected, etc. Based on the content of the config file, you might want to keep adding some sane default values to `defaults/defaults.yaml` (anything host specific would have to be pulled from the pillar).

### Supporting more

Odds are that these values won't work for all distros out there, and some grains would have to be leveraged.

To make a really solid formula, your point of departure shouldn't be your distro's values but the upstream/developer defaults. Namely, you'd look for the developer's opinion to set the values in the defaults.yam` file. You could also add some "custom" defaults in here, like the `package_name` or `service_name` values. The upstream developer shouldn't have those around, since the first value is about the os packaging system and the second is about the os init system, not directly related to the software in question, BUT it's safe to assume that those values will match the name that the developer set for the software (and then just cover the divergences). For example, the Bind DNS server has different names in different distros over time, some call it `bind`, some `named`, some others use even different names. You could start by calling it `bind` and then just specify "the other names" where applicable.

Then you'd try your root distro's opinions on the values and set them on the `os_family.yaml` file. Meaning that you'd look for `RedHat`'s values instead of using the ones from `Rocky Linux`, `Debian`'s instead of the ones from `Ubuntu`, etc.

At that point you could then see which values in your os diverge from the family and set those in the `os.yaml` file.

A practical way to get this rolling is to move your `defaults.yaml` values from the previous section into the `osfinger.yaml` file (assume they're all specific to your current distro version). Then start looking for the values from the different sources and whenever something matches (it will already apply to your distro version), remove it from your `osfinger.yaml` file. Hopefully, everything will eventually be covered by more general files.

Some distros introduce breaking changes over time. The best way to tackle those is to assume the "new" stuff as the "default" stuff since it's safe to assume that future versions will keep this "new" way, and that's why it should go into the `os.yaml` file. This will of course break all the "old" versions, which would have to be covered with the `osfinger.yaml` file.