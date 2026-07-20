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
{%- set default_sources = {'module' : 'spam', 'defaults' : True, 'pillar' : True, 'grains' : ['os_family', 'os', 'osfinger']} %}
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

spam_package_removal:
  pkg.removed:
    - name: {{ spam.package_name }}

{%- endif %}

{%- else -%}

formula-spam-is-disabled:
  test.show_notification:
    - text: |
        The spam formula is disabled for this host
        You could enable it by setting the "use" flag in the pillar

{%- endif %}
```

The content of the `files/config.jinja` will depend on the software itself, the format expected, etc.

You'd then have to populate your `defaults` files with values for `package_name`, `config_file_path`, and `service_name`depending on variations across distributions. A simple solution is to start by putting your distro's values into `defaults.yaml`. Then try the formula in a different linux family and move whatever values vary into the `os_family.yaml` file. You should also look for variations in the same family and move those into the `os.yaml` file. The `osfinger.yaml` file is usually leveraged in special cases, when certain versions of a distro break the way things were done in the family or os before it.