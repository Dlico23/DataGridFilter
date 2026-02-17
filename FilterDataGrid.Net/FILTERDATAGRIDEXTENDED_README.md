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

// Subscribe to filtered items changes
dataGrid.FilteredItemsChanging += (sender, e) =>
{
    Console.WriteLine($"Filtered items changing from {e.CurrentFilteredItemsCount} to {e.UpcomingFilteredItemsCount}");
    // UpcomingFilteredItemsCount is -1 if unknown
};

dataGrid.FilteredItemsChanged += (sender, e) =>
{
    Console.WriteLine($"Filtered items changed from {e.PreviousFilteredItemsCount} to {e.CurrentFilteredItemsCount}");
    // Perfect place to update UI, charts, or statistics
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

dataGrid.ItemsSourceChanged += (sender, e) =>
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
| `ItemsSourceChanged` | `ItemsSourceChangedEventArgs` | Raised after ItemsSource changed |
| `FilteredItemsChanging` | `FilteredItemsChangingEventArgs` | Raised before filtered items change (before filter applied/removed) |
| `FilteredItemsChanged` | `FilteredItemsChangedEventArgs` | Raised after filtered items changed (after filter applied/removed) |
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

### 4. Real-Time Statistics Update
```csharp
// Update statistics whenever filtered items change
dataGrid.FilteredItemsChanged += (sender, e) =>
{
    // Update your UI with new statistics
    totalItemsLabel.Text = $"Total: {dataGrid.Items.Count}";
    filteredItemsLabel.Text = $"Filtered: {e.CurrentFilteredItemsCount}";
    hiddenItemsLabel.Text = $"Hidden: {dataGrid.Items.Count - e.CurrentFilteredItemsCount}";
    
    // Update chart or graph
    UpdateChart(dataGrid.FilteredItems);
    
    // Log the change
    Logger.Info($"Filter changed items from {e.PreviousFilteredItemsCount} to {e.CurrentFilteredItemsCount}");
};
```

### 5. Show Loading Indicator During Filter
```csharp
dataGrid.FilteredItemsChanging += (sender, e) =>
{
    // Show loading indicator
    loadingSpinner.Visibility = Visibility.Visible;
    statusText.Text = "Applying filter...";
};

dataGrid.FilteredItemsChanged += (sender, e) =>
{
    // Hide loading indicator
    loadingSpinner.Visibility = Visibility.Collapsed;
    statusText.Text = $"Showing {e.CurrentFilteredItemsCount} items";
};
```

### 6. Complex Programmatic Filtering
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

## Event Handler Examples

```csharp
private void DataGrid_FilterCompleted(object sender, FilterCompletedEventArgs e)
{
    // Update status bar
    statusText.Text = $"Showing {e.FilteredItemsCount} of {dataGrid.Items.Count} items";
    
    // Auto-save filter state
    SaveFilterState();
    
    // Log filter details
    foreach (FilterCommon filter in e.ActiveFilters)
    {
        System.Diagnostics.Debug.WriteLine(
            $"Filter: {filter.FieldName} - Excluded: {filter.PreviouslyFilteredItems.Count}");
    }
}

private void DataGrid_FilteredItemsChanging(object sender, FilteredItemsChangingEventArgs e)
{
    // Show loading indicator
    loadingSpinner.Visibility = Visibility.Visible;
    statusText.Text = "Applying filter...";
    
    System.Diagnostics.Debug.WriteLine(
        $"Items changing from {e.CurrentFilteredItemsCount} to {e.UpcomingFilteredItemsCount}");
}

private void DataGrid_FilteredItemsChanged(object sender, FilteredItemsChangedEventArgs e)
{
    // Hide loading indicator
    loadingSpinner.Visibility = Visibility.Collapsed;
    
    // Update UI
    totalItemsLabel.Text = $"Total: {dataGrid.Items.Count}";
    filteredItemsLabel.Text = $"Filtered: {e.CurrentFilteredItemsCount}";
    hiddenItemsLabel.Text = $"Hidden: {dataGrid.Items.Count - e.CurrentFilteredItemsCount}";
    
    statusText.Text = $"Showing {e.CurrentFilteredItemsCount} items";
    
    // Update chart or statistics
    UpdateDashboard(dataGrid.FilteredItems);
    
    System.Diagnostics.Debug.WriteLine(
        $"Items changed from {e.PreviousFilteredItemsCount} to {e.CurrentFilteredItemsCount}");
}
```