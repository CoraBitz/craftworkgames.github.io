---
id: quick-start
sidebar_label: Quick Start
title: Collision Quick Start
description: Get actor/world collision queries running with CollisionWorld2D in MonoGame.Extended 6.0 preview.
---

:::note[Preview release]
This feature is currently only available in the preview release **6.0.0-preview.1**. If you find outdated information, [please open an issue](https://github.com/monogame-extended/monogame-extended.github.io/issues).
:::

This guide shows the main 6.0 collision workflow built around `ICollisionActor`, `CollisionShape2D`, `Layer`, and `CollisionWorld2D`.

Use this path when you want:

- explicit collision queries
- named collision layers
- access to `CollisionResult2D`
- a collision system built on the new bounding-volume APIs instead of `IShapeF`

## Prerequisites

- A MonoGame project with MonoGame.Extended installed
- Basic familiarity with `Game.Update`
- A gameplay object you want to test against the collision world

## Step 1: Add Namespaces

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Input;
using MonoGame.Extended;
using MonoGame.Extended.Collisions;
using MonoGame.Extended.Collisions.Layers;
```

## Step 2: Implement `ICollisionActor`

Each actor needs:

- a stable `Id`
- an optional `LayerName`
- a `CollisionShape2D`

`CollisionWorld2D` is query-oriented, so collision resolution happens in your game code rather than through actor callbacks.

```cs
public sealed class PlayerActor : ICollisionActor
{
    public PlayerActor(int id, Vector2 position, Vector2 size)
    {
        Id = id;
        Position = position;
        _size = size;
        UpdateShape();
    }

    public int Id { get; }

    public string LayerName => CollisionWorld2D.DefaultLayerName;

    public Vector2 Position { get; private set; }

    public CollisionShape2D Shape { get; private set; }

    private readonly Vector2 _size;

    public void Move(Vector2 delta)
    {
        Position += delta;
        UpdateShape();
    }
    private void UpdateShape()
    {
        BoundingBox2D bounds = BoundingBox2D.CreateFromPositionAndSize(Position, _size);
        Shape = new CollisionShape2D(bounds);
    }
}
```

For a wall actor, the same pattern works:

```cs
public sealed class WallActor : ICollisionActor
{
    public WallActor(int id, BoundingBox2D bounds)
    {
        Id = id;
        Shape = new CollisionShape2D(bounds);
    }

    public int Id { get; }

    public string LayerName => "walls";

    public CollisionShape2D Shape { get; }
}
```

## Step 3: Create the Collision World and Layers

`CollisionWorld2D` stores actors in named `Layer` instances. Each layer wraps a broadphase structure such as `SpatialHash`.

```cs
private CollisionWorld2D _collisionWorld;
private PlayerActor _player;
private WallActor _wall;

protected override void Initialize()
{
    base.Initialize();

    Layer defaultLayer = new Layer(new SpatialHash(new SizeF(64f, 64f)));
    Layer wallLayer = new Layer(new SpatialHash(new SizeF(64f, 64f)));

    _collisionWorld = new CollisionWorld2D(defaultLayer);
    _collisionWorld.AddLayer("walls", wallLayer);

    _player = new PlayerActor(1, new Vector2(40f, 40f), new Vector2(32f, 32f));
    _wall = new WallActor(
        2,
        BoundingBox2D.CreateFromPositionAndSize(new Vector2(96f, 40f), new Vector2(64f, 64f)));

    _collisionWorld.Insert(_player);
    _collisionWorld.Insert(_wall);
}
```

By default:

- the default layer is named `default`
- new non-default layers enable self-collision automatically
- new non-default layers also enable collision with the current default layer automatically

If you need to inspect or change that behavior, use:

- `EnableCollisionBetweenLayers(...)`
- `DisableCollisionBetweenLayers(...)`
- `IsCollisionEnabledBetweenLayers(...)`

## Step 4: Query Collisions

Call `QueryCollisions(...)` when you want collision results relative to one actor.

```cs
foreach (CollisionEvent2D collision in _collisionWorld.QueryCollisions(_player, "walls"))
{
    _player.Move(collision.Result.MinimumTranslationVector);
}
```

`collision.Result.MinimumTranslationVector` moves the queried actor out of the other actor. In this example, it moves `_player` out of the wall.

If you only need overlap state, use `Intersects(...)` on the shapes directly instead of a result-returning query.

## Step 5: Rebuild Dynamic Layers After Movement

`CollisionWorld2D` does not have an `Update` method. If your actors move, update their `Shape` values and then rebuild any dynamic broadphase layers before querying again.

For example:

```cs
protected override void Update(GameTime gameTime)
{
    Vector2 movement = Vector2.Zero;

    if (Keyboard.GetState().IsKeyDown(Keys.Right))
        movement.X += 2f;

    if (Keyboard.GetState().IsKeyDown(Keys.Left))
        movement.X -= 2f;

    if (Keyboard.GetState().IsKeyDown(Keys.Down))
        movement.Y += 2f;

    if (Keyboard.GetState().IsKeyDown(Keys.Up))
        movement.Y -= 2f;

    _player.Move(movement);

    foreach (Layer layer in _collisionWorld.Layers.Values)
        layer.Reset();

    foreach (CollisionEvent2D collision in _collisionWorld.QueryCollisions(_player, "walls"))
        _player.Move(collision.Result.MinimumTranslationVector);

    base.Update(gameTime);
}
```

`Layer.Reset()` rebuilds the broadphase when `Layer.IsDynamic` is `true`, which is the default.

## Querying Pairs

If you want all colliding pairs between two layers, use `QueryCollisionPairs(...)`:

```cs
foreach (CollisionPair2D pair in _collisionWorld.QueryCollisionPairs(
    CollisionWorld2D.DefaultLayerName,
    "walls"))
{
    CollisionResult2D playerResult = pair.FirstResult;
    CollisionResult2D wallResult = pair.SecondResult;
}
```

This is useful when you want one pass over all collisions in a layer pair instead of querying actor by actor.

## Choosing Shapes

`CollisionShape2D` can wrap:

- `BoundingBox2D`
- `BoundingCircle2D`
- `OrientedBoundingBox2D`
- `BoundingCapsule2D`
- `BoundingPolygon2D`

For many gameplay collision systems, `BoundingBox2D` is the simplest place to start.

:::warning
`CollisionShape2D.Intersects(...)` supports more shape pairs than `CollisionShape2D.TryGetCollision(...)`. If you need `CollisionResult2D`, verify that your shape pair supports result-returning queries in the current preview.
:::

## What's Next

- [Collision Overview](./collision.md) for the architecture and API roles
- [Migrating from 5.5.1](./migration.md) if you are upgrading from the old `IShapeF`-based system
- [2D Geometry](./2d-geometry.md) for the low-level bounding volume and `Collision2D` reference
