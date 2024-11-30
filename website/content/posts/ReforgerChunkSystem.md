---
title: Reforger Chunk System
date: 2024-11-30
draft: false
tags:
  - Reforger
  - Spawn System
  - Enfusion Engine
  - TrainWreck
---

When designing the TrainWreck series of mods we created a generic spawn point. We can place this anywhere, such as building 
prefabs, which enables us to spawn units there! 

We enjoy procedural systems which means we need to programmatically spawn things based on various conditions. One of the 
main conditions is distance! If we're on the Everon map there's over 10,000 spawn points - thanks to overriding vanilla
prefabs to have our spawn points! 

Players don't want enemies spawning within eyesight, nor should they spawn across the map where nothing is happening!
So how would you go about this?

## Distance-based checks

You could iterate over the 10,000+ spawn points PER player to ensure we get a list of "valid spawn points".

**Pros:**
- Super simple to write

**Cons:**
- Expensive to run because we need to perform a distance calculation PER player to spawn point
- Can't run often (must be infrequent to avoid performance hiccups)

![worldwide checks](/images/reforger/worldwide.gif)

## Radius Checks

Can use the engine's built-in query entities in sphere approach. This still performs distance checks, however we'd also need
to perform secondary checks to ensure they are outside a _minimum spawn radius_

![radius checks](/images/reforger/radius.gif)

Although we experienced a slight improvement over the distance-based checks, it was still not performant!

## Chunk System

I hope everyone is familiar with Minecraft by now. Similar to how Minecraft's chunk system works I designed a way to isolate
groups of whatever I want into a grid system. 

![chunk checks](/images/reforger/chunks.gif)

It works in the following way:

- Register a grid manager to a grid size. For instance, 250 meters. This means we end up with grid squares of 250x250.
- Register something to the grid
  - Divide position by grid size. Let's say we're at 1000,1000. Divide by 250 is 4, 4

Now we can take a list of our players and perform lookups quite easily! 

- Divide player position by the grid size of our grid manager.
- Get the chunk the player is in
  - Want more? Want to grab neighboring chunks too! Too easy... just add and subtract the grid coordinates to grab neighboring chunks! No comparisons! No complex computations!

Okay, although this system was by far, orders of magnitude faster, it introduced a problem... Now we had enemies spawning 
everywhere around players - great, but also bad because it could be right next to you!

Do we really want to do a distance check against players before "spawning" on a spawn point that we considered "okay"? ... heck no.

This became the introduction of "dead zones", the idea that "these coordinates" are "too close" to a player. Thus, 
when grabbing spawn points we have two sets of configurable values. The chunk radius for getting spawn points, and a "dead chunk radius"
where anything in these chunks are off limits. When retrieving a list of valid items we provide the dead zone radius as well. 
Anything that falls within this dead zone is skipped, thus our list of items becomes something we don't need to do checks on!

For performance reasons we can store player coordinates, and anti spawn chunks in a hashmap. At a configurable interval we can "monitor" those 
positions and update as appropriate. We only need to recalculate certain things when player positions changed, like spawn points!

This system was such a performance increase from former versions that it allowed us to check every 2 seconds where players were and spawn units 
around them appropriately! IronMist and myself would be cruising down a road blowing through towns. As we'd be approaching new areas we'd
be encountering enemies! Former systems didn't allow this. It would take a while before enemies would start spawning.

Since we have a hashmap of chunks where players are, and where things can spawn... we can easily unload chunks as well! Garbage collecting
any unit that's out of action.


