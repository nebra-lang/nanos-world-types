# nanos-world-types

[Nebra](https://github.com/nebra-lang/nebra) type declarations for the
**[nanos world](https://nanos.world)** sandbox game, plus the event annotations that make
subscribing to game events a one-liner.

Types-only package: it emits no Lua. The declarations are auto-discovered from
`nebra_modules/` once the package is installed.

## Install

```bash
nebra add github:nebra-lang/nanos-world-types
```

Or scaffold a fresh package that already has everything wired up:

```bash
nebra create https://raw.githubusercontent.com/nebra-lang/nanos-world-types/main/setup.lua my-package
```

The setup script lays out the canonical nanos world structure (`Client/`, `Server/`, `Shared/`),
writes a `Package.toml` manifest and a `nebra.toml` with the right import rewriting and `[sides]`
configuration, and drops a minimal `Index.neb` into each side folder.

## What you get

**The full API surface** in `src/nanos-world.d.neb`, generated from the official nanos world API
manifests: classes, static classes, structs, enums, utility classes and the standard libraries.

**Typed events.** Every type with events has an interface listing them, such as
`CharacterEvents` or `ServerEvents`, and its `Subscribe` takes only those names and a callback
typed by the event it names. Inherited events are included, with `self` typed as the class
subscribed on:

```lua
Character.Subscribe("Death", function(character, last_damage, last_bone, damage_type, hit_from, instigator)
    -- character: Character, last_damage: number, instigator: Player?
end)

Server.Subscribe("Tick", function(delta_time)   -- delta_time: number
end)

Character.Subscribe("Deth", function() end)     -- error: not an event of Character
```

An event a callback can cancel or change, such as `TakeDamage`, takes its return value as
optional. Custom events through `Events.Subscribe` and `SubscribeRemote` keep any name, and so
does `Unsubscribe`.

**Execution sides.** nanos world scripts run on the client, the server, or both. The declarations
carry `@side(...)` markers, so calling a server-only API from a client file is a compile error
rather than a runtime nil.

**Event annotations** in `annotations/`, which rewrite the IR at compile time so the subscription
sits next to the handler:

```lua
@CharacterEvent("TakeDamage")
function onDamage(character: Character, damage: number)
    Console.Log("took " .. tostring(damage))
end
```

That compiles to the handler plus the `Character.Subscribe("TakeDamage", onDamage)` call you would
otherwise have written by hand. Available annotations: `@Event`, `@PlayerEvent`,
`@CharacterEvent`, `@EntityEvent`, `@RemoteEvent`, `@Command`, `@ConsoleCommand` and
`@nanosPackage`.

**Inheriting entities.** A class extending an entity class goes through nanos world's
[inheriting system](https://docs.nanos-world.com/docs/core-concepts/scripting/inheriting-classes)
by itself. `extends` becomes `Prop.Inherit("Crate")`, the constructor becomes `Constructor`,
`super` calls the native methods through `self.Super`, and `new Crate(...)` spawns it:

```lua
class Crate extends Prop
    constructor(location: Vector)
        super(location, new Rotator(), "nanos-world::SM_Cube")
    end

    function Explode(): nil
        self:Destroy()
    end
end
```

The class is registered under its own name. `@specifyNanosClass("Name")` sets another one, so a
class on the server and a class on the client can be the same entity to nanos world while having
different names and implementations in Nebra:

```lua
@specifyNanosClass("Crate")
class ServerCrate extends Prop
end
```

`operator []` and `operator []=` become the `index` and `newindex` hooks of the class, and they
receive the fields the class declares as well, so a typed field can live in the entity's values:

```lua
class Crate extends Prop
    weight: number = 5

    operator [](key: string): any
        return self:GetValue(key, nil)
    end

    operator []=(key: string, value: any)
        self:SetValue(key, value)
    end
end
```

`value is Crate`, `instanceof` and type patterns in `match` go through `IsA`.

## Regenerating the declarations

The API manifests live in `api/`. When nanos world ships an update, drop the new manifests in and
run the generator:

```bash
cd generate
npm install
node generate.js
```

It rewrites `src/nanos-world.d.neb` from the manifests. Do not hand-edit that file, the next
generator run will overwrite it.

## Related

- [Nebra](https://github.com/nebra-lang/nebra) - the compiler and toolchain
- [Documentation](https://nebra-lang.github.io) - language reference and guides
- [nanos-world-deathmatch](https://github.com/nebra-lang/nanos-world-deathmatch) - a full gamemode
  built on this package
- [Sides](https://nebra-lang.github.io/docs/advanced/sides) - how client/server scoping works
- [Annotations](https://nebra-lang.github.io/docs/advanced/annotations) - how the event
  annotations are implemented
