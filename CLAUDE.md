# CLAUDE.md - AI Assistant Guide for RevitLookup

This document provides comprehensive guidance for AI assistants working with the RevitLookup codebase. It covers architecture, conventions, workflows, and key patterns to follow when contributing to this project.

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Directory Structure](#directory-structure)
- [Key Design Patterns](#key-design-patterns)
- [Development Workflow](#development-workflow)
- [Coding Conventions](#coding-conventions)
- [Build System](#build-system)
- [Testing](#testing)
- [UI Development](#ui-development)
- [Common Tasks](#common-tasks)
- [Important Gotchas](#important-gotchas)

## Project Overview

**RevitLookup** is an interactive Revit BIM database exploration tool for viewing and navigating element parameters, properties, and relationships. It's an essential tool for Revit add-in developers to introspect the Revit API at runtime.

### Key Facts

- **Current Maintainer**: Nice3point (since 2022)
- **License**: MIT
- **Language**: C# (.NET Framework 4.8 and .NET 8.0)
- **Supported Revit Versions**: 2021-2026 (6 versions simultaneously)
- **Architecture**: MVVM with Dependency Injection
- **UI Framework**: WPF with Wpf.Ui library
- **Build System**: NUKE build automation
- **Primary Pattern**: Descriptor-based object introspection

### Project Statistics

- 362+ C# source files
- 93 Descriptor classes (7,386+ lines of descriptor code)
- 24 XAML views
- 8 project assemblies
- Multi-target support: net48, net8.0-windows

## Architecture

### High-Level Architecture

RevitLookup follows a clean, layered architecture:

```
┌─────────────────────────────────────────┐
│  RevitLookup (Main Plugin)              │
│  - Commands (External Commands)         │
│  - ViewModels (UI Logic)                │
│  - Services (Business Logic)            │
│  - Core (Descriptor System)             │
└─────────────────────────────────────────┘
              ↓ depends on
┌─────────────────────────────────────────┐
│  RevitLookup.UI.Framework               │
│  - Views (XAML)                         │
│  - Controls (Reusable WPF)              │
│  - Converters, Styles                   │
└─────────────────────────────────────────┘
              ↓ depends on
┌─────────────────────────────────────────┐
│  RevitLookup.Abstractions               │
│  - Interfaces (Contracts)               │
│  - Models (DTOs, Settings)              │
│  - ViewModels (Interfaces)              │
└─────────────────────────────────────────┘
              ↓ depends on
┌─────────────────────────────────────────┐
│  RevitLookup.Common                     │
│  - Utilities, Helpers                   │
└─────────────────────────────────────────┘
```

### Dependency Injection Container

The application uses **Microsoft.Extensions.Hosting** with a sophisticated DI setup:

**Host.cs** (`/source/RevitLookup/Host.cs`)
- Centralized service registration
- Scoped lifetimes for UI services
- Singleton for application-wide services
- Auto-registration via Scrutor for Views/ViewModels

**Service Categories:**
1. **Frontend Services**: Navigation, Dialogs, Snackbar, Notifications
2. **Application Services**: Settings, Software Updates, Theme, Ribbon
3. **Decomposition Services**: Object introspection, Visual decomposition, Search
4. **UI Orchestration**: Service coordination and lifecycle

**Service Retrieval:**
```csharp
var service = Host.GetService<IServiceType>();
```

### Entry Points

**Application.cs** (`/source/RevitLookup/Application.cs`)
- Implements `ExternalApplication`
- Lifecycle: `OnStartup()` → Initialize Host → Create Ribbon → `OnShutdown()` → Stop Host
- Initializes theming and hardware rendering

**Commands** (`/source/RevitLookup/Commands/`)
- All implement Nice3point.Revit.Toolkit.External base classes
- Examples: `DecomposeSelectionCommand`, `ShowDashboardCommand`, `SearchElementsCommand`
- Controller: `CommandAlwaysAvailableController` (controls availability)

## Directory Structure

### Source Projects (`/source/`)

```
source/
├── RevitLookup/                    # Main plugin assembly
│   ├── Commands/                   # Revit external commands
│   ├── Config/                     # Configuration (Http, Logging, Options)
│   ├── Core/                       # Core business logic
│   │   ├── Decomposition/         # Descriptor system (93 files)
│   │   │   ├── Descriptors/       # Type-specific descriptors
│   │   │   └── DescriptorsMap.cs  # Central descriptor registry
│   │   ├── RevitSettings/
│   │   ├── Search/
│   │   ├── Units/
│   │   └── Visualization/
│   ├── Mappers/                    # Riok.Mapperly object mappers
│   ├── Resources/                  # Images, icons
│   ├── Services/                   # Service implementations
│   ├── Styles/                     # WPF styles and converters
│   │   └── ComponentStyles/       # Data templates for types
│   ├── Utils/                      # Utility classes
│   ├── ViewModels/                 # View model implementations
│   ├── Application.cs              # Revit add-in entry point
│   └── Host.cs                     # DI container host
│
├── RevitLookup.Abstractions/       # Contracts and interfaces
│   ├── Configuration/
│   ├── Models/                     # Data models
│   ├── ObservableModels/          # MVVM observable models
│   ├── Options/
│   ├── Services/                   # Service interfaces
│   ├── States/
│   └── ViewModels/                 # ViewModel interfaces
│
├── RevitLookup.Common/             # Shared utilities
│   ├── Tools/
│   └── Utils/
│
├── RevitLookup.UI.Framework/       # WPF UI components
│   ├── Controls/                   # Custom controls
│   ├── Converters/                 # Value converters
│   ├── Extensions/                 # WPF extensions
│   ├── Markup/                     # XAML markup extensions
│   ├── Resources/                  # Themes, images
│   ├── Services/                   # Presentation services
│   ├── Utils/
│   └── Views/                      # XAML views (24 files)
│       ├── Dashboard/
│       ├── Decomposition/
│       ├── Settings/
│       ├── Tools/
│       ├── Visualization/
│       ├── Windows/
│       └── EditDialogs/
│
├── RevitLookup.UI.Playground/      # UI development/testing
│   └── (Standalone WPF app for rapid UI development)
│
├── LookupEngine/                   # External git submodule
└── LookupEngine.UI/                # External git submodule
```

### Other Important Directories

```
/build/                  # NUKE build scripts
/install/               # WixSharp MSI installer project
/tests/                 # Unit tests (Nice3point.TUnit.Revit)
/.github/workflows/     # CI/CD pipelines
/branding/              # Logos, icons, assets
/history/               # Historical documentation
```

## Key Design Patterns

### 1. Descriptor Pattern (Core Architecture)

The **Descriptor Pattern** is the heart of RevitLookup. It provides extensible object introspection.

**Location**: `/source/RevitLookup/Core/Decomposition/Descriptors/`

**Central Registry**: `DescriptorsMap.cs`
- Maps object types to their descriptors
- Uses pattern matching for type resolution
- Supports exact and approximate matches

**Base Descriptor Interfaces** (from LookupEngine.Abstractions):

#### IDescriptorResolver
Resolves methods/properties with parameters dynamically.

**Generic Version: `IDescriptorResolver<TContext>`** (context = Document)
```csharp
public class ElementDescriptor(Element element) : Descriptor, IDescriptorResolver<Document>
{
    public virtual Func<Document, IVariant>? Resolve(string target, ParameterInfo[] parameters)
    {
        return target switch
        {
            nameof(Element.IsHidden) => ResolveIsHidden,
            nameof(Element.GetBoundingBox) => ResolveBoundingBox,
            _ => null
        };

        // Single value resolution
        IVariant ResolveIsHidden(Document context)
        {
            return Variants.Value(element.IsHidden(context.ActiveView), "Active view");
        }

        // Multiple value resolution
        IVariant ResolveBoundingBox(Document context)
        {
            return Variants.Values<BoundingBoxXYZ>(2)
                .Add(element.get_BoundingBox(null), "Model")
                .Add(element.get_BoundingBox(context.ActiveView), "Active view")
                .Consume();
        }
    }
}
```

**Non-generic Version: `IDescriptorResolver`**
```csharp
public class DocumentDescriptor(Document document) : Descriptor, IDescriptorResolver
{
    public virtual Func<IVariant>? Resolve(string target, ParameterInfo[] parameters)
    {
        return target switch
        {
            nameof(Document.Close) => Variants.Disabled, // Disable dangerous methods
            _ => null
        };
    }
}
```

**Targeting Specific Overloads:**
```csharp
public sealed class EntityDescriptor(Entity entity) : Descriptor, IDescriptorResolver
{
    public Func<IVariant>? Resolve(string target, ParameterInfo[] parameters)
    {
        return target switch
        {
            nameof(Entity.Get) when parameters.Length == 1 &&
                                    parameters[0].ParameterType == typeof(string)
                => ResolveGetByField,
            _ => null
        };

        IVariant ResolveGetByField()
        {
            return Variants.Value(entity.Get("ParameterName"));
        }
    }
}
```

#### IDescriptorExtension
Adds custom properties/methods that don't exist in the original type.

**Without Context:**
```csharp
public sealed class ColorDescriptor(Color color) : Descriptor, IDescriptorExtension
{
    public void RegisterExtensions(IExtensionManager manager)
    {
        manager.Register("HEX", () => Variants.Value(ColorRepresentationUtils.ColorToHex(color)));
        manager.Register("RGB", () => Variants.Value(ColorRepresentationUtils.ColorToRgb(color)));
        manager.Register("HSL", () => Variants.Value(ColorRepresentationUtils.ColorToHsl(color)));
    }
}
```

**With Context:**
```csharp
public sealed class SchemaDescriptor(Schema schema) : Descriptor, IDescriptorExtension<Document>
{
    public void RegisterExtensions(IExtensionManager<Document> manager)
    {
        manager.Register("GetElements", context => Variants.Value(context
            .GetElements()
            .WherePasses(new ExtensibleStorageFilter(schema.GUID))
            .ToElements()));
    }
}
```

#### IDescriptorRedirector
Redirects inspection from one object to another (e.g., ElementId → Element).

```csharp
public sealed class ElementIdDescriptor(ElementId elementId) : Descriptor, IDescriptorRedirector<Document>
{
    public bool TryRedirect(string target, Document context, out object result)
    {
        if (elementId == ElementId.InvalidElementId)
        {
            result = null;
            return false;
        }

        result = elementId.ToElement(context);
        return result is not null;
    }
}
```

#### IDescriptorCollector
Marker interface indicating the descriptor can decompose object members.

```csharp
public sealed class ApplicationDescriptor : Descriptor, IDescriptorCollector
{
    public ApplicationDescriptor(Application application)
    {
        Name = application.VersionName;
    }
}
```

#### IDescriptorConnector
Integrates with UI to add context menu options.

```csharp
public sealed class ElementDescriptor : Descriptor, IDescriptorConnector
{
    public void RegisterMenu(ContextMenu contextMenu)
    {
        contextMenu.AddMenuItem()
            .SetHeader("Show element")
            .SetAvailability(_element is not ElementType)
            .SetCommand(_element, element =>
            {
                Context.UiDocument.ShowElements(element);
                Context.UiDocument.Selection.SetElementIds([element.Id]);
            })
            .AddShortcut(ModifierKeys.Alt, Key.F7);
    }
}
```

### 2. MVVM Pattern

**Framework**: CommunityToolkit.Mvvm (v8.4.0)

**Convention-Based Registration:**
- ViewModels ending in `ViewModel` are auto-registered
- Views are auto-registered by assembly scanning
- ViewModels are resolved via DI

**ViewModel Base Classes:**
- Use `ObservableObject` from CommunityToolkit.Mvvm
- Use `[ObservableProperty]` for properties
- Use `[RelayCommand]` for commands

**Example:**
```csharp
public partial class DecompositionViewModel : ObservableObject
{
    [ObservableProperty] private string _searchText;
    [ObservableProperty] private ObservableCollection<Item> _items;

    [RelayCommand]
    private void PerformSearch()
    {
        // Search logic
    }
}
```

### 3. Service-Oriented Architecture

**Service Lifetimes:**
- **Singleton**: Settings, Application-wide state, Ribbon
- **Scoped**: UI services (per window/dialog)
- **Transient**: Short-lived, per-operation services

**Service Interfaces** are in `RevitLookup.Abstractions/Services/`
**Implementations** are in `RevitLookup/Services/`

### 4. Object Mapping

**Framework**: Riok.Mapperly (v4.2.1)

Uses source generation for zero-overhead mapping:
```csharp
[Mapper]
public partial class MyObjectMapper
{
    public partial DestinationType Map(SourceType source);
}
```

## Development Workflow

### Prerequisites

- Windows 10 or newer
- .NET Framework 4.8
- .NET 9 SDK
- JetBrains Rider (recommended) or Visual Studio
- Git with submodules support
- Revit 2021-2026 (for testing)

### Initial Setup

1. **Clone repository:**
   ```bash
   git clone https://github.com/lookup-foundation/RevitLookup.git
   cd RevitLookup
   ```

2. **Initialize submodules:**
   ```bash
   git submodule update --init --force --recursive
   cd source/LookupEngine
   git sparse-checkout init --cone
   git sparse-checkout set source/
   cd ../LookupEngine.UI
   git sparse-checkout init --cone
   git sparse-checkout set source/
   cd ../..
   ```

3. **Open solution:**
   - Open `RevitLookup.sln` in JetBrains Rider or Visual Studio

4. **Select configuration:**
   - For Revit 2021: `Debug R21` or `Release R21`
   - For Revit 2025: `Debug R25` or `Release R25`
   - For UI development: `Debug Frontend`

### Building

**From IDE:**
1. Select desired configuration (e.g., `Debug R25`)
2. Build → Build Solution
3. Debug → Start Debugging (launches Revit)

**From Command Line (NUKE):**
```bash
# Install NUKE global tool (first time only)
dotnet tool install Nuke.GlobalTool --global

# Compile all versions
nuke

# Create installer
nuke createinstaller

# Create installer + bundle
nuke createinstaller createbundle
```

### Multi-Version Support

The project supports 6 Revit versions via build configurations:

| Config | Revit Version | Target Framework | RevitAPI Version |
|--------|---------------|------------------|------------------|
| R21    | 2021          | net48            | 2021.0.0         |
| R22    | 2022          | net48            | 2022.0.0         |
| R23    | 2023          | net48            | 2023.0.0         |
| R24    | 2024          | net48            | 2024.0.0         |
| R25    | 2025          | net8.0-windows   | 2025.0.0         |
| R26    | 2026          | net8.0-windows   | 2026.0.0         |

**Version Mapping** (in `Directory.Build.props`):
```xml
<PropertyGroup Condition="$(Configuration.Contains('R21'))">
    <TargetFramework>net48</TargetFramework>
    <RevitVersion>2021</RevitVersion>
</PropertyGroup>
```

**Important**: Don't use `#if REVIT2025` conditional compilation for most cases. Use configuration-specific builds instead.

## Coding Conventions

### File Organization

**Naming Patterns:**
- **Descriptors**: `{TypeName}Descriptor.cs` (e.g., `ElementDescriptor.cs`)
- **Services**: `{Purpose}Service.cs` + interface `I{Purpose}Service`
- **ViewModels**: `{Feature}ViewModel.cs` (auto-registered by suffix)
- **Commands**: `{Action}{Target}Command.cs`
- **Views**: `{Feature}Page.xaml`, `{Feature}Dialog.xaml`, `{Feature}Window.xaml`

### Code Style

**Follow existing patterns:**
- Use file-scoped namespaces: `namespace RevitLookup.Core;`
- Use primary constructors where appropriate: `public class Foo(Bar bar)`
- Enable nullable reference types (already enabled globally)
- Use implicit usings (configured in `Directory.Build.props`)
- Follow C# naming conventions (PascalCase for public, _camelCase for private fields)

### XML Documentation

**Required for:**
- Public classes and interfaces
- Public methods and properties
- Especially for service interfaces

**Example:**
```csharp
/// <summary>
///     Provides services for decomposing Revit objects into displayable descriptors
/// </summary>
public interface IDecompositionService
{
    /// <summary>
    ///     Decomposes a Revit object into a tree of descriptors
    /// </summary>
    /// <param name="obj">The object to decompose</param>
    /// <returns>The root descriptor node</returns>
    Descriptor Decompose(object obj);
}
```

### Copyright Header

All source files should include:
```csharp
// Copyright (c) Lookup Foundation and Contributors
//
// Permission to use, copy, modify, and distribute this software in
// object code form for any purpose and without fee is hereby granted,
// provided that the above copyright notice appears in all copies and
// that both that copyright notice and the limited warranty and
// restricted rights notice below appear in all supporting
// documentation.
//
// THIS PROGRAM IS PROVIDED "AS IS" AND WITH ALL FAULTS.
// NO IMPLIED WARRANTY OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR USE IS PROVIDED.
// THERE IS NO GUARANTEE THAT THE OPERATION OF THE PROGRAM WILL BE
// UNINTERRUPTED OR ERROR FREE.
```

### Logging

Use **Serilog** for logging:
```csharp
using Serilog;

public class MyService
{
    private static readonly ILogger Logger = Log.ForContext<MyService>();

    public void DoWork()
    {
        Logger.Information("Work started");
        Logger.Debug("Detail: {Detail}", detail);
        Logger.Error(ex, "Work failed");
    }
}
```

## Build System

### NUKE Build Automation

**Build Project**: `/build/Build.csproj`

**Partial Build Classes:**
- `Build.cs` - Main entry point, defines targets
- `Build.Configuration.cs` - Version mappings, configurations
- `Build.Compile.cs` - Compilation logic
- `Build.Clean.cs` - Cleanup operations
- `Build.CreateBundle.cs` - Autodesk .bundle creation
- `Build.CreateInstaller.cs` - MSI installer with WixSharp
- `Build.Publish.GitHub.cs` - GitHub release publishing
- `Build.Sign.cs` - Code signing with Azure Key Vault
- `Build.Changelog.cs` - Changelog extraction

**Key Targets:**
```bash
nuke                       # Default: Compile all configurations
nuke Clean                 # Clean build artifacts
nuke CreateInstaller       # Build MSI installer
nuke CreateBundle          # Create Autodesk bundle
nuke Sign                  # Code sign assemblies
```

### ILRepack (Assembly Merging)

Dependencies are merged into a single assembly using ILRepack:
- Configured per-project with `<IsRepackable>true</IsRepackable>`
- Excludes: `LookupEngine*.dll` (kept separate)
- Reduces deployment complexity

### Central Package Management (CPM)

**File**: `Directory.Packages.props`

All NuGet package versions are centralized:
```xml
<ItemGroup>
  <PackageVersion Include="CommunityToolkit.Mvvm" Version="8.4.0" />
  <PackageVersion Include="Nice3point.Revit.Api.Revit2025" Version="*" />
</ItemGroup>
```

Projects reference without version:
```xml
<PackageReference Include="CommunityToolkit.Mvvm" />
```

### GitHub Actions CI/CD

**Workflows** (`.github/workflows/`):

1. **Compile.yml** - Build verification on push/PR
2. **PublishRelease.yml** - Automated releases on tag push
   - Triggered by: `git tag 1.0.0 && git push origin 1.0.0`
   - Builds all configurations
   - Creates MSI installer and bundle
   - Publishes GitHub release with artifacts

## Testing

### Framework

**Nice3point.TUnit.Revit** - Custom test framework for Revit plugins

**Location**: `/tests/RevitLookup.Tests.Unit/`

**Characteristics:**
- Runs tests inside Revit process
- Multi-version support (R22-R26)
- Access to full Revit API during tests

**Test Files:**
- `LookupComposerTests.cs` - Descriptor composition tests
- `RevitApiTests.cs` - Revit API integration tests

**Running Tests:**
1. Select test configuration (e.g., `Debug R25`)
2. Set `RevitLookup.Tests.Unit` as startup project
3. Run tests (launches Revit with test runner)

### Testing Best Practices

- Test descriptor resolution logic
- Test service registrations
- Test ViewModel logic with mock services
- Use UI Playground for visual testing (see below)

## UI Development

### UI Playground

**Project**: `RevitLookup.UI.Playground`

**Purpose**: Develop and test UI without launching Revit

**Setup:**
1. Select `Debug Frontend` or `Release Frontend` configuration
2. Set `RevitLookup.UI.Playground` as startup project
3. Run (launches standalone WPF app)

**Features:**
- Mock services and data
- Fast iteration cycle (no Revit startup time)
- Isolated UI testing
- Simulates real Revit objects with Bogus library

**Workflow:**
1. Design/implement UI in Playground
2. Test with mock data
3. Refine and polish
4. Integrate into main RevitLookup project
5. Test in actual Revit

### UI Architecture

**XAML Views**: `RevitLookup.UI.Framework/Views/`
- Dashboard, Decomposition, Settings, Tools, Visualization, Windows, EditDialogs

**Styling**: `RevitLookup/Styles/ComponentStyles/`
- Data templates for specific types
- Template selectors for dynamic template selection

**Example - Custom Type Display:**

1. **Create DataTemplate** (`ComponentStyles/ObjectsTree/TreeGroupTemplates.xaml`):
```xml
<DataTemplate x:Key="SummaryMediaColorItemTemplate"
              DataType="{x:Type decomposition:ObservableDecomposedObject}">
    <Rectangle Width="16" Height="16"
               Fill="{Binding RawValue, Converter={StaticResource ColorToBrushConverter}}" />
</DataTemplate>
```

2. **Register in Selector** (`TreeViewItemTemplateSelector.cs`):
```csharp
public override DataTemplate? SelectTemplate(object? item, DependencyObject container)
{
    var decomposedObject = (ObservableDecomposedObject) item;
    var templateName = decomposedObject.RawValue switch
    {
        Color => "SummaryMediaColorItemTemplate",
        _ => "DefaultSummaryTreeItemTemplate"
    };

    return (DataTemplate) presenter.FindResource(templateName);
}
```

### WPF Controls

**Custom Controls** (`RevitLookup.UI.Framework/Controls/`):
- ColorPicker
- ContentPlaceholder
- Automation helpers

**Wpf.Ui Library:**
- Modern WPF controls
- Fluent design system
- Navigation service integration

## Common Tasks

### Adding a New Descriptor

**Scenario**: You need to introspect a new Revit type (e.g., `Connector`).

**Steps:**

1. **Create descriptor class** (`/source/RevitLookup/Core/Decomposition/Descriptors/ConnectorDescriptor.cs`):
```csharp
using Autodesk.Revit.DB;
using LookupEngine.Abstractions.Decomposition;
using LookupEngine.Descriptors;

namespace RevitLookup.Core.Decomposition.Descriptors;

public sealed class ConnectorDescriptor(Connector connector) : Descriptor, IDescriptorCollector
{
    // Add constructor if you need custom Name
    // public ConnectorDescriptor(Connector connector) : base(connector)
    // {
    //     Name = $"Connector {connector.Id}";
    // }
}
```

2. **Register in DescriptorsMap** (`/source/RevitLookup/Core/Decomposition/DescriptorsMap.cs`):
```csharp
public static Descriptor FindDescriptor(object? obj, Type? type)
{
    return obj switch
    {
        // ... existing mappings ...
        Connector value when type is null || type == typeof(Connector)
            => new ConnectorDescriptor(value),
        // ... rest ...
    };
}
```

3. **Add resolution logic** (if needed):
```csharp
public sealed class ConnectorDescriptor(Connector connector) : Descriptor,
    IDescriptorCollector,
    IDescriptorResolver<Document>
{
    public Func<Document, IVariant>? Resolve(string target, ParameterInfo[] parameters)
    {
        return target switch
        {
            nameof(Connector.GetMEPConnectorInfo) => ResolveMEPInfo,
            _ => null
        };

        IVariant ResolveMEPInfo(Document context)
        {
            var info = connector.GetMEPConnectorInfo();
            return Variants.Value(info);
        }
    }
}
```

4. **Add extensions** (if needed):
```csharp
public sealed class ConnectorDescriptor(Connector connector) : Descriptor,
    IDescriptorCollector,
    IDescriptorExtension
{
    public void RegisterExtensions(IExtensionManager manager)
    {
        manager.Register("IsConnected", () =>
            Variants.Value(connector.IsConnected));
        manager.Register("AllRefs", () =>
            Variants.Value(connector.AllRefs.Cast<Connector>().ToList()));
    }
}
```

5. **Test**:
   - Build and run in Revit
   - Select object with Connector
   - Verify descriptor displays correctly

### Adding a New Command

**Scenario**: Add a new Revit command (e.g., "Export Selection Data").

**Steps:**

1. **Create command class** (`/source/RevitLookup/Commands/ExportSelectionDataCommand.cs`):
```csharp
using Autodesk.Revit.Attributes;
using Nice3point.Revit.Toolkit.External;
using RevitLookup.Services.MyServices;

namespace RevitLookup.Commands;

[UsedImplicitly]
[Transaction(TransactionMode.Manual)]
public class ExportSelectionDataCommand : ExternalCommand
{
    public override void Execute()
    {
        var exportService = Host.GetService<IExportService>();
        var selection = Context.UiDocument.Selection.GetElementIds();

        exportService.ExportElements(selection);
    }
}
```

2. **Register in Ribbon** (`/source/RevitLookup/Services/Application/RevitRibbonService.cs`):
```csharp
public void CreateRibbon()
{
    var panel = Application.CreatePanel("RevitLookup", "Lookup");

    panel.AddPushButton<ExportSelectionDataCommand>("Export\nSelection")
        .SetImage("/RevitLookup;component/Resources/Icons/Export.png")
        .SetToolTip("Export selection data to JSON");
}
```

3. **Create service** (if needed):
```csharp
// Interface in RevitLookup.Abstractions/Services/
public interface IExportService
{
    void ExportElements(ICollection<ElementId> elementIds);
}

// Implementation in RevitLookup/Services/
public class ExportService : IExportService
{
    public void ExportElements(ICollection<ElementId> elementIds)
    {
        // Export logic
    }
}

// Register in Host.cs
builder.Services.AddTransient<IExportService, ExportService>();
```

### Adding a New View/ViewModel

**Scenario**: Add a new settings page.

**Steps:**

1. **Create ViewModel interface** (`/source/RevitLookup.Abstractions/ViewModels/Settings/IMySettingsViewModel.cs`):
```csharp
namespace RevitLookup.Abstractions.ViewModels.Settings;

public interface IMySettingsViewModel
{
    bool SomeSetting { get; set; }
}
```

2. **Create ViewModel** (`/source/RevitLookup/ViewModels/Settings/MySettingsViewModel.cs`):
```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace RevitLookup.ViewModels.Settings;

public partial class MySettingsViewModel : ObservableObject, IMySettingsViewModel
{
    [ObservableProperty] private bool _someSetting;

    public MySettingsViewModel()
    {
        // Initialize from settings
    }
}
```

3. **Create View** (`/source/RevitLookup.UI.Framework/Views/Settings/MySettingsPage.xaml`):
```xml
<Page x:Class="RevitLookup.UI.Framework.Views.Settings.MySettingsPage"
      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
      xmlns:viewModels="clr-namespace:RevitLookup.Abstractions.ViewModels.Settings"
      d:DataContext="{d:DesignInstance viewModels:IMySettingsViewModel}">
    <StackPanel>
        <CheckBox Content="Some Setting" IsChecked="{Binding SomeSetting}" />
    </StackPanel>
</Page>
```

4. **Code-behind** (minimal):
```csharp
namespace RevitLookup.UI.Framework.Views.Settings;

public partial class MySettingsPage
{
    public MySettingsPage(IMySettingsViewModel viewModel)
    {
        DataContext = viewModel;
        InitializeComponent();
    }
}
```

**Auto-Registration**: Views and ViewModels are auto-registered via Scrutor in `Host.cs`. No manual registration needed!

### Adding a New Service

1. **Define interface** in `RevitLookup.Abstractions/Services/`
2. **Implement service** in `RevitLookup/Services/`
3. **Register in Host.cs**:
```csharp
builder.Services.AddSingleton<IMyService, MyService>();
// or AddScoped, AddTransient
```
4. **Inject via constructor**:
```csharp
public class MyViewModel(IMyService myService)
{
    public void DoWork() => myService.PerformOperation();
}
```

## Important Gotchas

### 1. Submodule Projects (LookupEngine)

**DO NOT** modify code in:
- `/source/LookupEngine/`
- `/source/LookupEngine.UI/`

These are **git submodules** from external repositories. Changes must be made in their source repos.

### 2. Multi-Version Builds

**Problem**: Changes work in R25 but break R21.

**Solution**:
- Test multiple configurations (especially net48 vs net8.0)
- Avoid using APIs not available in older Revit versions
- Use PolySharp polyfills for C# features
- Check API compatibility: https://www.revitapidocs.com/

### 3. ILRepack Assembly Merging

**Problem**: Dependencies not found at runtime.

**Solution**:
- Ensure `<IsRepackable>true</IsRepackable>` in project file
- Check build logs for ILRepack warnings
- Don't merge LookupEngine assemblies (excluded by design)

### 4. Descriptor Registration

**Problem**: Custom descriptor not being used.

**Solution**:
- Verify registration in `DescriptorsMap.cs`
- Check type matching pattern: `type is null || type == typeof(MyType)`
- Ensure exact type or base type matches
- Pattern matching is order-dependent (more specific first)

### 5. ViewModel Auto-Registration

**Problem**: ViewModel not being resolved by DI.

**Solution**:
- Ensure class name ends with `ViewModel`
- Verify it's in the `RevitLookup` or `RevitLookup.UI.Framework` assembly
- Check it has a public constructor
- Scrutor assembly scanning looks for `*ViewModel` suffix

### 6. Hardware Rendering

**Problem**: WPF performance issues in Revit.

**Context**: Revit forces software rendering. RevitLookup optionally re-enables hardware rendering.

**Setting**: User can toggle in Settings → `UseHardwareRendering`

**Implementation**: `Application.EnableHardwareRendering()` / `DisableHardwareRendering()`

### 7. Revit Context

**Problem**: Accessing Revit API outside valid context.

**Solution**:
- Use `RevitShell.ActionEventHandler` for deferred execution
- Commands automatically run in valid Revit context
- Services must not call Revit API in constructors

### 8. Git Submodules

**Problem**: Submodule directories are empty after clone.

**Solution**:
```bash
git submodule update --init --force --recursive
cd source/LookupEngine
git sparse-checkout init --cone
git sparse-checkout set source/
cd ../LookupEngine.UI
git sparse-checkout init --cone
git sparse-checkout set source/
```

### 9. Build Configuration Selection

**Problem**: Wrong Revit version DLL is loaded.

**Solution**:
- Verify correct configuration (R21-R26) is selected
- Clean solution before switching configurations
- Check project output path matches selected configuration

### 10. Nullable Reference Types

**All projects have nullable enabled.**

**Rules:**
- Use `?` for nullable reference types
- Handle null cases explicitly
- Avoid `!` (null-forgiving) unless absolutely necessary
- Enable warnings as errors for nullable violations

## Version Release Process

### Creating a Release

Releases are triggered by **git tags** following semantic versioning.

**Tag Format:**
- Production: `1.0.0`, `2.5.3`
- Pre-release: `1.0.0-alpha`, `1.0.0-beta.2.20250101`

**Steps:**

1. **Update Changelog** (`/Changelog.md`):
```markdown
# 1.2.0

- Added new descriptor for Connector objects
- Fixed issue with hardware rendering toggle
- Improved performance of decomposition search
```

2. **Create and push tag:**
```bash
git tag 1.2.0
git push origin 1.2.0
```

3. **Automated Process** (via GitHub Actions):
   - Builds all configurations (R21-R26)
   - Runs tests
   - Creates MSI installer
   - Creates Autodesk bundle
   - Signs assemblies (if configured)
   - Publishes GitHub release
   - Uploads artifacts

**Pre-release tags** (e.g., `1.0.0-alpha.1.20250101`):
- Marked as pre-release on GitHub
- For testing and early feedback
- Not recommended for production use

## Configuration Files Reference

### Directory.Build.props
Global MSBuild properties for all projects:
- Nullable enabled
- LangVersion: latest
- Platform: x64
- ImplicitUsings: true
- Revit version → framework mapping

### Directory.Packages.props
Central Package Management:
- All NuGet versions centralized
- Floating versions for Revit packages (`*`)
- Transitive pinning enabled

### global.json
SDK version control:
- .NET SDK 9.0.0
- rollForward: latestMinor

### RevitLookup.addin
Revit add-in manifest:
- AddInId: `356CDA5A-E6C5-4c2f-A9EF-B3222116B8C8`
- VendorId: `LookupFoundation`
- UseRevitContext: True (Revit 2025+ isolated loading)

## Dependencies Reference

### Core Revit
- **Nice3point.Revit.*** - Revit API wrappers, build tasks, extensions
- **Autodesk.Revit.*** - Official Revit API assemblies

### UI & MVVM
- **CommunityToolkit.Mvvm** (8.4.0) - MVVM framework
- **Wpf.Ui** - Modern WPF controls
- **Riok.Mapperly** (4.2.1) - Object mapping via source generation

### Dependency Injection
- **Microsoft.Extensions.Hosting** (9.0.7) - Generic host
- **Microsoft.Extensions.Http** (9.0.7) - HttpClient factory
- **Scrutor** (6.1.0) - Assembly scanning for DI

### Logging
- **Serilog** (4.3.0) - Structured logging
- **Serilog.Sinks.Autodesk.Revit** (2.0.1) - Revit-specific log output

### Build & Deployment
- **Nuke.Common** (9.0.4) - Build automation
- **WixSharp** (1.26.0) - MSI installer generation
- **ILRepack** (2.0.44) - Assembly merging
- **Autodesk.PackageBuilder** (2.0.1) - Bundle creation

### Testing
- **Nice3point.TUnit.Revit** - Revit-aware test framework

### Utilities
- **JetBrains.Annotations** (2025.2.0) - Code annotations
- **PolySharp** (1.15.0) - C# polyfills for older frameworks
- **Bogus** (35.6.3) - Fake data generation (UI Playground)

## Resources

### Documentation
- **Readme.md** - Project overview
- **Contributing.md** - Detailed contribution guide with architecture details
- **Changelog.md** - Version history
- **License.md** - MIT license

### External Links
- [Revit API Documentation](https://www.revitapidocs.com/)
- [LookupEngine Framework](https://github.com/lookup-foundation/LookupEngine)
- [Nice3point.Revit.Toolkit](https://github.com/Nice3point/RevitToolkit)
- [The Building Coder Blog](https://thebuildingcoder.typepad.com/)

### Community
- [GitHub Issues](https://github.com/lookup-foundation/RevitLookup/issues)
- [GitHub Discussions](https://github.com/lookup-foundation/RevitLookup/discussions)
- [Contributors](https://github.com/lookup-foundation/RevitLookup/graphs/contributors)

## Quick Reference Commands

```bash
# Build all configurations
nuke

# Build specific version
dotnet build -c "Debug R25"

# Create installer
nuke createinstaller

# Create installer and bundle
nuke createinstaller createbundle

# Run tests (select test project configuration first)
dotnet test -c "Debug R25"

# Update submodules
git submodule update --remote

# Create release tag
git tag 1.2.0 && git push origin 1.2.0
```

## AI Assistant Guidelines

When working on this codebase as an AI assistant:

1. **Always check Contributing.md** for detailed architecture documentation
2. **Follow existing patterns** - consistency is critical
3. **Test across multiple Revit versions** when making API calls
4. **Use UI Playground** for rapid UI development iteration
5. **Add XML documentation** to all public APIs
6. **Register new descriptors** in DescriptorsMap.cs
7. **Follow naming conventions** for auto-registration to work
8. **Consider descriptor interfaces** - use the right combination for your needs
9. **Don't modify submodule code** - it's externally managed
10. **Check nullability** - all projects have nullable enabled
11. **Use DI** - don't create services with `new`, inject them
12. **Leverage source generators** - Mapperly for mapping, CommunityToolkit.Mvvm for MVVM

## Conclusion

RevitLookup is a mature, well-architected Revit plugin with:
- Clean separation of concerns
- Extensive use of modern .NET patterns
- Robust multi-version support
- Comprehensive descriptor system for object introspection

The descriptor pattern is the core innovation - understanding it is key to extending RevitLookup effectively.

For detailed architectural information, always refer to **Contributing.md**. This CLAUDE.md provides a practical guide, while Contributing.md offers deeper architectural insights.

**Happy coding!** 🚀
