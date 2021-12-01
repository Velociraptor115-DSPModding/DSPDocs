```csharp
public void UpdateInputSlots(CargoTraffic traffic, SignData[] signPool);
```

This loops through all the slots of the station.
For every input slot
  * If the slot has `counter` > 0,
    (which is another weird thing. The counter is only ever assigned to zero. However, Kremnev did find out that it acted like a skip delay)
    the `counter` is decremented by 1. Basically, `counter` acts as a measure of how many ticks this slot will skip working
  * Get the `CargoPath` of the belt attached to the slot
  * Call `CargoPath.TryPickItemAtRear(int[] needs, out int needIdx)`. This picks the item from the end of the belt if it matches any of the items on the station's needs array (This is updated in `StationComponent.UpdateNeeds()`, which is called before this method in a GameTick).
  * If an item is picked, call `StationComponent.InputItem(int itemId, int needIdx)`, which either updates `StationComponent.storage` by incrementing `StationStore.count` by 1 or increments `warperCount` by 1.
  * Also set the `storageIdx` of the particular slot (I have no idea why this would be relevant)
  * If `itemId > 0`, then update the icon for the particular belt's `entityId`

```csharp
public void UpdateOutputSlots(CargoTraffic traffic, SignData[] signPool);
```

This loops through all the slots of the station.
For every output slot
  * If the slot has `counter` > 0, the `counter` is decremented by 1.
  * Get the `CargoPath` of the belt attached to the slot
  * From the `storageIdx` assigned to the slot, figure out the `itemId` and if the storage slot's `count > 0`
  * Try inserting the item into the belt by calling `CargoPath.TryInsertItemAtHeadAndFillBlank(int itemId)`
  * If the item is inserted, decrement the `StationStore.count` by 1



Note:
 * I assume the direction of the slots is assigned when a belt is built into or out of a slot. When belts are removed, it is set to None (This needs verification)
 * The way the items are taken as input into the station has some interesting implications. For eg. if there are 6 belts feeding into the station for the same item and the (count, max) of the item is (999, 1000) before this method is called, the `needs` array will ask for the particular `itemId` and all 6 belts will feed it. Therefore it seems like it would have 1005 count at the end of this method (This needs verification)