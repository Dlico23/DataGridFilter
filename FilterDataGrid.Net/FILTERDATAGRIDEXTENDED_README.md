# FilterDataGrid - Enhanced Features

## Overview

The `FilterDataGrid` class now includes built-in support for programmatic control over filters, state preservation, and additional events. All features are integrated directly into the base class

## New Features (Integrated into Base Class)

### 1. **Get Current Filter State**
Access the current active filters without needing external files.

```csharp
// Get active filters as read-only collection
IReadOnlyList<FilterCommon> activeFilters = dataGrid.ActiveFilters;

// Get serializable filter state
FilterStateData filterState = dataGrid.GetCurrentFilterState();
```

### 2. **Get Filtered Items Collection**
Access only the filtered items, not the complete collection.

```csharp
// Get filtered items
IEnumerable filteredItems = dataGrid.FilteredItems;
int filteredCount = dataGrid.FilteredItemsCount;

// Convert to typed list
List<MyDataClass> typedList = filteredItems.Cast<MyDataClass>().ToList();
```

### 3. **Set Filters Programmatically**

#### Method 1: Using Saved Filter State
```csharp
// Load previously saved state
FilterStateData savedState = LoadFromDatabase();

// Apply the filter
dataGrid.ApplyFilterState(savedState);
```

#### Method 2: Using Filter Builder
```csharp
// Create and apply filters
dataGrid.CreateFilterBuilder()
    .AddFilter("Status", "Inactive", "Pending")  // Exclude these values
    .AddIncludeFilter("Category", "A", "B", "C") // Include only these values
    .AddFilter("Priority", 0)
    .Apply();
```

### 4. **Filter Change Events**
Know when filters are applied or removed.

```csharp
// Subscribe to filter changes
dataGrid.FilterCompleted += (sender, e) =>
{
    Console.WriteLine($"Filtered Items: {e.FilteredItemsCount}");
    Console.WriteLine($"Active Filters: {e.ActiveFilters.Count}");
    Console.WriteLine($"Timestamp: {e.Timestamp}");
    
    if (e.LastModifiedFilter != null)
    {
        Console.WriteLine($"Last Modified: {e.LastModifiedFilter.FieldName}");
    }
};
```

### 5. **State Preservation on ItemsSource Change**
Automatically maintain filter, selection, and scroll position when updating data.

```csharp
// Enable state preservation (enabled by default)
dataGrid.PreserveStateOnItemsSourceChange = true;

// Subscribe to events
dataGrid.ItemsSourceChanging += (sender, e) =>
{
    // Optionally cancel preservation
    if (someCondition)
        e.Cancel = true;
};

dataGrid.ItemsSourceChangedComplete += (sender, e) =>
{
    if (e.StateRestored)
        Console.WriteLine("Filter and view state restored!");
};

// Update data - filter, selection, and scroll will be preserved
dataGrid.ItemsSource = GetUpdatedData();
```

## API Reference

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `ActiveFilters` | `IReadOnlyList<FilterCommon>` | Gets current active filters |
| `FilteredItems` | `IEnumerable` | Gets the filtered items collection |
| `FilteredItemsCount` | `int` | Gets count of filtered items |
| `PreserveStateOnItemsSourceChange` | `bool` | Enable/disable state preservation (default: true) |

### Methods

| Method | Description |
|--------|-------------|
| `GetCurrentFilterState()` | Returns serializable filter state |
| `ApplyFilterState(FilterStateData)` | Applies filters from state data |
| `CreateFilterBuilder()` | Creates a filter builder for programmatic filtering |
| `RemoveFilters()` | Removes all filters |
| `RefreshFilter()` | Refreshes filter without changing state |

### Events

| Event | EventArgs | Description |
|-------|-----------|-------------|
| `FilterCompleted` | `FilterCompletedEventArgs` | Raised when filter operation completes |
| `ItemsSourceChanging` | `ItemsSourceChangingEventArgs` | Raised before ItemsSource changes (cancellable) |
| `ItemsSourceChangedComplete` | `ItemsSourceChangedEventArgs` | Raised after ItemsSource changed |

### FilterBuilder Methods

| Method | Description |
|--------|-------------|
| `AddFilter(fieldName, ...excludedValues)` | Adds filter to exclude specific values |
| `AddIncludeFilter(fieldName, ...includedValues)` | Adds filter to include only specific values |
| `Apply()` | Applies all filters |
| `Clear()` | Clears all filters from builder |

## Common Use Cases

### 1. Auto-Save Filter State
```csharp
dataGrid.FilterCompleted += (s, e) =>
{
    FilterStateData state = dataGrid.GetCurrentFilterState();
    SaveToDatabase(state);
};
```

### 2. Export Filtered Data
```csharp
void ExportFilteredData()
{
    var filtered = dataGrid.FilteredItems.Cast<MyDataClass>().ToList();
    ExportToCsv(filtered);
}
```

### 3. Refresh Data Without Losing Filter
```csharp
void RefreshData()
{
    // Just update ItemsSource - filter will be maintained
    dataGrid.ItemsSource = LoadFreshDataFromDatabase();
}
```

### 4. Complex Programmatic Filtering
```csharp
void ApplyBusinessRuleFilter()
{
    var builder = dataGrid.CreateFilterBuilder();
    
    // Exclude closed items
    builder.AddFilter("Status", "Closed", "Archived");
    
    // Only high priority
    builder.AddIncludeFilter("Priority", 1, 2);
    
    // Exclude old items
    var oldItems = dataGrid.Items.Cast<MyDataClass>()
        .Where(x => x.Date < DateTime.Now.AddMonths(-6))
        .Select(x => (object)x.Date)
        .ToArray();
    builder.AddFilter("Date", oldItems);
    
    builder.Apply();
}
```