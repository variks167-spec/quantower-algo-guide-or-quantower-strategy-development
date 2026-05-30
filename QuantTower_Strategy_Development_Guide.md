# Building Custom Strategies in QuantTower — A Complete Developer Guide

*Written from real-world experience building a footprint imbalance strategy with anchored VWAP on QuantTower v1.145.17 with Rithmic data feed.*

---

## 1. Introduction

QuantTower is a modern, professional trading platform with a powerful C# scripting API. Unlike NinjaTrader, which has its own built-in editor and proprietary scripting layer, QuantTower uses standard Visual Studio and standard C# — which means you get full IDE support, proper debugging, NuGet packages, and all the tooling you're already familiar with.

However, the documentation is sparse. This guide documents what actually works, including the pitfalls that aren't written down anywhere.

### Key differences from NinjaTrader

| Feature | NinjaTrader 8 | QuantTower |
|---|---|---|
| IDE | Built-in NinjaScript editor | Visual Studio 2022 |
| Language | C# with NT wrappers | Standard C# |
| Bar update hook | `OnBarUpdate()` | Event subscriptions |
| Tick hook | `Calculate.OnEachTick` | `Symbol.NewLast` event |
| Drawing | `Draw.Rectangle()` etc | GDI+ via `OnPaintChart()` |
| Orders | `EnterLong()` / `SetStopLoss()` | `Core.Instance.PlaceOrder()` |
| Footprint data | `VolumetricBarsType` | `VolumeAnalysisData.PriceLevels` |
| Symbol/Account | Inherited properties | `[InputParameter]` fields |

---

## 2. Setting Up the Environment

### Prerequisites
- QuantTower v1.145.17 or later (download from quantower.com directly, not Microsoft Store)
- Visual Studio 2022 Community (free) with **.NET desktop development** workload
- QuantTower Algo VS extension v0.0.36 or later

### Installing the VS Extension
1. Open Visual Studio → **Extensions → Manage Extensions**
2. Search "Quantower" in the Online tab
3. Download and install, restart VS when prompted

### Configuring the Extension Path
After installing the extension, VS needs to know where QuantTower is installed:

1. Go to **Tools → Options → Quantower Algo → General**
2. Click **Browse** next to the path field
3. Navigate to your QuantTower installation — typically `C:\Quantower\TradingPlatform\v1.145.17\`
4. Select `Starter.exe` and click OK

> **Note:** The path field may appear unclickable. Click **Set up my environment** first, then try again. If it still won't accept input, type the path directly into the File Name bar of the browse dialog.

### Finding the QuantTower Installation

QuantTower installs as a portable app. If you can't find it via File Explorer, use PowerShell:

```powershell
Get-ChildItem -Path "C:\Quantower" -Recurse -Filter "*.exe" | Select-Object FullName
```

The key files are:
- `C:\Quantower\TradingPlatform\v1.145.17\Starter.exe` — the main executable
- `C:\Quantower\TradingPlatform\v1.145.17\bin\TradingPlatform.BusinessLayer.dll` — the API DLL you reference in your project

---

## 3. Project Structure

### Creating a New Project
In Visual Studio: **File → New → Project** → search "Strategy" → select the QuantTower Strategy template.

### The .csproj File

Here is a complete, working `.csproj` for QuantTower v1.145.17:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8</TargetFramework>
    <LangVersion>latest</LangVersion>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <Platforms>AnyCPU</Platforms>
    <AlgoType>Strategy</AlgoType>
    <AssemblyName>YourStrategyName</AssemblyName>
    <RootNamespace>YourStrategyName</RootNamespace>
    <QuantowerInstallPath>C:\Quantower\TradingPlatform\v1.145.17\</QuantowerInstallPath>
    <StartAction>Program</StartAction>
    <StartProgram>C:\Quantower\TradingPlatform\v1.145.17\Console.StarterNew.exe</StartProgram>
    <StartArguments>--address 127.0.0.1 --port 51541</StartArguments>
  </PropertyGroup>

  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
    <OutputPath>C:\Quantower\TradingPlatform\v1.145.17\..\..\Settings\Scripts\Strategies\YourStrategyName</OutputPath>
  </PropertyGroup>

  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|AnyCPU'">
    <OutputPath>C:\Quantower\TradingPlatform\v1.145.17\..\..\Settings\Scripts\Strategies\YourStrategyName</OutputPath>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="System.Drawing.Common" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Reference Include="TradingPlatform.BusinessLayer">
      <HintPath>C:\Quantower\TradingPlatform\v1.145.17\bin\TradingPlatform.BusinessLayer.dll</HintPath>
      <Private>False</Private>
    </Reference>
  </ItemGroup>

  <!-- Auto-deploy to QT Scripts folders after every build -->
  <Target Name="PostBuild" AfterTargets="PostBuildEvent">
    <Copy SourceFiles="$(OutputPath)\$(AssemblyName).dll"
          DestinationFolder="C:\Quantower\TradingPlatform\v1.145.17\bin\Scripts\Strategies\"
          OverwriteReadOnlyFiles="true" />
    <Copy SourceFiles="$(OutputPath)\$(AssemblyName).dll"
          DestinationFolder="C:\Quantower\TradingPlatform\v1.145.17\bin\Scripts\Indicators\"
          OverwriteReadOnlyFiles="true" />
  </Target>
</Project>
```

The post-build event automatically copies your compiled DLL to both the Strategies and Indicators folders in QT after every build — no manual copying required.

### DLL Deployment Paths

QuantTower reads scripts from two separate locations:
- **Strategies:** `C:\Quantower\TradingPlatform\v1.145.17\bin\Scripts\Strategies\`
- **Indicators:** `C:\Quantower\TradingPlatform\v1.145.17\bin\Scripts\Indicators\`

If your project contains both a Strategy and an Indicator class (compiled into the same DLL), copy it to both folders. QT scans them separately.

---

## 4. Critical Naming Rules

This is the most important section in the guide. Getting naming wrong causes silent crashes that are very hard to diagnose.

### Rule 1: Never use the same name for your namespace and your class

**Wrong:**
```csharp
namespace MyStrategy
{
    public class MyStrategy : Strategy { ... }  // CRASH — null key in ScriptManager
}
```

**Correct:**
```csharp
namespace FootprintStrategy
{
    public class FootprintStagesQT : Strategy { ... }
}
```

QT's `ScriptManager` uses the class name as a dictionary key. When the namespace and class name are identical, the key resolution returns null and the strategy silently fails to register — the + button and strategy selector will appear completely unresponsive with no error shown in the UI.

### Rule 2: Always set Name and Description in the constructor

```csharp
public class FootprintStagesQT : Strategy
{
    public FootprintStagesQT()
        : base()
    {
        this.Name        = "Footprint Stages QT";
        this.Description = "Your strategy description here";
    }
}
```

Without this, `ScriptManager` gets a null name and crashes silently.

### Rule 3: Mark helper classes as internal

Any class that is not a Strategy or Indicator should be marked `internal`, not `public`. QT's ScriptManager scans every `public` class in your DLL and tries to register it. If it finds a public class with no `Name` property it will crash:

```csharp
internal class ImbalanceZone { ... }      // correct
internal class TradeRecord { ... }        // correct
internal static class SharedState { ... } // correct
public class FootprintStagesQT : Strategy { ... } // only this one is public
```

---

## 5. Strategy Architecture

### The Correct Pattern

Unlike NinjaTrader's `OnBarUpdate()`, QuantTower strategies use event subscriptions:

```csharp
public class MyStrategy : Strategy
{
    // Symbol and Account MUST be InputParameters — they are NOT inherited
    [InputParameter("Symbol", 0)]
    public Symbol symbol { get; set; }

    [InputParameter("Account", 1)]
    public Account account { get; set; }

    private HistoricalData historicalData;

    public MyStrategy()
        : base()
    {
        this.Name        = "My Strategy";
        this.Description = "Strategy description";
    }

    protected override void OnRun()
    {
        // Fetch 1-minute bar history
        this.historicalData = this.symbol.GetHistory(
            Period.MIN1,
            HistoryType.Last,
            Core.Instance.TimeUtils.DateTimeUtcNow.AddDays(-90));

        // Subscribe to bar close events
        this.historicalData.NewHistoryItem += OnNewHistoryItem;

        // Subscribe to live tick events
        this.symbol.NewLast += OnNewLast;
    }

    // Fires once per completed bar (equivalent to NinjaTrader's IsFirstTickOfBar)
    private void OnNewHistoryItem(object sender, HistoryEventArgs e)
    {
        var bar = this.historicalData[0] as HistoryItemBar;
        if (bar == null) return;

        // Your closed-bar logic here
    }

    // Fires on every incoming tick (equivalent to NinjaTrader's Calculate.OnEachTick)
    private void OnNewLast(Symbol symbol, Last last)
    {
        double livePrice = last.Price;
        // Your intrabar logic here
    }

    protected override void OnStop()
    {
        this.historicalData.NewHistoryItem -= OnNewHistoryItem;
        this.symbol.NewLast -= OnNewLast;
    }

    protected override void OnRemove()
    {
        this.historicalData?.Dispose();
    }
}
```

### Key Differences from NinjaTrader

**Symbol and Account are InputParameters:**
In NinjaTrader, `this.Symbol` and `this.Account` are inherited from the Strategy base class. In QuantTower they must be declared as `[InputParameter]` fields. The user selects them in the Strategy Runner UI before pressing Run.

**`OnUpdate()` is Indicator-only:**
`protected override void OnUpdate(UpdateArgs args)` exists only on `Indicator`, not `Strategy`. Strategies use event subscriptions exclusively.

**History is fetched explicitly:**
Call `this.symbol.GetHistory()` in `OnRun()` to get historical bars. The result is an `IHistoricalData` object you subscribe events on.

---

## 6. Input Parameters

```csharp
// Boolean toggle
[InputParameter("Enable Trade Tracking", 0)]
public bool EnableTradeTracking { get; set; } = true;

// Integer with min/max/step
[InputParameter("Stacked Levels", 1, 1, 10, 1, 0)]
public int StackedLevels { get; set; } = 3;

// Double with decimal places
[InputParameter("Imbalance Ratio", 2, 1.0, 10.0, 0.1, 1)]
public double ImbalanceRatio { get; set; } = 3.0;
```

The `[InputParameter]` attribute signature is:
```
[InputParameter("Display Name", order, min, max, increment, decimalPlaces)]
```

---

## 7. Placing Orders

Orders go through `Core.Instance.PlaceOrder()` with a `PlaceOrderRequestParameters` object. Stop loss and take profit are attached directly to the order via `SlTpHolder`:

```csharp
var result = Core.Instance.PlaceOrder(new PlaceOrderRequestParameters
{
    Symbol      = this.symbol,
    Account     = this.account,
    Side        = Side.Buy,
    OrderTypeId = OrderType.Market,
    Quantity    = 1,
    StopLoss    = SlTpHolder.CreateSL(5.0, PriceMeasurement.Offset),  // 5 points
    TakeProfit  = SlTpHolder.CreateTP(10.0, PriceMeasurement.Offset), // 10 points
    Comment     = "LongEntry"
});

if (result.Status != TradingOperationResultStatus.Success)
    Log($"Order failed: {result.Message}", StrategyLoggingLevel.Error);
```

This is cleaner than NinjaTrader's separate `SetStopLoss()` / `SetProfitTarget()` calls — everything goes on the order in one shot.

---

## 8. Volume Analysis (Footprint Data)

### Accessing Per-Price Bid/Ask Volume

To access bid and ask volume at each price level (footprint/cluster data), implement `IVolumeAnalysisIndicator` on your **Indicator** class:

```csharp
public class MyIndicator : Indicator, IVolumeAnalysisIndicator
{
    public bool IsRequirePriceLevelsCalculation => true;

    public void VolumeAnalysisData_Loaded()
    {
        // Called once historical volume data is fully loaded
        // Safe to start scanning bars after this fires
    }

    protected override void OnUpdate(UpdateArgs args)
    {
        var vaData = this.HistoricalData[0].VolumeAnalysisData;

        if (vaData?.PriceLevels == null) return;

        // BuyVolume  = aggressive buys (ask side) — equivalent to NT GetAskVolumeForPrice
        // SellVolume = aggressive sells (bid side) — equivalent to NT GetBidVolumeForPrice
        foreach (var kvp in vaData.PriceLevels)
        {
            double price     = kvp.Key;
            double askVolume = kvp.Value.BuyVolume;
            double bidVolume = kvp.Value.SellVolume;
        }
    }
}
```

### Important Limitations

- `IVolumeAnalysisIndicator` works on **Indicator** classes only — implementing it on a Strategy class does not trigger `VolumeAnalysisData_Loaded()`
- Volume analysis data is **not available in the backtester** — `PriceLevels` will be null during backtesting regardless of the interface
- Volume analysis only works with live or paper trading connected to a real data feed (Rithmic, CQG, etc.)

### Stacked Imbalance Detection

The diagonal imbalance check that matches QuantTower's native footprint chart rendering:

```csharp
double tickSize = this.Symbol.TickSize;

for (double price = bar.Low; price <= bar.High; price += tickSize)
{
    double p  = Math.Round(price / tickSize) * tickSize;           // always round to tick
    double pB = Math.Round((price - tickSize) / tickSize) * tickSize;
    double pA = Math.Round((price + tickSize) / tickSize) * tickSize;

    double askVol      = vaData.PriceLevels.TryGetValue(p,  out var lv)  ? lv.BuyVolume   : 0;
    double bidVolBelow = vaData.PriceLevels.TryGetValue(pB, out var lvB) ? lvB.SellVolume : 0;

    // Bullish imbalance: ask at this level >> bid one tick below
    bool bullishImbalance = bidVolBelow > 0 && askVol >= bidVolBelow * ratio;
}
```

> **Always round price levels to the nearest tick before dictionary lookup.** Floating point arithmetic on price levels causes key misses that silently return zero volume.

---

## 9. Custom Indicator Drawing

### Accessing the Graphics Object

```csharp
public class MyIndicator : Indicator
{
    public MyIndicator()
        : base()
    {
        this.Name           = "My Indicator";
        this.IsOverlay      = true;
        this.SeparateWindow = false;
    }

    public override void OnPaintChart(PaintChartEventArgs args)
    {
        Graphics  gr        = args.Graphics;
        Rectangle panel     = args.Rectangle;

        // Coordinate converter — price/time to pixel
        var converter = this.CurrentChart?.Windows?[args.WindowIndex]?.CoordinatesConverter;
        if (converter == null) return;
    }
}
```

### Coordinate Conversion

```csharp
// Price → Y pixel
float y = (float)converter.GetChartY(price);

// DateTime → X pixel
float x = (float)converter.GetChartX(dateTime);
```

### Drawing Primitives

```csharp
// Horizontal line at a price level
gr.DrawLine(new Pen(Color.DodgerBlue, 2), panel.Left, y, panel.Right, y);

// Filled rectangle (zone)
gr.FillRectangle(
    new SolidBrush(Color.FromArgb(40, 0, 255, 0)),  // 40/255 opacity green
    xStart, yTop, width, height);

// Text label
gr.DrawString("VWAP 5250.25", new Font("Arial", 10), Brushes.White, x, y);
```

### Visible Bar Range

Only draw what's visible on screen — don't iterate all bars:

```csharp
int leftBar  = args.LeftVisibleBarIndex;
int rightBar = args.RightVisibleBarIndex;
```

---

## 10. Strategy + Indicator Communication (Shared State Pattern)

When you want a Strategy to drive an Indicator's visuals, use a static class as a shared memory bridge. Since both classes compile into the same DLL and are loaded in the same AppDomain, static state is shared between them automatically:

```csharp
// In a shared file (or at the top of either file)
internal static class SharedState
{
    public static readonly List<ZoneData> Zones = new List<ZoneData>();
    public static double VWAP = double.NaN;
    public static readonly object Lock = new object();
}

// Strategy writes
lock (SharedState.Lock)
    SharedState.VWAP = anchoredVWAP;

// Indicator reads in OnPaintChart
lock (SharedState.Lock)
{
    float y = (float)converter.GetChartY(SharedState.VWAP);
    gr.DrawLine(vwapPen, panel.Left, y, panel.Right, y);
}
```

Always use a lock object for thread safety — the strategy writes from the data thread and the indicator reads from the UI thread.

---

## 11. Logging

```csharp
Log("Strategy started", StrategyLoggingLevel.Info);
Log($"VWAP: {vwap:F2}", StrategyLoggingLevel.Trading);
Log($"Order failed: {result.Message}", StrategyLoggingLevel.Error);
```

Logs appear in the Strategies Manager panel's Logs tab and in:
```
C:\Quantower\Logs\Serilog\YYYYMMDD.slog
```

The slog files are JSON lines format and can be searched with PowerShell:
```powershell
Select-String -Path "C:\Quantower\Logs\Serilog\*.slog" -Pattern "YourSearchTerm" | Select-Object -Last 20
```

---

## 12. Common Pitfalls

### Pitfall 1: Namespace = Class name → null key crash
**Symptom:** + button in Strategies Manager does nothing, strategy never appears in selector.
**Cause:** `namespace MyStrategy { public class MyStrategy : Strategy }` — identical names cause ScriptManager to return a null dictionary key.
**Fix:** Use different names. E.g. namespace `FootprintStrategy`, class `FootprintStagesQT`.

### Pitfall 2: No constructor with this.Name set
**Symptom:** Same as above — strategy silently fails to register.
**Fix:** Always add a constructor that sets `this.Name` and `this.Description`.

### Pitfall 3: Public helper classes crash ScriptManager
**Symptom:** Strategy never loads, null key errors in slog.
**Fix:** Mark all non-Strategy/Indicator classes as `internal`.

### Pitfall 4: IVolumeAnalysisIndicator on Strategy doesn't work
**Symptom:** `VolumeAnalysisData_Loaded()` never fires, `volumeDataReady` stays false.
**Fix:** Implement `IVolumeAnalysisIndicator` on an Indicator class, not a Strategy class.

### Pitfall 5: Volume analysis null in backtester
**Symptom:** `bar.VolumeAnalysisData.PriceLevels` is null during backtesting.
**Fix:** Volume analysis only works in live/paper mode with a real data feed. Use a fallback or skip detection during backtesting.

### Pitfall 6: DLL deploying to wrong location
**Symptom:** Changes don't appear in QT despite successful builds.
**Fix:** Check the `ClaudeImbalance ->` line in the build output for the actual output path. Use the post-build event in the csproj to auto-deploy to both Scripts\Strategies\ and Scripts\Indicators\.

### Pitfall 7: Floating point price level key misses
**Symptom:** `PriceLevels[price]` returns nothing even when volume exists.
**Fix:** Always round: `Math.Round(price / tickSize) * tickSize` before dictionary lookup.

### Pitfall 8: OnUpdate doesn't exist on Strategy
**Symptom:** Compiler error `no suitable method found to override`.
**Fix:** `OnUpdate(UpdateArgs args)` is Indicator-only. Use `historicalData.NewHistoryItem` and `symbol.NewLast` event subscriptions in strategies.

---

## 13. Quick Reference

### Strategy skeleton
```csharp
namespace YourNamespace  // NEVER same as class name
{
    public class YourStrategy : Strategy
    {
        [InputParameter("Symbol", 0)] public Symbol symbol { get; set; }
        [InputParameter("Account", 1)] public Account account { get; set; }

        private HistoricalData historicalData;

        public YourStrategy() : base()
        {
            this.Name        = "Your Strategy";
            this.Description = "Description";
        }

        protected override void OnRun()
        {
            this.historicalData = this.symbol.GetHistory(Period.MIN1, HistoryType.Last,
                Core.Instance.TimeUtils.DateTimeUtcNow.AddDays(-90));
            this.historicalData.NewHistoryItem += OnBar;
            this.symbol.NewLast += OnTick;
        }

        private void OnBar(object sender, HistoryEventArgs e)
        {
            var bar = this.historicalData[0] as HistoryItemBar;
        }

        private void OnTick(Symbol s, Last last) { }

        protected override void OnStop()
        {
            this.historicalData.NewHistoryItem -= OnBar;
            this.symbol.NewLast -= OnTick;
        }

        protected override void OnRemove() => this.historicalData?.Dispose();
    }
}
```

### Indicator skeleton with drawing
```csharp
public class YourIndicator : Indicator
{
    public YourIndicator() : base()
    {
        this.Name           = "Your Indicator";
        this.IsOverlay      = true;
        this.SeparateWindow = false;
    }

    protected override void OnInit() { }
    protected override void OnUpdate(UpdateArgs args) { }

    public override void OnPaintChart(PaintChartEventArgs args)
    {
        var gr        = args.Graphics;
        var panel     = args.Rectangle;
        var converter = this.CurrentChart?.Windows?[args.WindowIndex]?.CoordinatesConverter;
        if (converter == null) return;

        float y = (float)converter.GetChartY(somePrice);
        gr.DrawLine(Pens.Blue, panel.Left, y, panel.Right, y);
    }

    protected override void OnClear() { /* dispose GDI resources */ }
}
```
