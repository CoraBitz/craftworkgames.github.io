---
slug: version-6
title: Version 6.0.0 - 2D Geometry, Tilemaps, Collision, and ECS Updates
authors: aris
tags: ['updates', 'releases', 'six-oh', 'tilemaps', 'collision', 'ecs']
enableComments: true
---

Hi everyone,

Version 6.0.0 of MonoGame.Extended is here.

This release has been a long time coming, and it touches some of the oldest and most heavily used parts of the library. The biggest changes in 6.0.0 are a new 2D geometry suite, a fully overhauled 2D collision system, a completely rewritten tilemap system, and an important upgrade to the Entity Component System.

Some of these features started life as separate efforts and grew over time as the design became clearer. Rather than treat them as isolated additions, 6.0.0 brings them together into a more consistent foundation for games that rely on geometry, collisions, worlds, and rendering-heavy content pipelines.

<!-- truncate -->

## Entity Component System Changes

### Component Type Limit Increased to 256

The ECS component system now supports up to 256 different component types, up from the previous limit of 32. This removes a significant constraint for complex games with diverse entity compositions.

The implementation replaces `BitVector32` with a new `ComponentBits` structure that uses 256 bits (32 bytes) to track component membership. The new structure keeps the same high-performance spirit as the original while giving quite a bit more room to grow.

**Breaking Change:** The `ComponentBits` property on entities and aspect bit sets now returns `ComponentBits` instead of `BitVector32`. The indexer behavior has also changed from mask-based to index-based access:

```cs
// Before (BitVector32 - mask-based)
BitVector32 bits = entity.ComponentBits;
bool hasComponent = bits[1 << componentId];

// After (ComponentBits - index-based)
ComponentBits bits = entity.ComponentBits;
bool hasComponent = bits[componentId];
```

Most users will not need to change anything because the high-level ECS APIs remain the same:

```cs
// These APIs work exactly the same
entity.Has<Position>();
entity.Attach(new Velocity());
Aspect.All(typeof(Position), typeof(Velocity)).Build(componentManager);
```

This mainly affects advanced usage where code was directly inspecting `ComponentBits` or manipulating aspect bit sets by hand.

Reference: https://github.com/MonoGame-Extended/MonoGame-Extended/issues/64

---

## 2D Geometry

Earlier in the 6.0 cycle, I wrote about how the collision work I was doing for MonoGame.Extended had grown into something that felt like it belonged in MonoGame itself rather than exclusively here. I reached out to the MonoGame Foundation, they were on board with the idea, and I submitted a pull request directly to the MonoGame repository with the full implementation.

That work is still relevant here, and in 6.0.0 it ships as part of MonoGame.Extended so it can be used right now in real projects. If the equivalent work lands upstream in MonoGame later, these APIs can eventually be deprecated in favor of the core framework types. For now, they are part of the Version 6 foundation.

The implementation is based on Christer Ericson's *Real-Time Collision Detection*. Five bounding volume types are provided: `BoundingBox2D`, `BoundingCircle2D`, `BoundingCapsule2D`, `OrientedBoundingBox2D`, and `BoundingPolygon2D`. Three geometric primitive types are also included: `Line2D`, `LineSegment2D`, and `Ray2D`. Every type is a `struct` with no heap allocation overhead.

All types support intersection tests, containment queries, and distance computations against every other type in the suite. For example:

```cs
BoundingBox2D box = new BoundingBox2D(new Vector2(0, 0), new Vector2(100, 100));
BoundingCircle2D circle = new BoundingCircle2D(new Vector2(80, 80), 30f);
Ray2D ray = new Ray2D(new Vector2(0, 50), Vector2.UnitX);

bool boxHitsCircle = box.Intersects(circle);
ContainmentType containment = box.Contains(circle);

if (ray.Intersects(box, out float? tMin, out float? tMax))
{
    Vector2 entryPoint = ray.GetPoint(tMin.Value);
}
```

For the full API including construction helpers, transformation, merging, closest-point queries, and the `Collision2D` static class, see the [2D Geometry documentation](https://monogame-extended.github.io/docs/features/collision/2d-geometry).

---

## Collision System Overhaul

The 2D collision system has been substantially reworked in 6.0.0.

The old collision component APIs had been showing their age for a while. They mixed concerns, still carried some legacy shape abstractions, and made it harder than it needed to be to express things like layer ownership, layer filtering, and collision queries. This release replaces that older direction with a shape-based, query-oriented collision model built on top of the new 2D geometry types.

At the narrowphase level, the system now exposes `CollisionResult2D` and high-level `TryGetCollision` APIs across the supported 2D bounding volume types. That includes the SAT-style convex shape handling that had been requested for a long time, along with consistent minimum translation vector and collision normal results.

On the actor side, collision data is now centered around `CollisionShape2D`, which gives actors an explicit shape representation instead of relying on the older `IShapeF` driven collision flow. This also helped close out the remaining boxing and compatibility problems that had built up around the old abstractions.

The broadphase layer has also been updated to use `BoundingBox2D` consistently. `QuadTreeSpace` and `SpatialHash` now work against the same broadphase contract, which makes the collision pipeline easier to reason about and easier to extend. As part of this work, the spatial hash implementation was tightened up to handle cell coverage, negative coordinates, duplicate suppression, and related edge cases more reliably.

`CollisionWorld2D` has been reworked around named layers and explicit query APIs. Instead of pushing everything through the older collision component style, you can now:

- insert actors into named layers
- control which layers collide with which other layers
- query broadphase candidates by bounds or by actor
- query narrowphase collision results directly
- query collision pairs between layers with duplicate suppression

That makes the system much more flexible for real game scenarios where not every actor should be checked against every other actor.

This overhaul also removes the remaining legacy compatibility paths for the old shape helper and penetration bridge APIs, and it updates the collision source and tests to match the current repository coding guidelines.

For more on the collision APIs and supporting docs, see the [Collision feature documentation](https://monogame-extended.github.io/docs/features/collision/collision).

---

## New Tilemap System

The old `MonoGame.Extended.Tiled` integration has been replaced by a completely new tilemap system.

The fundamental problem with the old system was that it was tightly coupled to the Tiled map editor format. Every API was shaped around Tiled concepts. If you used a different editor, or if your game needed to switch editors later, you were starting over. The new system is format-agnostic. You load a `Tilemap` object, and whether it came from a Tiled `.tmx` file, an LDtk `.ldtk` project, or an Ogmo `.ogmo` file does not matter to the rest of your game code.

Tiled, LDtk, and Ogmo Editor are all supported. All three share the same runtime `Tilemap` type, the same layer access API, the same object and property model, and the same renderers. Maps can be loaded through the content pipeline or at runtime using a format-specific parser.

Two renderers are provided. `TilemapSpriteBatchRenderer` integrates with `SpriteBatch` and performs frustum culling, submitting only tiles visible in the current camera view. `TilemapRenderer` uses `GraphicsDevice` directly and pre-bakes all tile geometry into GPU buffers at load time, which produces fewer draw calls on static maps and supports merging multiple layers into a single draw call via layer groups.

Getting a map on screen is straightforward:

```cs
_tilemap = Content.Load<Tilemap>("maps/level1");

_renderer = new TilemapSpriteBatchRenderer();
_renderer.BlendState = BlendState.AlphaBlend;
_renderer.LoadTilemap(_tilemap);
```

```cs
protected override void Draw(GameTime gameTime)
{
    GraphicsDevice.Clear(_tilemap.BackgroundColor ?? Color.Black);
    _renderer.Draw(_spriteBatch, _camera);
}
```

Beyond rendering, the system provides typed access to tile layers, object layers, image layers, custom properties, and coordinate conversion between tile and world space.

World maps are also supported for multi-room games. A world map loads multiple individual tilemap levels and positions them in a shared coordinate space, which is useful for GridVania-style layouts or any game where the world spans more than one map file. Tiled `.world` files and LDtk projects are supported natively, and a small custom `.tilemapworld` format is provided for editors like Ogmo that do not have a native world file format.

```cs
TilemapWorld world = Content.Load<TilemapWorld>("maps/world");
```

For the full API, see the [Tilemaps documentation](https://monogame-extended.github.io/docs/features/tilemaps).

---

## Migrating from MonoGame.Extended.Tiled

The old `MonoGame.Extended.Tiled` namespace is gone in 6.0.0. The new system has a different API surface, different content pipeline importers, and a different namespace structure, so migration is not a find-and-replace operation. A migration guide covering the most common patterns is available in the documentation:

[https://monogame-extended.github.io/docs/features/tilemaps/migration](https://monogame-extended.github.io/docs/features/tilemaps/migration)

---

## Closing Thoughts

Version 6.0.0 represents a large amount of work across several parts of the library that have needed attention for a long time. Some of these changes are additive, some are cleanup, and some are foundational enough that they will shape how future work in MonoGame.Extended is built.

The 2D geometry suite, the collision overhaul, the tilemap rewrite, and the ECS bitset expansion all move the library in the same direction: clearer APIs, fewer legacy constraints, better performance characteristics, and a better foundation for real games.

As always, thank you to everyone who has supported the project through issue reports, pull requests, Discord questions, testing, and GitHub Sponsors. It makes a real difference.

\- ❤ Chris Whitley (AristurtleDev)
