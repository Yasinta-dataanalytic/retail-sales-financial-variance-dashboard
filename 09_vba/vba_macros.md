# VBA Automation

Three macros power the buttons in the dashboard header. They live in a standard module of `retail_sales_dashboard.xlsm`.

| Button | Macro | What it does |
|---|---|---|
| Reset | `ResetDashboard` | Sets Year = 2025 and clears the Month, Region and Category slicers |
| Export PDF | `ExportDashboardPDF` | Saves the dashboard as a one-page landscape PDF in Documents, with the date in the file name |
| Refresh | `RefreshAndStamp` | Refreshes all Power Query connections and PivotTables, then stamps the time in the "Last refreshed" box |

> **Note:** Macros only run in `.xlsm` files. Click **Enable Content** when Excel asks.

## Code

```vba
Option Explicit

' 1. Reset all slicers: Year = 2025, everything else cleared.
'    Month is cleared first so Year never sits next to a partial Month selection.
Sub ResetDashboard()
    On Error GoTo Fail
    Application.ScreenUpdating = False
    With ThisWorkbook
        .SlicerCaches("Slicer_Month_Short").ClearManualFilter
        .SlicerCaches("Slicer_Region").ClearManualFilter
        .SlicerCaches("Slicer_Category").ClearManualFilter
        .SlicerCaches("Slicer_Year1").VisibleSlicerItemsList = Array("[Dim_Date].[Year].&[2025]")
    End With
    Application.ScreenUpdating = True
    Exit Sub
Fail:
    Application.ScreenUpdating = True
    MsgBox "Reset failed: " & Err.Description, vbExclamation
End Sub

' 2. Save the dashboard sheet as a dated PDF in Documents.
Sub ExportDashboardPDF()
    On Error GoTo Fail
    Dim ws As Worksheet, p As String
    Set ws = ThisWorkbook.Worksheets("dashboard")
    With ws.PageSetup
        .Orientation = xlLandscape
        .Zoom = False
        .FitToPagesWide = 1
        .FitToPagesTall = 1
    End With
    p = Environ("USERPROFILE") & "\Documents\Retail_Dashboard_" & Format(Date, "yyyy-mm-dd") & ".pdf"
    ws.ExportAsFixedFormat Type:=xlTypePDF, Filename:=p, Quality:=xlQualityStandard, OpenAfterPublish:=True
    Exit Sub
Fail:
    MsgBox "PDF export failed: " & Err.Description, vbExclamation
End Sub

' 3. Refresh all data and stamp the time (needs a cell named LastRefreshed).
Sub RefreshAndStamp()
    Dim nm As Name
    On Error Resume Next
    Set nm = ThisWorkbook.Names("LastRefreshed")
    On Error GoTo Fail

    If nm Is Nothing Then
        MsgBox "Cell named 'LastRefreshed' not found. Name a cell via the Name Box.", vbExclamation
        Exit Sub
    End If

    ThisWorkbook.RefreshAll
    Application.CalculateUntilAsyncQueriesDone
    nm.RefersToRange.Value = "Last refreshed: " & Format(Now, "dd mmm yyyy hh:mm")
    Exit Sub
Fail:
    MsgBox "Refresh failed: " & Err.Description, vbExclamation
End Sub
```

## Why automate

- **Reset** returns the dashboard to a clean full-year view in one click, so every export and screenshot starts from the same state.
- **Export PDF** produces a consistent, shareable report without manual page setup.
- **Refresh** updates the whole model and shows when the data was last refreshed, so readers know how current the numbers are.

## Limitations

- The slicer names (`Slicer_Year1`, `Slicer_Month_Short`, `Slicer_Region`, `Slicer_Category`) are specific to this workbook.
- `RefreshAndStamp` requires the Power Query source paths to point to the CSV files on your machine (see the main README).
- The PDF is saved to the Windows `Documents` folder.
