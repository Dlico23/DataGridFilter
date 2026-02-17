# FilterDataGrid - Enhanced Features

## Overview

The `FilterDataGrid` class now includes built-in support for programmatic control over filters, state preservation, and additional events. All features are integrated directly into the base class - no extended class needed!

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

## Complete Implementation Example

### XAML
```xml
<Window x:Class="YourApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:fg="clr-namespace:FilterDataGrid;assembly=FilterDataGrid.Net">
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Toolbar -->
        <ToolBar Grid.Row="0">
            <Button Content="Refresh Data" Click="RefreshButton_Click"/>
            <Button Content="Apply Custom Filter" Click="ApplyFilterButton_Click"/>
            <Button Content="Clear Filters" Click="ClearFiltersButton_Click"/>
            <Button Content="Export Filtered" Click="ExportButton_Click"/>
            <Button Content="Save Filter" Click="SaveFilterButton_Click"/>
            <Button Content="Load Filter" Click="LoadFilterButton_Click"/>
        </ToolBar>
        
        <!-- DataGrid -->
        <fg:FilterDataGrid 
            x:Name="dataGrid"
            Grid.Row="1"
            PreserveStateOnItemsSourceChange="True"
            FilterCompleted="DataGrid_FilterCompleted"
            ItemsSourceChangedComplete="DataGrid_ItemsSourceChangedComplete"
            ShowStatusBar="True"
            ShowElapsedTime="True"
            ShowRowsCount="True"
            PersistentFilter="False"
            AutoGenerateColumns="False">
            
            <fg:FilterDataGrid.Columns>
                <fg:DataGridTextColumn Header="Name" FieldName="Name" IsColumnFiltered="True"/>
                <fg:DataGridTextColumn Header="Status" FieldName="Status" IsColumnFiltered="True"/>
                <fg:DataGridNumericColumn Header="Priority" FieldName="Priority" IsColumnFiltered="True"/>
                <fg:DataGridTextColumn Header="Category" FieldName="Category" IsColumnFiltered="True"/>
                <fg:DataGridTextColumn Header="Date" FieldName="Date" IsColumnFiltered="True" 
                                       Binding="{Binding Date, StringFormat={}{0:yyyy-MM-dd}}"/>
            </fg:FilterDataGrid.Columns>
            
        </fg:FilterDataGrid>
        
        <!-- Status Bar -->
        <StatusBar Grid.Row="2">
            <StatusBarItem>
                <TextBlock x:Name="statusText" Text="Ready"/>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

### Code-Behind
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Windows;
using FilterDataGrid;

namespace YourApp
{
    public partial class MainWindow : Window
    {
        private const string FilterStateFile = "gridFilter.json";
        
        public MainWindow()
        {
            InitializeComponent();
            
            // Setup event handlers
            dataGrid.FilterCompleted += DataGrid_FilterCompleted;
            dataGrid.ItemsSourceChanging += DataGrid_ItemsSourceChanging;
            dataGrid.ItemsSourceChangedComplete += DataGrid_ItemsSourceChangedComplete;
            
            // Load initial data
            LoadData();
            
            // Restore previous filter if exists
            if (System.IO.File.Exists(FilterStateFile))
            {
                RestoreFilterState();
            }
        }
        
        private void LoadData()
        {
            // Load your data
            List<MyDataClass> data = GetDataFromDatabase();
            dataGrid.ItemsSource = data;
        }
        
        private void RefreshButton_Click(object sender, RoutedEventArgs e)
        {
            // Refresh data while maintaining filter and view state
            LoadData();
            statusText.Text = "Data refreshed - filter and position maintained";
        }
        
        private void ApplyFilterButton_Click(object sender, RoutedEventArgs e)
        {
            // Apply a custom filter programmatically
            dataGrid.CreateFilterBuilder()
                .AddFilter("Status", "Archived", "Deleted")
                .AddIncludeFilter("Priority", 1, 2, 3)
                .Apply();
            
            statusText.Text = "Custom filter applied";
        }
        
        private void ClearFiltersButton_Click(object sender, RoutedEventArgs e)
        {
            dataGrid.RemoveFilters();
            statusText.Text = "All filters cleared";
        }
        
        private void ExportButton_Click(object sender, RoutedEventArgs e)
        {
            // Export only filtered data
            List<MyDataClass> filteredData = dataGrid.FilteredItems
                .Cast<MyDataClass>()
                .ToList();
            
            // Export to CSV, Excel, etc.
            ExportToFile(filteredData);
            statusText.Text = $"Exported {filteredData.Count} filtered items";
        }
        
        private void SaveFilterButton_Click(object sender, RoutedEventArgs e)
        {
            SaveFilterState();
            statusText.Text = "Filter state saved";
        }
        
        private void LoadFilterButton_Click(object sender, RoutedEventArgs e)
        {
            LoadFilterState();
            statusText.Text = "Filter state loaded";
        }
        
        private void SaveFilterState()
        {
            FilterStateData state = dataGrid.GetCurrentFilterState();
            string json = System.Text.Json.JsonSerializer.Serialize(state);
            System.IO.File.WriteAllText(FilterStateFile, json);
        }
        
        private void LoadFilterState()
        {
            if (!System.IO.File.Exists(FilterStateFile)) return;
            
            string json = System.IO.File.ReadAllText(FilterStateFile);
            FilterStateData state = System.Text.Json.JsonSerializer.Deserialize<FilterStateData>(json);
            dataGrid.ApplyFilterState(state);
        }
        
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
        
        private void DataGrid_ItemsSourceChanging(object sender, ItemsSourceChangingEventArgs e)
        {
            // Optionally cancel state preservation
            System.Diagnostics.Debug.WriteLine("ItemsSource is about to change...");
        }
        
        private void DataGrid_ItemsSourceChangedComplete(object sender, ItemsSourceChangedEventArgs e)
        {
            if (e.StateRestored)
            {
                System.Diagnostics.Debug.WriteLine("Filter, selection, and scroll position restored");
            }
        }
        
        private void ExportToFile(List<MyDataClass> data)
        {
            // Implementation for exporting data
        }
        
        private List<MyDataClass> GetDataFromDatabase()
        {
            // Implementation for loading data
            return new List<MyDataClass>();
        }
    }
    
    // Data class
    public class MyDataClass
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Status { get; set; }
        public int Priority { get; set; }
        public string Category { get; set; }
        public DateTime Date { get; set; }
    }
}
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

## Performance Considerations

- State preservation is implemented efficiently using direct member access (no reflection overhead)
- Filter state serialization is fast and lightweight
- FilteredItems property iterates the collection - cache if needed for repeated access
- State preservation on ItemsSource change is async and non-blocking

## Migration Note

**All features are now built into the base `FilterDataGrid` class!** No extended class is needed. Simply use:

```xml
<fg:FilterDataGrid .../>
```

All enhanced features are available by default.
