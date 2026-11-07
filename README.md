# SpreadTask

> Walk large collections across multiple ticks — never freeze the main thread again.

A small, dependency-free utility for **PocketMine-MP** plugins that processes
large lists incrementally over server ticks instead of all at once.

## The Problem

PocketMine runs everything on a single main thread with a **50ms tick budget**.
A loop over 50,000 blocks, players, or entities will block that thread, freeze
the server, and cause lag spikes that players feel immediately.

## The Solution

`SpreadTask` splits the work into small batches and visits only `N` entries per
tick. The server stays responsive, players keep playing, and the job still gets
done — just a few ticks later.

```php
SpreadTask::spread(
    $this,           // your PluginBase
    $blocks,         // any iterable (array, Traversable, Generator)
    500,             // max entries per tick
    function (Vector3 $pos): void {
        $world->setBlock($pos, VanillaBlocks::AIR());
    },
    function (): void {
        $this->getLogger()->info("Reset complete!");
    }
);
```

## Features

- **Zero main-thread blocking** — configurable per-tick budget
- **Self-cancelling** — no leaked task handlers, ever
- **Safe by default** — snapshots the input list so later mutations can't corrupt an in-flight task
- **Framework-agnostic** — works with arrays, `Traversable`, and `Generator`
- **Completion callback** — run logic after the last entry is visited
- **Tiny surface area** — one class, one static method, no config

## Installation

Drop the `SpreadTask.php` file into your plugin's source tree, or require it via Composer:

```bash
composer require am/skyminez-useless
```

Then import and use:

```php
use AM\SkyMineZ\useless\SpreadTask;
```

## API

### `SpreadTask::spread()`

```php
SpreadTask::spread(
    PluginBase $plugin,
    iterable $list,
    int $perTick,
    callable $callback,        // fn(mixed $entry, int $index): void
    ?callable $onComplete = null,
    int $delay = 1
): ?TaskHandler
```

| Parameter    | Description                                                        |
|--------------|--------------------------------------------------------------------|
| `$plugin`    | Your plugin instance. The task is scheduled through its scheduler. |
| `$list`      | Any iterable. Materialised once into an array.                     |
| `$perTick`   | How many entries to visit per tick. Must be ≥ 1.                   |
| `$callback`  | Called for every entry with `($entry, $index)`.                    |
| `$onComplete`| Called once after the last entry. Optional.                        |
| `$delay`     | Initial delay in ticks before the first batch. Defaults to `1`.    |

Returns the `TaskHandler`, or `null` if the list is empty or the plugin is
disabled. When the list is empty, `$onComplete` still runs so callers stay in
sync.

## Choosing `perTick`

Rule of thumb — keep one tick's work under **~25ms** (half the tick budget):

```
perTick ≈ 25 / (time per operation in ms)
```

| Workload                     | Suggested `perTick` |
|------------------------------|---------------------|
| `setBlock` / `getBlock`      | 500 – 2000          |
| `sendMessage` to players     | 100 – 500           |
| Entity spawning / despawning | 50 – 200            |
| Anything touching I/O        | Don't. Use `AsyncTask`. |

## Examples

### Reset a large mine

```php
SpreadTask::spread(
    $this,
    $region->getBlocks(),
    1000,
    fn (Vector3 $pos) => $world->setBlock($pos, VanillaBlocks::STONE()),
    fn () => $player->sendMessage("§aMine reset!")
);
```

### Reward every online player safely

```php
SpreadTask::spread(
    $this,
    $recipients,
    50,
    function (Player $player): void {
        if ($player->isOnline()) {
            $player->giveItem(VanillaItems::DIAMOND()->setCount(10));
        }
    }
);
```

### Lazy generation with a Generator

```php
function positions(int $r): Generator {
    for ($x = -$r; $x <= $r; $x++)
        for ($z = -$r; $z <= $r; $z++)
            yield new Vector3($x, 64, $z);
}

SpreadTask::spread($this, positions(100), 500,
    fn (Vector3 $pos) => $world->setBlock($pos, VanillaBlocks::GRASS())
);
```

## What it is *not*

- **Not a replacement for `AsyncTask`.** Heavy I/O (files, database, network)
  should still run asynchronously. `SpreadTask` only breaks up CPU-bound work
  on the main thread.
- **Not for tiny lists.** If you're iterating fewer than a few hundred items,
  a plain `foreach` is faster and simpler.

## How it works

`SpreadTask` extends `pocketmine\scheduler\Task` and schedules itself as a
**delayed repeating task**. Each run processes up to `$perTick` entries, then
yields. When the index reaches the end, it cancels its own handler and fires
the optional completion callback.

The key detail: the per-tick counter is **local to each `onRun()` call**. If it
lived on the instance, the first tick would consume the entire budget and every
later tick would process zero entries — stalling the task forever.

## License

MIT — do whatever you want with it.
