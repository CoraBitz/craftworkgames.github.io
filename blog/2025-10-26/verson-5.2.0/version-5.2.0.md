---
slug: version-5-2-0
title: Version 5.2.0 Released - ParticleEffectSerializer & Ember Fixes
authors: aris
tags: ['updates', 'releases', 'five-oh', 'ember']
enableComments: true
---

Hi everyone,

MonoGame Extended version 5.2.0 is now available, along with Ember 1.0.4. This release addresses some critical bugs in Ember's file serialization and introduces a new unified `ParticleEffectSerializer` class that aligns with standard .NET serialization patterns.

- GitHub: [https://github.com/monogame-extended/monogame-extended](https://github.com/monogame-extended/monogame-extended)
- Release Notes: [https://github.com/MonoGame-Extended/Monogame-Extended/releases/tag/v5.2.0](https://github.com/MonoGame-Extended/Monogame-Extended/releases/tag/v5.2.0)

## What's Changed

### New ParticleEffectSerializer

The `ParticleEffectReader` and `ParticleEffectWriter` classes have been consolidated into a single static `ParticleEffectSerializer` class. This brings the particle system's serialization in line with how other .NET serializers work, like `JsonSerializer`.

The new API is cleaner and more intuitive:

```csharp
// Loading a particle effect
ParticleEffect effect = ParticleEffectSerializer.Deserialize(filePath, contentManager);

// Saving a particle effect
ParticleEffectSerializer.Serialize(filePath, effect);
```

The old `ParticleEffectReader` and `ParticleEffectWriter` classes are now marked as obsolete and will be removed in version 6.0.0. If you're using these classes directly in your code, you should migrate to the new `ParticleEffectSerializer` API.

Reference: [https://github.com/MonoGame-Extended/Monogame-Extended/pull/1044](https://github.com/MonoGame-Extended/Monogame-Extended/pull/1044)

### Ember Serialization Fixes

There was a bug in Ember that was causing texture names to be saved incorrectly in the `.ember` XML files. Additionally, the enabled states for modifiers and interpolators weren't being persisted at all. Both of these issues have been resolved with this release.

If you created particle effects in previous versions of Ember, you may need to verify that texture references are correct when loading them in this version.

### XML Utility Improvements

New static utility methods have been added for working with XML attributes, making it easier to read and write particle effect data. These improvements are used internally by the `ParticleEffectSerializer` but are also available if you're extending the particle system or creating custom serialization logic.

### Additional Fixes

This release also includes a few smaller fixes:

- **FastRandom Upper Bounds**: The `FastRandom.Next` methods now correctly treat the upper bound as exclusive, matching the behavior of `System.Random`. This ensures consistent random number generation across the framework.
- **ShapeExtensions Disposal Check**: Added a defensive check in `ShapeExtensions` to prevent `ObjectDisposedException` when getting textures after a graphics device reset or on Android when the app returns from background.
- **Documentation Cleanup**: Removed outdated references to `UIBatcher` in code comments.

References:

- [https://github.com/MonoGame-Extended/Monogame-Extended/pull/1043](https://github.com/MonoGame-Extended/Monogame-Extended/pull/1043)
- [https://github.com/MonoGame-Extended/Monogame-Extended/pull/1041](https://github.com/MonoGame-Extended/Monogame-Extended/pull/1041)
- [https://github.com/MonoGame-Extended/Monogame-Extended/pull/1042](https://github.com/MonoGame-Extended/Monogame-Extended/pull/1042)

## Migration Guide

If you're using `ParticleEffectReader` or `ParticleEffectWriter` directly, here's how to migrate:

**Old way:**

```cs
// Reading
using ParticleEffectReader reader = new ParticleEffectReader(filePath, contentManager);
ParticleEffect effect = reader.ReadParticleEffect();

// Writing
using ParticleEffectWriter writer = new ParticleEffectWriter(filePath);
writer.WriteParticleEffect(effect);
```

**New way:**

```cs
// Reading
ParticleEffect effect = ParticleEffectSerializer.Deserialize(filePath, contentManager);

// Writing
ParticleEffectSerializer.Serialize(filePath, effect);
```

The static methods handle all resource management internally, so you no longer need to worry about `using` statements or disposal.

## What's Next?

With these fixes in place, the particle system continues in maintenance mode. My focus remains on the tile map system, where I'm working on bringing similar tooling and documentation quality that we've achieved with particles.

As always, if you encounter any issues or have feedback, please open an issue on GitHub. Your input helps make MonoGame Extended better for everyone.

Happy coding!

\- ❤️ Chris Whitley ([AristurtleDev](https://github.com/aristurtledev))