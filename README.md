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
