# Extra SaltStack Formulas
This is a collection of formulas for systems which are not covered by the official SaltStack formula collection. The formulas here do not adhere to the [official conventions](https://docs.saltproject.io/en/latest/topics/development/conventions/formulas.html#writing-formulas) but makes a new set of those:
- Avoid breaking up states into sub-functions, like `foo.install`, `foo.config`, `foo.service`. Control such operation via pillar values instead.
- Sub-modules are still a thing. Say your `foo` app has a `bar` sub component, then `foo.bar` would be the state to use.
- There's a unified method to load the values from the defaults and the pillar, which are flattened/merged into a single object. It's done using the `load_config.jinja` script, instead of dedicated `map.jinja` ones, which is maintained and documented [here](extra-formulas-common/)
- Some shared content is hosted in the [extra-formulas-common](extra-formulas-common/) repo, which should be added to your SaltStack `fileroot` along the actual formula.

## Dummy example

Directory structure:
- spam
  - defaults
    - defaults.yaml
    - os.yaml
    - os_family.yaml
    - osfinger.yaml
  - init.sls

The "init.sls" file should start with something like this:
"""
{%- set default_sources = {'module' : 'spam', 'defaults' : True, 'pillar' : True, 'grains' : ['os_family', 'os', 'osfinger']} %}
{%- from "extra_formulas_common/load_config.jinja" import config as spam with context %}

{%- if spam.use is defined %}
...
"""

## Submodule example

Directory structure:
- foo
  - bar
    - init.sls
  - defaults
    - defaults.yaml
    - os.yaml
    - os_family.yaml
    - osfinger.yaml
  - init.sls

The "bar/init.sls" file should start with something like this:
"""
{%- set default_sources = {'module' : ['foo', 'bar'], 'defaults' : True, 'pillar' : True, 'grains' : ['os_family', 'os', 'osfinger']} %}
{%- from "extra_formulas_common/load_config.jinja" import config as bar with context %}

{%- if bar.use is defined %}
...
"""