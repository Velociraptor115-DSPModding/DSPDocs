```csharp
public uint InternalUpdate(float power, int[] productRegister, int[] consumeRegister);
```

The return value from this method is used to determine animation and sign state
The assembler stops working when supplied `power` is less than 10%
In case `power` is present, the rest of the code works similar to a state machine

Section 1
* If it's not replicating
  *  Check whether there are enough items to do one round of replication by comparing `served` and `requireCounts`
    * If not, set `time` to 0 and return 0
    * If yes, mark the `consumeRegister` and set `replicating` to true

Section 2
* If it's replicating (includes starting to replicate in this tick), but not outputting
  * advance `time` w.r.t. the `power` received
  * If `time` is gte the time required for the recipe, set `outputting` to true

Section 3
* If it's outputting (includes outputting set in this tick)
  * If the `produced` + `productCounts` for any output material exceeds the stack size, return 0
  * If not
    * Add the `productCounts` from the recipe to the `produced` stack
    * Record it in the `productRegister`
    * Set `outputting` to false
    * Set back `time` by `timeSpend`
      (This is done instead of resetting `time` to 0, so that the extra work carries over into the next tick as long as `served` meets `requireCounts`)
    * Set `replicating` to false
    * Now there's this weird thing where it does `Section 1` again. 🧐 Why?? It offsets the statistics cycle *sigh*
  

```csharp
public void UpdateNeeds()
```

This just updates the `needs` array based on the `served` exceeding the `requireCounts` * stacking amount
This is in turn used to update `PlanetFactory.entityNeeds`