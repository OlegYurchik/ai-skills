---
name: python-development-settings
description: Application configuration and settings management. Use when defining, loading, validating, or extending application settings.
---

## Rules
- The root settings class must inherit from `pydantic_settings.BaseSettings`. This enables automatic loading from environment variables and `.env` files.
- Every nested settings class (for individual services, adapters, or components) must inherit from `pydantic.BaseModel`, not from `BaseSettings`. Only the root class needs the env-loading behavior.
- The root settings class exposes one field per top-level service or subsystem. Provide sensible defaults as plain values for immutable types (str, int, bool, None) and use `Field(default_factory=...)` only when the default is a mutable object (dict, list, or model instance). Never use mutable literals as class-attribute defaults in Pydantic v2. Provide defaults so the application starts without a config file when reasonable.
- A settings class can accept either a settings object or a string key referencing another entry in the same config. The resolver (or a `@model_validator`) replaces string references with resolved instances after initial parsing. This is how services declare their dependencies declaratively.
- Use `@model_validator(mode="after")` on the root settings class to validate cross-references between subsystems, resolve string keys to actual instances, and enforce consistency across the configuration graph.
- Support loading from YAML config files with custom tags. Register `!include` for file inclusion and `!env` for environment variable interpolation inside YAML. The YAML loader builder must live as a `@staticmethod` inside the root settings class. Example:
  ```python
  @staticmethod
  def get_yaml_loader(base_path: pathlib.Path) -> yaml.BaseLoader:
      loader = type("Loader", (yaml.SafeLoader,), {})
      loader.add_constructor(
          "!include",
          yaml_include.Constructor(base_dir=str(base_path)),
      )
      loader.add_constructor("!env", construct_env_tag)
      return loader
  ```
- Support loading from JSON config files using standard `json.load`. JSON does not support custom tags; use string interpolation or env var resolution handled by `BaseSettings` after load.
- Provide a single `load()` classmethod on the root settings class that reads the config file (YAML or JSON), then passes the parsed dict into the `BaseSettings` constructor along with optional `_env_prefix` and `_env_file`. Example:
  ```python
  @classmethod
  def load(cls, config_file: pathlib.Path | None = None) -> Self:
      data = {}
      if config_file is not None:
          if config_file.suffix in (".yaml", ".yml"):
              data.update(yaml.load(config_file.read_text(), Loader=get_loader()))
          elif config_file.suffix == ".json":
              data.update(json.loads(config_file.read_text()))
      return cls(**data)
  ```
- Keep each subsystem's settings in its own `settings.py` module inside the subsystem package. Import and compose them into the root settings class. Do not inline subsystem schemas into the root file.
- Use `pydantic.Field(default_factory=...)` for mutable default values (dicts, lists, models). Never use mutable literals as class-attribute defaults in Pydantic v2.
