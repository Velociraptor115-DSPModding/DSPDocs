# Planet-level Statistics

```csharp
public int[] productRegister;
public int[] consumeRegister;
```

These registers are expected to have a size of 12000 currently.
These are used to store the production and consumption counts of every item for the current GameTick
The registers are indexed with the `itemId` of items from `ItemProto`
These are used for calculations of the Production tab in Statistics Panel

```csharp
public long powerGenRegister;
public long powerConRegister;
public long powerDisRegister;
public long powerChaRegister;
```

These registers are used to store power generation, consumption, discharge and charge in joules (I think, need to verify) for the current GameTick
These are used for calculations of the Power tab in Statistics Panel

```csharp
public long hashRegister;
```

This register is used to store the hashes computed by Matrix labs for the current GameTick
These are used for calculations of the Research tab in Statistics Panel

```csharp
public long energyConsumption;
```

This is used to store the energy consumption on the planet over the course of the entire game.
This includes energy consumed by orbital collectors, which is not included anywhere else

```csharp
public bool itemChanged;
```

This is set false at the start of every GameTick and becomes true iff a new itemId is either produced or consumed on the planet for the current GameTick
This raises `OnItemChanged` when it is false at the end of a GameTick
`OnItemsChanged` is only subscribed to by the `UIStatisticsWindow` where it resets `UIStatisticsWindow.lastBeginIndex` to -1

```csharp
public int[] productIndices;
```

This is indexed with `itemId` and maps to the index of the referred item in the `productPool` array
This is expected to have a size of 12000 currently

```csharp
public ProductStat[] productPool;
public int productCapacity;
public int productCursor = 1;
```

This is the meat of the class, where it stores the item production and consumption statistics
The length of the array is dynamic and is expected to be reflected by `productCapacity`. It is initially set to 8
As with most of the self-managed arrays in DSP, they multiply the size by 2 whenever it isn't sufficient
The `productCursor` is expected to be `number of items in productPool + 1`
Will expand on the structure of the data in `productPool` below

```csharp
public PowerStat[] powerPool;
```

This is expected to be an array of 5 elements currently.  
0: Power Generation  
1: Power Consumption  
2: Power Charge  
3: Power Discharge  
4: Hashes Uploaded  

Will expand on the structure of the data in `powerPool` below

## *Pool data structure

The `ProductStat` and `PowerStat` classes have almost equivalent fields

| ProductStat      | PowerStat         |
|------------------|-------------------|
| int[7200] count  | long[3600] energy |
| int[12]   cursor | long[6] cursor    |
| int[14]   total  | long[7] total     |
| int       itemId |                   |

The `count` and `energy` are each multiple circular buffers, each of capacity 600

Before we look at the specific capacities of each array, let us look at the time window options for the statistics panel

1: 1 minute  
2: 10 minutes  
3: 1 hour  
4: 10 hours  
5: 100 hours  
x: Total (Slightly more nuanced option, will explain further at the end, ignore for now)

Now, the 600 entries in the circular buffers are used to plot the statistics graphs
Taking the first 5 entries above, let me rewrite as:

<pre>
0: every tick  
   (1 tick per entry * 600 entries = 600 ticks = 10 seconds)  
1: once every 6 ticks  
   (6 ticks per entry * 600 entries = 3600 ticks = 1 minute)  
2: once every 60 ticks (10 mintues)  
3: once every 3600 ticks (1 hour)  
4: once every 36000 ticks (10 hours)  
5: once every 360000 ticks (100 hours)  
</pre>

For `i` corresponding to 1-5 in the list above, 
* `cursor[i]` gives the current index in the corresponding `energy` / `count` arrays
* `i * 600` <= `cursor[i]` < `(i + 1) * 600`
* `total[i]` gives the moving sum of the values for the time window implied by `i`
* The `total` array has an extra element which is the overall sum without any time window

Now, the reason `ProductStat` has the capacities doubled is because it stores both production and consumption within the same instance.  
The first half corresponds to the production statistics and the second half corresponds to consumption
For eg.
* if `cursor[1]` is the current index for the "per 6 ticks" aggregated production data, the current index for the corresponding "per 6 ticks" aggregated consumption will be in `cursor[7]`
* if `total[1]` is the moving sum of production for the "1 minute" time window, the moving sum of consumption for the corresponding "1 minute" time window will be in `total[8]`
