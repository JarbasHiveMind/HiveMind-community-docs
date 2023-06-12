# Terminology

- `node` - anything connected to the hivemind, or listening for hivemind connections
- `mind` - a `node` that is listening for connections and providing ovos-core
    - `fakecroft` a `mind` that is not running ovos-core but pretends to be (might be running in a different `mind` further in the chain or not use ovos-core at all)
- `terminal` - user facing `node` that connects to some `mind` and does not itself accept connections
- `bridge` - `node` that connects some external service to a `mind`
- `hive` - a collection of interconnected `node`s
- `master mind` - the top level `node` of a `hive`, not connected to anything but receiving connections
