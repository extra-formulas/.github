# Extra SaltStack Formulas
This is a collection of formulas for systems which are not covered by the official SaltStack formula collection. The formulas here do not adhere to the [official conventions](https://docs.saltproject.io/en/latest/topics/development/conventions/formulas.html#writing-formulas) but makes a new set of those:
- Avoid breaking up states into sub-functions, like `foo.install`, `foo.config`, `foo.service`. The formulas here are about setting up (or tearing down) services; if you just want to install a package (the `foo.install` case) there's already [a formula for that](https://github.com/saltstack-formulas/packages-formula). The `foo.config` and `foo.service` states are closely tied, there's really little use cases to have one without the other.
- Sub-modules are still a thing. Say your `foo` app has a `bar` sub component, then `foo.bar` would be the state to use. This is useful for big pieces of software which have different components. You could have a sub-state for each component instead of having different formulas (this assumes components are independent enough, to the point that they can be installed isolated from each other).
- There's a unified method to load the values from the defaults and the pillar, which are flattened/merged into a single object. It's done using the `load_config.jinja` script, instead of dedicated `map.jinja` ones, which is maintained and documented [here](../../../../extra-formulas-common/)
- Some shared content is hosted in the [extra-formulas-common](../../../../extra-formulas-common/) repo, which should be added to your SaltStack `fileroots` along the actual formula.

The way these formulas work is slightly different than upstream SaltStack:
1. By adding a formula to the `fileroots` in your master, you're claiming that a formula exists (akin to a registry).
2. Setting a state to certain hosts, via state `top.sls` file, would be the equivalent of "installing" the formula's "code" to those hosts (the states would be then available to the hosts).
3. By leveraging the `use` pillar flag you can then enable/disable the formula and install or uninstall the software/service itself.

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
{%- set default_sources = {'module' : 'spam', 'defaults' : False, 'pillar' : True, 'grains' : ['osfinger']} %}
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

Remember to replace `spam` with the actual name of your software. Populate your `defaults/osfinger.yaml` file with values for `package_name`, `config_file_path`, and `service_name`. Keep in mind that the root of the file is a dict/mapping with the `osfinger` values as keys. To get your key, use SaltStack's `grains.get osfinger` on your host. The content of the `files/config.jinja` will depend on the software itself, the format expected, etc. Based on the content of the config file, you might want to keep adding some sane default values to `defaults/osfinger.yaml` (anything host specific would have to be pulled from the pillar).

### Supporting more

To populate the `defaults.yaml` file you'd look at the upstream developer's opinion. You could also add some "custom" defaults in here, like the `package_name` or `service_name` values. The upstream developer shouldn't have those around, since the first value is about the OS' packaging system and the second is about the OS' init system, not directly related to the software in question, BUT it's safe to assume that those values will match the name that the developer set for the software (and then just cover the divergences). For example, the Bind DNS server has different names in different distros over time, some call it `bind`, some `named`, some others use even different names. You could start by calling it `bind` and then just specify "the other names" where applicable.

Then you'd try your root distro's opinions on the values and set them on the `os_family.yaml` file. Meaning that you'd look for `RedHat`'s values instead of using the ones from `Rocky Linux`, `Debian`'s instead of the ones from `Ubuntu`, etc.

At that point you could then see which values in your OS diverge from the family and set those in the `os.yaml` file.

Some distros introduce breaking changes over time. The best way to tackle those is to assume the "new" stuff as the "default" stuff since it's safe to assume that future versions will keep this "new" way, and that's why it should go into the `os.yaml` file. This will of course break all the "old" versions, which would have to be covered in the `osfinger.yaml` file.

At this point you could remove any redundant values from your original entry in the `osfinger.yaml` file. With any luck, everything will be covered by more general files and the whole entry could be deleted.

Remember to update your `default_sources` variable at every step of the way for the changes to apply.

### What about building source tarballs?

Upstream SaltStack encourages the build and installation of software directly from source tarballs. It's a REALLY BAD IDEA to do so, you're basically bleeding the package management functionality into SaltStack. Package management is a big and complicated problem on its own, and each distro tackles it in their own way, by "simplifying" it with salt you're not doing anyone any favors. You could say that what separates a distro from the next is the way each tackles package management.

We'd say that "the sane approach" to achieving the same result is by running a private repo compatible with your OS. You'd publish native packages of pre-built pieces of software there, taking into consideration all the potential issues: distinguish different distro versions, take into account name collisions, keep track of package versions, etc. By doing this you can leverage your OS' packaging system including the install and removal functionalities that you'd then leverage in your formula. Even if you feel the temptation because your software is self-contained (it lives completely inside a directory in `/opt/`) you'd still find a lot of issues with "installed version detection" which is a critical functionality to handle upgrades. It's always better to just use the native packaging and the functionalities there.
