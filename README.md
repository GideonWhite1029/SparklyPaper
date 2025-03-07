<p align="center">
<img src="sparklypaper.png">
</p>

<p align="center">
<h1 align="center">✨ SparklyPaper ✨</h1>
</p>

![Minecraft Version](https://badgen.now.sh/badge/minecraft%20version/1.20.6/blue) ![Blazing Fast](https://badgen.now.sh/badge/speed/blazing%20%F0%9F%94%A5/green)

SparklyPower's Paper fork, making large servers snappier with high-performance optimizations and improvements! Focused on performance improvements for Survival servers with high player counts.

Our fork has handmade patches to add and optimize some of the things that we have in our server, with some cherry-picked patches from other forks.

## Features

This does not include all patches included in SparklyPaper, only the patches handmade for SparklyPaper! To see all patches, check out the ["patches" directory](patches).

SparklyPaper's config file is `sparklypaper.yml`, the file is, by default, placed on the root of your server.

* Skip `distanceToSqr` call in `ServerEntity#sendChanges` if the delta movement hasn't changed
  * The `distanceToSqr` call is a bit expensive, so avoiding it is pretty nice, around ~15% calls are skipped with this check. Currently, we only check if both Vec3 objects have the same identity, that means, if they are literally the same object. (that works because Minecraft's code reuses the Vec3 object when caching the current delta movement)
* Skip `MapItem#update()` if the map does not have the default `CraftMapRenderer` present
  * By default, maps, even those with custom renderers, fetch the world data to update the map data. With this change, "image in map" maps that have removed the default `CraftMapRenderer` can avoid these hefty updates, without requiring the map to be locked, which some old map plugins may not do.
  * This has the disadvantage that the vanilla map data will never be updated while the CraftMapRenderer is not present, so if you readd the default renderer, the server will need to update the map data, but that's not a huuuge problem, after all, it is a very rare circumstance that you may need the map data to always be up-to-date when you have a custom renderer on the map.
  * But still, if you made your own custom "image on map" plugin, don't forget to `mapView.isLocked = true` to get the same performance benefits in vanilla Paper!
* Fix concurrency issues when using `imageToBytes` in multiple threads
  * Useful if one of your plugins is parallelizng map creation on server startup
* Skip dirty stats copy when requesting player stats
  * There's literally only one `getDirty` call. Because the map was only retrieved once, we don't actually need to create a copy of the map just to iterate it, we can just access it directly and clear it manually after use.
* ~~Avoid unnecessary `ItemFrame#getItem()` calls~~
  * ~~When ticking an item frame, on each tick, it checks if the item on the item frame is a map and, if it is, it adds the map to be carried by the entity player~~
  * ~~However, the `getItem()` call is a bit expensive, especially because this is only really used if the item in the item frame is a map~~
  * ~~We can avoid this call by checking if the `cachedMapId` is not null, if it is, then we get the item in the item frame, if not, then we ignore the `getItem()` call.~~
  * Replaced by [Warriorrrr's "Rewrite framed map tracker ticking" patch (Paper #9605)](https://github.com/PaperMC/Paper/pull/9605)
* Optimize `EntityScheduler`'s `executeTick`
  * On each tick, Paper runs `EntityScheduler`'s `executeTick` of each entity. This is a bit expensive, due to `ArrayDeque`'s `size()` call because it ain't a simple "get the current queue size" function, due to the thread checks, and because it needs to iterate all server entities to tick them.
  * To avoid those hefty calls, instead of iterating all entities in all worlds, we use a set to track which entities have scheduled tasks that we need to tick. When a task is scheduled, we add the entity to the set, when all entity tasks are executed, the entity is removed from the set. We don't need to care about entities that do not have any tasks scheduled, even if the scheduler has a `tickCount`, because `tickCount` is relative and not bound to the server's current tick, so it doesn't matter if we don't increase it.
  * Most entities won't have any scheduled tasks, so this is a nice performance bonus, even if you have plugins that do use the entity scheduler because, for 99,99% of use cases, you aren't going to create tasks for all entities in your server. With this change, the `executeTick` loop in `tickChildren` CPU % usage drops from 7.60% to 0.00% (!!!) in a server with ~15k entities! Sweet!
    * Yeah, I know... "but you are cheating! the loop doesn't show up in the profiler because you replaced the loop with a for each!" and you are right! Here's a comparison of the `tickChildren` function CPU usage % between vanilla Paper and SparklyPaper, removing all other functions from the profiler result: 7.70% vs 0.02% (wow, such improvement, low mspt)
  * Of course, this doesn't mean that `ArrayDeque#size()` is slow! It is mostly that because the `executeTick` function is called each tick for each entity, it would be better for us to avoid as many useless calls as possible.
* Blazingly Simple Farm Checks
  * Changes Minecraft's farm checks for crops, stem blocks, and farm lands to be simpler and less resource intensive
  * If a farm land is moisturised, the farm land won't check if there's water nearby to avoid intensive block checks. Now, instead of the farm land checking for moisture, the crops themselves will check when attempting to grow, this way, farms with fully grown crops won't cause lag.
  * The growth speed of crops and stems are now fixed based on if the block below them is moist or not, instead of doing vanilla's behavior of "check all blocks nearby to see if at least one of them is moist" and "if the blocks nearby are of the same time, make them grow slower".
    * In my opinion: Who cares about the vanilla behavior lol, most players only care about farm land + crop = crop go brrrr
  * Another optimization is that crop behavior can be changed to skip from age zero to the last age directly, while still keeping the original growth duration of the crop. This way, useless block updates due to crop growth can be avoided!
  * (Incompatible with Paper's Dry and Wet Farmland custom tick rates)
* Spooky month optimizations
  * The quintessential patch that other performance forks also have for... some reason??? I thought that this optimization was too funny to not do it in SparklyPaper.
  * Caches when Bat's spooky season starts and ends, and when Skeleton and Zombies halloween starts and ends. The epoch is updated every 90 days. If your server is running for 90+ days straight without restarts, congratulations!
  * Avoids unnecessary date checks, even tho that this shouldn't really improve performance that much... unless you have a lot of bats/zombies/skeletons spawning.
* Cache coordinate key used for nearby players when ticking chunks
  * The `getChunkKey(...)` call is a bit expensive, using 0.24% of CPU time with 19k chunks loaded.
  * So instead of paying the price on each tick, we pay the price when the chunk is loaded.
  * Which, if you think about it, is actually better, since we tick chunks more than we load chunks.
* Optimize `canSee(...)` checks
  * The `canSee(...)` checks is in a hot path (`ChunkMap#updatePlayers()`), invoked by each entity for each player on the server if they are in tracking range, so optimizing it is pretty nice.
  * First, we change the original `HashMap` to fastutil's `Object2ObjectOpenHashMap`, because fastutil's `containsKey` throughput is better.
  * Then, we add a `isEmpty()` check before attempting to check if the map contains something. This seems stupid, but it does seem that it improves the performance a bit, and it makes sense, `containsKey(...)` does not attempt to check the map size before attempting to check if the map contains the key.
* Fix `MC-117075`: TE Unload Lag Spike
  * Paper already has a patch that fixes the lag spike, however, we can reduce the lag even... further, beyond!
  * We replaced the `blockEntityTickers` list with a custom list based on fastutil's `ObjectArrayList` with a small yet huge change for us: A method that allows us to remove a list of indexes from the list.
    * This is WAY FASTER than using `removeAll` with a list of entries to be removed (which is what Paper currently does), because we don't need to calculate the identity of each block entity to be removed, and we can jump directly to where the search should begin, giving a performance boost for small removals (because we don't need to loop thru the entire list to find what element should be removed) and a performance boost for big removals (no need to calculate the identity of each block entity).
* Optimize `tickBlockEntities`
  * We cache the last `shouldTickBlocksAt` result, because the `shouldTickBlocksAt` is expensive because it requires pulling chunk holder info from an map for each block entity (even if the block entities are on the same chunk!) every single time. So, if the last chunk position is the same as our cached value, we use the last cached `shouldTickBlocksAt` result!
    * We could use a map for caching, but here's why this is way better than using a map: The block entity ticking list is sorted by chunks! Well, sort of... It is sorted by chunk when the chunk has been loaded, newly placed blocks will be appended to the end of the list until the chunk unloads and loads again.  Most block entities are things that players placed to be there for a long time anyway (like hoppers, etc)
    * But here's the thing: We don't care if we have a small performance penalty if the players have placed new block entities, the small performance hit of when a player placed new block entities is so small ('tis just a long comparsion after all), that the performance boost from already placed block entities is bigger, this helps a lot if your server has a lot of chunks with multiple block entities, and the block entities will be automatically sorted after the chunk is unloaded and loaded again, so it ain't that bad.
  * And finally, we also cache the chunk's coordinate key when creating the block entity, which is actually "free" because we just reuse the already cached chunk coordinate key from the chunk!
* Check how much MSPT (milliseconds per tick) each world is using in `/mspt`
  * Useful to figure out which worlds are lagging your server.
![Per World MSPT](docs/per-world-mspt.png)
* Parallel World Ticking
  * "mom can we have folia?" "we already have folia at home" folia at home: [Parallel World Ticking](docs/PARALLEL_WORLD_TICKING.md)

While we could cherry-pick *everything* from other forks, only patches that I can see and think "yeah, I can see how this would improve performance" or patches that target specific performance/feature pain points in our server are cherry-picked! In fact, some patches that are used in other forks [may be actually borked](docs/BORKED_PATCHES.md)...

## Upstreamed Optimizations & Features

These optimizations and features were originally in SparklyPaper, but now they are in Paper, yay! Thanks Paper team :3

* Lazily create `LootContext` for criterions (Merged in [Paper #9969](https://github.com/PaperMC/Paper/pull/9969))
  * For each player on each tick, enter block triggers are invoked, and these create loot contexts that are promptly thrown away since the trigger doesn't pass the predicate.
  * To avoid this, we now lazily create the LootContext if the criterion passes the predicate AND if any of the listener triggers require a loot context instance.
* Remove unnecessary durability check in `ItemStack#isSimilar(...)` (Merged in [Paper #9979](https://github.com/PaperMC/Paper/pull/9979))
  * I know that this looks like a stupid optimization that doesn't do anything, but hear me out: `getDurability()` ain't free, when you call `getDurability()`, it calls `getItemMeta()`, which then allocates a new `ItemMeta` or clones the current item's item meta.
  * However, this means that we are unnecessarily allocating useless `ItemMeta` objects if we are comparing two items, or one of the two items, that don't have any durability, and this impact can be noticed if one of your plugins has a `canHoldItem` function that checks if a inventory can hold an item.
  * To avoid this, we can just... not check for the item's durability! Don't worry, the durability of the item is checked when it checks if both item metas are equal.
  * This is a leftover from when checking for the item's durability was "free" because the durability was stored in the `ItemStack` itself, this [was changed in Minecraft 1.13](https://hub.spigotmc.org/stash/projects/SPIGOT/repos/bukkit/commits/f8b2086d60942eb2cd7ac25a2a1408cb790c222c#src/main/java/org/bukkit/inventory/ItemStack.java).
  * (The reason I found out that this had a performance impact was because the `getDurability()` was using 0.08ms each tick according to spark... yeah, sadly it ain't a super big crazy optimization, the performance impact would be bigger if you have more plugins using `isSimilar(...)` tho)
* Configurable Farm Land moisture tick rate when the block is already moisturised (Merged in [Paper #9968](https://github.com/PaperMC/Paper/pull/9968))
  * The `isNearWater` check is costly, especially if you have a lot of farm lands. If the block is already moistured, we can change the tick rate of it to avoid these expensive `isNearWater` checks.
  * (Incompatible with the Blazingly Simple Farm Checks feature)
  
We attempt to upstream everything that we know helps performance and makes the server go zoom, and not stuff that we only *hope* that it improves performance. I'm still learning after all, so some of my patches may be worthless and not good enough. :)

## Results

If you are curious about SparklyPaper's performance, check out [our results page](docs/RESULTS.md)!

## Support

Because this is a fork made for SparklyPower, we won't give support for any issues that may happen in your server when using SparklyPaper. We know that SparklyPaper may break some plugins, but unless we use these plugins on SparklyPower, we won't go out of our way to fix it!

If you only care about some of the patches included in SparklyPaper, it is better for you to [create your own fork](https://github.com/PaperMC/paperweight-examples) and cherry-pick the patches, this way you have full control of what patches you want to use in your server, and even create your own changes!

## Downloads

There are two kinds of builds: One with the Parallel World Ticking feature, and another without it. If you don't want to risk using a very experimental feature that may lead to server crashes and corruption, or if you aren't a developer that can't fix plugin issues related to the feature, then use the version without Parallel World Ticking! (We do run Parallel World Ticking in production @ SparklyPower tho, we live on the edge :3)

It is recommended to use a Mojang mapped (mojmap) version unless if you *really* have a reason (example: plugins that break on a mojmap JAR) to use a Spigot mapped (reobf) version. Paper, since 1.20.5, provides a mojmapped server JAR and remaps any class/field/Reflection access made by non-mojmap aware plugins, so things (hopefully!) shouldn't break.

* **SparklyPaper:** https://github.com/SparklyPower/SparklyPaper/actions/workflows/build.yml
* **SparklyPaper (without Parallel World Ticking):** https://github.com/SparklyPower/SparklyPaper/actions/workflows/build-without-pwt.yml

Click on a workflow run, scroll down to the Artifacts, and download!