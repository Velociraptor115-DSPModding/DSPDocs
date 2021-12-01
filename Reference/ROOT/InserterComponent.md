```csharp
public void InternalUpdateNoAnim(PlanetFactory factory, int[][] needsPool, float power);
public void InternalUpdate(PlanetFactory factory, int[][] needsPool, AnimData[] animPool, float power);
```

The only difference between the two methods is that the `NoAnim` version skips updating the sorter animation as well as `PlanetFactory.entitySignPool` for the power/disconnection signs, etc.

Looking at `InternalUpdateNoAnim`,
* If the power is less than 10%, it returns without doing anything
* The rest of the method depends on the current `stage` of the inserter
  * `Returning` (The sorter is moving back to `pickTarget`)
    * Increment `time` by `power * speed`
    * If `time` exceeds `stt`, go into `Picking` mode and reset `time` to 0
  * `Sending` (The sorter is moving from `pickTarget` to `insertTarget`)
    * Increment `time` by `power * speed`
    * If `time` exceeds `stt`, go into `Inserting` mode and rewind `time` by `stt`, so the extra work is considered as part of the next stage
    * In case there's no `itemId` - which could happen if someone picked items out of the sorter manually - go directly into `Returning` mode and set `time` to be `stt - time`, which sort of sets `time` in reverse
  * `Picking`
    * If either the `pickTarget` or `insertTarget` doesn't exist, return without doing anything
    * If `itemId` is 0 (meaning there's nothing picked yet)
      * If `careNeeds` (meaning the `insertTarget` is a building which requires specific input items)
        * Decrement `idleTick`. If `idleTick` <= 0
          * If `needsPool` is available and atleast one need is present
            * Try picking an item which matches the `filter` and needs from the `pickTarget`
              * If successful, update the `itemId`, `stackCount` and reset `time` to 0
              * If not, set `idleTick` to 10
      * Otherwise (probably outputting to belt)
        * Try picking an item which matches the `filter` and needs from the `pickTarget`
          * If successful, update the `itemId`, `stackCount` and reset `time` to 0
    * if `stackCount` < `stackSize`, do pretty much the same as before (the "if `itemId` is 0" case), but don't bother updating `itemId`
    * If `itemId` <= 0 (couldn't pick anything), reset `time` to 0 and return
    * Otherwise,
      * Increment `time` by `speed`
      * If `stackCount` is equal to `stackSize` or if `time` exceeds sorter `delay` (this is introduced for a stacking sorter), increment `time` again by `power * speed` and go into `Sending` stage
  * `Inserting`
    * If the `insertTarget` exists
      * If there are no more items in the sorter
        * Set `itemId`, `stackCount` to 0, increment `time` by `power * speed` and go into `Returning` stage
      * Decrement `idleTick`. If `idleTick` <= 0
        * If `careNeeds` and the `insertTarget` doesn't have a need, set `idleTick` to 10 and return
        * Try inserting into the `insertTarget`
          * If successful, decrement `stackCount`
          * If there are no more items in sorter
            * Set `itemId`, `stackCount` to 0, increment `time` by `power * speed` and go into `Returning` stage
