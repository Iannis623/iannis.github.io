---
title: Work in Progress - Custom Editor Mode in UE5
description: Learn how to make a custom editor mode with no source edits
author: iannis
date: 2024-09-15 22:45:00 +0100
categories: [Gamedev, Editor]
tags: [gamedev, editor, nosource]     # TAG names should always be lowercase
image:
  path: /assets/img/posts/2024-09-15-editor-mode/under-construction.jpg
---

# Introduction
In this post you'll learn to create a custom editor mode to interface with your systems step by step.\
Will include examples, custom styling and an example project with more in depth implementations for my Grid Gen, showcasing the full potential of a system like this.

>Code inside Classes is added in order of presentation unless otherwise specified.
{: .prompt-warning }

## Getting started
To get started, we will create a new Plugin that will hold all our logic.\
Note that you can do it directly in the Source of the project too.

You wanna go `Edit -> Plugins -> Add`, then select `Blank` and fill out the informations.\
I called my plugin `ExampleEditorMode`.

> There is an `Editor Mode template` available, but we are creating one from scratch to understand the inner workings of the entire system.
{: .prompt-info }

## ExampleEditorMode.uplugin
Make sure `"Type": "Runtime",` is instead of `"Type": "Editor",`

## ExampleEditorMode.Build.cs

You want these Private dependecies to start off:
```cpp
PrivateDependencyModuleNames.AddRange(
	new string[]
	{
		"CoreUObject",
		"Engine",
		"Slate",
		"SlateCore",
		"InputCore",
		"EditorFramework",
		"EditorStyle",
		"UnrealEd",
		"LevelEditor",
		"InteractiveToolsFramework",
		"EditorInteractiveToolsFramework",
		"Projects"
		// ... add private dependencies that you statically link with here ...	
	}
	);
```

## ExampleEditorModeCore.h

>This is our core file that everything will depend on.
{: .prompt-info }

### Includes
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Templates/SharedPointer.h"
#include "Tools/UEdMode.h"
#include "ExampleEditorModeCore.generated.h"
```

### Class
Now the main Class, inheriting from UEdMode which is the base class of Editor Modes.
```cpp
class UExampleEditorModeCore : public UEdMode
{
	GENERATED_BODY()
	
public:
	
};
```

### FEditorModeID
The unique identifier aka ID of the Editor Mode
```cpp
const static FEditorModeID EM_ExampleEditorModeId;
```

### SelectedToolName
A FString containing the name of the selected tool that we'll implement later.
```cpp
FString SelectedToolName;
```

### Constructor & Destructor
```cpp
UExampleEditorModeCore();
virtual ~UExampleEditorModeCore();
```

### Enter()
Function handling what happens when we enter the Editor Mode.
```cpp
virtual void Enter() override;
```

### CreateToolkit()
Function handling the creation of the Toolkit for initializing the UI panel of the tools, basically a helper.
```cpp
virtual void CreateToolkit() override;
```

### GetModeCommands()
Function handling the UI commands, this is a helper too for registering the buttons and dictate how they act.
```cpp
virtual TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> GetModeCommands() const override;
```

### OnToolStarted() & OnToolEnded()
Functions handling what happens when a tool is selected or deselected.
```cpp
virtual void OnToolStarted(UInteractiveToolManager* Manager, UInteractiveTool* Tool) override;
virtual void OnToolEnded(UInteractiveToolManager* Manager, UInteractiveTool* Tool) override;
```

### Full Code
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Templates/SharedPointer.h"
#include "Tools/UEdMode.h"
#include "ExampleEditorModeCore.generated.h"

/**
 * This class provides an example of how to extend a UEdMode to add some simple tools
 * using the InteractiveTools framework. The various UEdMode input event handlers (see UEdMode.h)
 * forward events to a UEdModeInteractiveToolsContext instance, which
 * has all the logic for interacting with the InputRouter, ToolManager, etc.
 * The functions provided here are the minimum to get started inserting some custom behavior.
 * Take a look at the UEdMode markup for more extensibility options.
 */
UCLASS()
class UExampleEditorModeCore : public UEdMode
{
	GENERATED_BODY()
	
public:
	const static FEditorModeID EM_ExampleEditorModeId;
	
	FString SelectedToolName;

	UExampleEditorModeCore();
	virtual ~UExampleEditorModeCore();

	/** UEdMode interface */
	virtual void Enter() override;
	virtual void CreateToolkit() override;
	virtual TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> GetModeCommands() const override;
	virtual void OnToolStarted(UInteractiveToolManager* Manager, UInteractiveTool* Tool) override;
	virtual void OnToolEnded(UInteractiveToolManager* Manager, UInteractiveTool* Tool) override;
};
```

## ExampleEditorModeCommands.h

> This class contains info about the full set of commands used in this editor mode.
{: .prompt-info }

### Includes
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Framework/Commands/Commands.h"
```

### Class
```cpp
class FExampleEditorModeCommands : public TCommands<FExampleEditorModeCommands>
{
public:
	
};
```

### Constructor
```cpp
FExampleEditorModeCommands();
```

### RegisterCommands()
In the implementation, this function will handle actually registering the tools and assigning the UI functionality.
```cpp
virtual void RegisterCommands() override;
```

### GetCommands()
Getter for Commands.\
Will be the return of `GetModeCommands()` in `ExampleEditorModeCore.h`
```cpp
static TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> GetCommands();
```

### Commands
The actual Array of available Commands.\
Gets filled in `RegisterCommands()`.
```cpp
TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> Commands;
```

### Full Code
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Framework/Commands/Commands.h"

/**
 * This class contains info about the full set of commands used in this editor mode.
 */
class FExampleEditorModeCommands : public TCommands<FExampleEditorModeCommands>
{
public:
	FExampleEditorModeCommands();

	virtual void RegisterCommands() override;
	static TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> GetCommands();

protected:
	TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> Commands;
};
```

## ExampleEditorModeToolkit.h

>This FModeToolkit just creates a basic UI panel that allows various InteractiveTools to\
> be initialized, and a DetailsView used to show properties of the active Tool.
{: .prompt-info }

### Includes
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Toolkits/BaseToolkit.h"
#include "ExampleEditorModeCore.h"
```

### Class
```cpp
class FExampleEditorModeToolkit : public FModeToolkit
{
public:

};
```

### Constructor
```cpp
FExampleEditorModeToolkit();
```

### Init()
Initializer, boilerplate with no custom changes.
```cpp
virtual void Init(const TSharedPtr<IToolkitHost>& InitToolkitHost, TWeakObjectPtr<UEdMode> InOwningMode) override;
```
### GetToolPaletteNames()
Boilerplate too, nothing that interests us.
```cpp
virtual void GetToolPaletteNames(TArray<FName>& PaletteNames) const override;
```
### GetToolkitFName()
Getter for name of the Toolkit, we'll use `ExampleEditorMode`
```cpp
virtual FName GetToolkitFName() const override;
```
### GetBaseToolkitName()
Getter for Localized Toolkit name, more boilerplate that must be implemented from Abstract class.\
We'll be using `ExampleEditorMode Toolkit`
```cpp
virtual FText GetBaseToolkitName() const override;
```

### Full Code
```cpp
#pragma once

#include "CoreMinimal.h"
#include "Toolkits/BaseToolkit.h"
#include "ExampleEditorModeCore.h"

/**
 * This FModeToolkit just creates a basic UI panel that allows various InteractiveTools to
 * be initialized, and a DetailsView used to show properties of the active Tool.
 */
class FExampleEditorModeToolkit : public FModeToolkit
{
public:
	FExampleEditorModeToolkit();

	/** FModeToolkit interface */
	virtual void Init(const TSharedPtr<IToolkitHost>& InitToolkitHost, TWeakObjectPtr<UEdMode> InOwningMode) override;
	virtual void GetToolPaletteNames(TArray<FName>& PaletteNames) const override;

	/** IToolkit interface */
	virtual FName GetToolkitFName() const override;
	virtual FText GetBaseToolkitName() const override;
};
```

## ExampleEditorModeStyle.h

> This FExampleEditorModeStyle will hold all the Style info for our Editor Mode.\
> Will allow us to have custom icons for example.
{: .prompt-info }

### Includes
```cpp
#pragma once

#include "Templates/SharedPointer.h"

struct FSlateBrush;
```

### Class
```cpp
class EXAMPLEEDITORMODE_API FExampleEditorModeStyle
{
public:

};
```

### Initialize()
Registersthe Style.
```cpp
static void Initialize();
```
### Shutdown()
Unregisters the Style.
```cpp
static void Shutdown();
```

### Get()
Getter for StyleSet underneath.
```cpp
static TSharedPtr< class ISlateStyle > Get();
```

### GetStyleSetName()
Name of the Style, it's basically like an ID since the StyleSet is created from it.
```cpp
static FName GetStyleSetName();
```

### GetBrush()
Use to access icons defined by the style set by name, eg GetBrush("BrushFalloffIcons.Smooth")
```cpp
static const FSlateBrush* GetBrush(FName PropertyName, const ANSICHAR* Specifier = NULL);
```

### InContent()
Sets the Content directory for the Resources used by the Style.
```cpp
private:
	static FString InContent(const FString& RelativePath, const ANSICHAR* Extension);
```

### StyleSet
A slate style chunk that contains a collection of named properties that guide the appearance of Slate.\
We'll be setting the icons throught this with `StyleSet->Set(...)`
```cpp
static TSharedPtr< class FSlateStyleSet > StyleSet;
```

### Full Code
```cpp
#pragma once

#include "Templates/SharedPointer.h"

struct FSlateBrush;

class EXAMPLEEDITORMODE_API FExampleEditorModeStyle
{
public:
	static void Initialize();

	static void Shutdown();

	static TSharedPtr< class ISlateStyle > Get();

	static FName GetStyleSetName();

	// use to access icons defined by the style set by name, eg GetBrush("BrushFalloffIcons.Smooth")
	static const FSlateBrush* GetBrush(FName PropertyName, const ANSICHAR* Specifier = NULL);

private:
	static FString InContent(const FString& RelativePath, const ANSICHAR* Extension);

	static TSharedPtr< class FSlateStyleSet > StyleSet;
};
```

## ExampleEditorModeStyle.cpp
We start with implementing the Style since it's self contained and others rely on it.

### Setup
Download this icon and place it into `YourProject -> Plugins -> ExampleEditorMode -> Content -> Icons`
![Desktop View](/assets/img/posts/2024-09-15-editor-mode/icon_Example_16px.png){: width="512" height="512" }

> If you want to use Editor included ones, have a look here\
> <a href="https://github.com/EpicKiwi/unreal-engine-editor-icons">https://github.com/EpicKiwi/unreal-engine-editor-icons</a>
{: .prompt-tip }

### Includes
```cpp
#include "ExampleEditorModeStyle.h"
#include "Interfaces/IPluginManager.h"
#include "Styling/SlateStyleRegistry.h"
```

### IMAGE_PLUGIN_BRUSH
This will point out to where the icons are held and the format expected.\
We're implementing `InContent()` in a moment.

> All the following code will between the IMAGE_PLUGIN_BRUSH unless otherwise specified.
{: .prompt-warning }
```cpp
#define IMAGE_PLUGIN_BRUSH( RelativePath, ... ) FSlateImageBrush( FExampleEditorModeStyle::InContent( RelativePath, ".png" ), __VA_ARGS__ )

// This is to fix the issue that SlateStyleMacros like IMAGE_BRUSH look for RootToContentDir but StyleSet->RootToContentDir is how this style is set up
#define RootToContentDir StyleSet->RootToContentDir

///
/* Rest of code goes here */
///

#undef IMAGE_PLUGIN_BRUSH
```

### InContent()
Setting `ContentDir` to the path leading to the Content of the Plugin to access the icons from.
```cpp
FString FExampleEditorModeStyle::InContent(const FString& RelativePath, const ANSICHAR* Extension)
{
	static FString ContentDir = IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir();
	return (ContentDir / RelativePath) + Extension;
}
```

### StyleSet
Initializing the StyleSet here and setting up a Getter.
```cpp
TSharedPtr< FSlateStyleSet > FExampleEditorModeStyle::StyleSet = nullptr;
TSharedPtr< class ISlateStyle > FExampleEditorModeStyle::Get() { return StyleSet; }
```

### GetStyleSetName()
Getter for the StyleName, will be used to initialize the StyleSet.
```cpp
FName FExampleEditorModeStyle::GetStyleSetName()
{
	static FName ExampleStyleName(TEXT("ExampleStyle"));
	return ExampleStyleName;
}
```

### GetBrush()
Use to access icons defined by the style set by name, eg GetBrush("BrushFalloffIcons.Smooth")
```cpp
const FSlateBrush* FExampleEditorModeStyle::GetBrush(FName PropertyName, const ANSICHAR* Specifier)
{
	return Get()->GetBrush(PropertyName, Specifier);
}
```

### Initialize()
```cpp
void FExampleEditorModeStyle::Initialize()
{
	// Texture size definitions
	const FVector2D Icon16x16(16.0f, 16.0f);
	const FVector2D Icon20x20(20.0f, 20.0f);
	const FVector2D Icon40x40(40.0f, 40.0f);

  // Only register once
	if (StyleSet.IsValid())
	{
		return;
	}

  StyleSet = MakeShareable(new FSlateStyleSet(GetStyleSetName()));

	// If we get asked for something that we don't set, we should default to editor style
	StyleSet->SetParentStyleName("EditorStyle");

	// Icons path
	StyleSet->SetContentRoot(IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir());
	StyleSet->SetCoreContentRoot(IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir());

  // Editor Mode icon, when needing to use it, call it with ExampleEditorMode.ExampleEdMode for example
	StyleSet->Set("ExampleEditorMode.ExampleEdMode", new IMAGE_PLUGIN_BRUSH("Icons/icon_Example_16px", Icon16x16));

  FSlateStyleRegistry::RegisterSlateStyle(*StyleSet.Get());
}

#undef IMAGE_PLUGIN_BRUSH
```

### Shutdown()
We unregister the style on shutdown.
> This has to be outside the IMAGE_PLUGIN_BRUSH define.
{: .prompt-warning }
```cpp
void FExampleEditorModeStyle::Shutdown()
{
	if (StyleSet.IsValid())
	{
		FSlateStyleRegistry::UnRegisterSlateStyle(*StyleSet.Get());
		ensure(StyleSet.IsUnique());
		StyleSet.Reset();
	}
}
```

### Full Code
```cpp
#include "ExampleEditorModeStyle.h"
#include "Interfaces/IPluginManager.h"
#include "Styling/SlateStyleRegistry.h"

#define IMAGE_PLUGIN_BRUSH( RelativePath, ... ) FSlateImageBrush( FExampleEditorModeStyle::InContent( RelativePath, ".png" ), __VA_ARGS__ )

// This is to fix the issue that SlateStyleMacros like IMAGE_BRUSH look for RootToContentDir but StyleSet->RootToContentDir is how this style is set up
#define RootToContentDir StyleSet->RootToContentDir

FString FExampleEditorModeStyle::InContent(const FString& RelativePath, const ANSICHAR* Extension)
{
	static FString ContentDir = IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir();
	return (ContentDir / RelativePath) + Extension;
}

TSharedPtr< FSlateStyleSet > FExampleEditorModeStyle::StyleSet = nullptr;
TSharedPtr< class ISlateStyle > FExampleEditorModeStyle::Get() { return StyleSet; }

FName FExampleEditorModeStyle::GetStyleSetName()
{
	static FName ExampleStyleName(TEXT("ExampleStyle"));
	return ExampleStyleName;
}

const FSlateBrush* FExampleEditorModeStyle::GetBrush(FName PropertyName, const ANSICHAR* Specifier)
{
	return Get()->GetBrush(PropertyName, Specifier);
}

void FExampleEditorModeStyle::Initialize()
{
	// Texture size definitions
	const FVector2D Icon16x16(16.0f, 16.0f);
	const FVector2D Icon20x20(20.0f, 20.0f);
	const FVector2D Icon40x40(40.0f, 40.0f);

	// Only register once
	if (StyleSet.IsValid())
	{
		return;
	}

	StyleSet = MakeShareable(new FSlateStyleSet(GetStyleSetName()));

	// If we get asked for something that we don't set, we should default to editor style
	StyleSet->SetParentStyleName("EditorStyle");

	// Icons path
	StyleSet->SetContentRoot(IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir());
	StyleSet->SetCoreContentRoot(IPluginManager::Get().FindPlugin(TEXT("ExampleEditorMode"))->GetContentDir());

	// Editor Mode icon, when needing to use it, call it with ExampleEditorMode.ExampleEdMode for example
	StyleSet->Set("ExampleEditorMode.ExampleEdMode", new IMAGE_PLUGIN_BRUSH("Icons/icon_Example_16px", Icon16x16));

	FSlateStyleRegistry::RegisterSlateStyle(*StyleSet.Get());
}

#undef IMAGE_PLUGIN_BRUSH

void FExampleEditorModeStyle::Shutdown()
{
	if (StyleSet.IsValid())
	{
		FSlateStyleRegistry::UnRegisterSlateStyle(*StyleSet.Get());
		ensure(StyleSet.IsUnique());
		StyleSet.Reset();
	}
}
```

## ExampleEditorModeCore.cpp

### Includes
```cpp
#include "ExampleEditorModeCore.h"
#include "ExampleEditorModeToolkit.h"
#include "EdModeInteractiveToolsContext.h"
#include "InteractiveToolManager.h"
#include "ExampleEditorModeCommands.h"
#include "ExampleEditorModeStyle.h"

#include "Interfaces/IPluginManager.h"
#include "Styling/SlateStyleMacros.h"
#include "Styling/SlateStyleRegistry.h"
```

### LOCTEXT_NAMESPACE
Localization scope.

> All the following code will between the LOCTEXT_NAMESPACE
{: .prompt-warning }

```cpp
#define LOCTEXT_NAMESPACE "ExampleEditorMode"
///
/* All the following code goes in between here */
///
#undef LOCTEXT_NAMESPACE
```

### EM_ExampleEditorModeId
We initialize the ID with a TEXT()\
Unique identifier of this Editor Mode.
```cpp
const FEditorModeID UExampleEditorModeCore::EM_ExampleEditorModeId = TEXT("EM_ExampleEditorMode");
```

### Constructor & Destructor
We are mostly interested in the Slate icon here and LOCTEXT.\
This will be the actual icon of the Editor Mode in the dropdown and the display name.\
Icon name coincides with the name implemented in the Style.
```cpp
UExampleEditorModeCore::UExampleEditorModeCore()
{
	//FModuleManager::Get().LoadModule("EditorStyle");
	
	// appearance and icon in the editing mode ribbon can be customized here
	Info = FEditorModeInfo(UExampleEditorModeCore::EM_ExampleEditorModeId,
		LOCTEXT("ModeName", "Example Editor Mode"),
		FSlateIcon(FExampleEditorModeStyle::GetStyleSetName(), "ExampleEditorMode.ExampleEdMode"),
		true);
}

UExampleEditorModeCore::~UExampleEditorModeCore()
{
}
```

### Enter()
We'll register our tool here when we create one.
```cpp
void UExampleEditorModeCore::Enter()
{
	UEdMode::Enter();

	const FExampleEditorModeCommands& SampleToolCommands = FExampleEditorModeCommands::Get();	
}
```

### CreateToolkit()
We create the actual Toolkit here.
```cpp
void UExampleEditorModeCore::CreateToolkit()
{
	Toolkit = MakeShareable(new FExampleEditorModeToolkit);
}
```

### GetModeCommands()
Utility class for getting the Commands from `FExampleEditorModeCommands()`
```cpp
TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> UExampleEditorModeCore::GetModeCommands() const
{
	return FExampleEditorModeCommands::Get().GetCommands();
}
```

### OnToolStarted() & OnToolEnded()
We're setting `SelectedToolName` when we select a new tool.
```cpp
void UExampleEditorModeCore::OnToolStarted(UInteractiveToolManager* Manager, UInteractiveTool* Tool)
{
	SelectedToolName = Manager->GetActiveToolName(EToolSide::Left);
}

void UExampleEditorModeCore::OnToolEnded(UInteractiveToolManager* Manager, UInteractiveTool* Tool)
{
}
```

### Full Code
```cpp
#include "ExampleEditorModeCore.h"
#include "ExampleEditorModeToolkit.h"
#include "EdModeInteractiveToolsContext.h"
#include "InteractiveToolManager.h"
#include "ExampleEditorModeCommands.h"
#include "ExampleEditorModeStyle.h"

#include "Interfaces/IPluginManager.h"
#include "Styling/SlateStyleMacros.h"
#include "Styling/SlateStyleRegistry.h"

#define LOCTEXT_NAMESPACE "ExampleEditorMode"

const FEditorModeID UExampleEditorModeCore::EM_ExampleEditorModeId = TEXT("EM_ExampleEditorMode");

UExampleEditorModeCore::UExampleEditorModeCore()
{
	//FModuleManager::Get().LoadModule("EditorStyle");
	
	// appearance and icon in the editing mode ribbon can be customized here
	Info = FEditorModeInfo(UExampleEditorModeCore::EM_ExampleEditorModeId,
		LOCTEXT("ModeName", "Example"),
		FSlateIcon(FExampleEditorModeStyle::GetStyleSetName(), "ExampleEditorMode.ExampleEdMode"),
		true);
}

UExampleEditorModeCore::~UExampleEditorModeCore()
{
}

void UExampleEditorModeCore::Enter()
{
	UEdMode::Enter();
	
	// Register the ToolBuilders for your Tools here.
	// The string name you pass to the ToolManager is used to select/activate your ToolBuilder later

	const FExampleEditorModeCommands& SampleToolCommands = FExampleEditorModeCommands::Get();	
}

void UExampleEditorModeCore::CreateToolkit()
{
	Toolkit = MakeShareable(new FExampleEditorModeToolkit);
}

TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> UExampleEditorModeCore::GetModeCommands() const
{
	return FExampleEditorModeCommands::Get().GetCommands();
}

void UExampleEditorModeCore::OnToolStarted(UInteractiveToolManager* Manager, UInteractiveTool* Tool)
{
	SelectedToolName = Manager->GetActiveToolName(EToolSide::Left);
}

void UExampleEditorModeCore::OnToolEnded(UInteractiveToolManager* Manager, UInteractiveTool* Tool)
{
}

#undef LOCTEXT_NAMESPACE
```

## ExampleEditorModeCommands.cpp

### Includes
```cpp
#include "ExampleEditorModeCommands.h"
#include "ExampleEditorModeCore.h"
#include "EditorStyleSet.h"
#include "ExampleEditorModeStyle.h"
#include "Styling/SlateStyleMacros.h"
```

### LOCTEXT_NAMESPACE
Localization scope.

> All the following code will between the LOCTEXT_NAMESPACE
{: .prompt-warning }

```cpp
#define LOCTEXT_NAMESPACE "ExampleEditorModeCommands"
///
/* All the following code goes in between here */
///
#undef LOCTEXT_NAMESPACE
```

### Constructor
Initializes and Localizes the Commands.\
For custom versions, just change Example to your custom name.
```cpp
FExampleEditorModeCommands::FExampleEditorModeCommands()
	: TCommands<FExampleEditorModeCommands>("ExampleEditorMode",
		NSLOCTEXT("ExampleEditorMode", "ExampleEditorModeCommands", "Example Editor Mode"),
		NAME_None,
		FExampleEditorModeStyle::GetStyleSetName())
{
}
```

### RegisterCommands()
We register the actual commands here.\
For now there's not gonna be any since we didn't create one yet.
```cpp
void FExampleEditorModeCommands::RegisterCommands()
{
	TArray <TSharedPtr<FUICommandInfo>>& ToolCommands = Commands.FindOrAdd(NAME_Default);

}
```

### GetCommands()
Getter for simplifying access.
```cpp
TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> FExampleEditorModeCommands::GetCommands()
{
	return FExampleEditorModeCommands::Get().Commands;
}
```

### Full Code
```cpp
#include "ExampleEditorModeCommands.h"
#include "ExampleEditorModeCore.h"
#include "EditorStyleSet.h"
#include "ExampleEditorModeStyle.h"
#include "Styling/SlateStyleMacros.h"

#define LOCTEXT_NAMESPACE "ExampleEditorModeCommands"

FExampleEditorModeCommands::FExampleEditorModeCommands()
	: TCommands<FExampleEditorModeCommands>("ExampleEditorMode",
		NSLOCTEXT("ExampleEditorMode", "ExampleEditorModeCommands", "Example Editor Mode"),
		NAME_None,
		FExampleEditorModeStyle::GetStyleSetName())
{
}

void FExampleEditorModeCommands::RegisterCommands()
{
	TArray <TSharedPtr<FUICommandInfo>>& ToolCommands = Commands.FindOrAdd(NAME_Default);
	
}

TMap<FName, TArray<TSharedPtr<FUICommandInfo>>> FExampleEditorModeCommands::GetCommands()
{
	return FExampleEditorModeCommands::Get().Commands;
}

#undef LOCTEXT_NAMESPACE
```

## ExampleEditorModeToolkit.cpp

### Includes
```cpp
#include "ExampleEditorModeToolkit.h"
#include "ExampleEditorModeCore.h"
#include "Engine/Selection.h"

#include "Modules/ModuleManager.h"
#include "PropertyEditorModule.h"
#include "IDetailsView.h"
#include "EditorModeManager.h"
```

### LOCTEXT_NAMESPACE
Localization scope.

> All the following code will between the LOCTEXT_NAMESPACE
{: .prompt-warning }

```cpp
#define LOCTEXT_NAMESPACE "ExampleEditorModeToolkit"
///
/* All the following code goes in between here */
///
#undef LOCTEXT_NAMESPACE
```

### Constructor
Nothing here.
```cpp
FExampleEditorModeToolkit::FExampleEditorModeToolkit()
{
}
```

### Init()
Calls the parent's implementation.
```cpp
void FExampleEditorModeToolkit::Init(const TSharedPtr<IToolkitHost>& InitToolkitHost, TWeakObjectPtr<UEdMode> InOwningMode)
{
	FModeToolkit::Init(InitToolkitHost, InOwningMode);
}
```

### GetToolPaletteNames()
Keep it as is.
```cpp
void FExampleEditorModeToolkit::GetToolPaletteNames(TArray<FName>& PaletteNames) const
{
	PaletteNames.Add(NAME_Default);
}
```

### GetToolkitFName()
Getter for name of the Toolkit, we'll use `ExampleEditorMode`
```cpp
FName FExampleEditorModeToolkit::GetToolkitFName() const
{
	return FName("ExampleEditorMode");
}
```


### GetBaseToolkitName()
Getter for Localized Toolkit name, more boilerplate that must be implemented from Abstract class.\
We'll be using `ExampleEditorMode Toolkit`
```cpp
FText FExampleEditorModeToolkit::GetBaseToolkitName() const
{
	return LOCTEXT("DisplayName", "ExampleEditorMode Toolkit");
}
```

### Full Code
```cpp
#include "ExampleEditorModeToolkit.h"
#include "ExampleEditorModeCore.h"
#include "Engine/Selection.h"

#include "Modules/ModuleManager.h"
#include "PropertyEditorModule.h"
#include "IDetailsView.h"
#include "EditorModeManager.h"

#define LOCTEXT_NAMESPACE "ExampleEditorModeToolkit"

FExampleEditorModeToolkit::FExampleEditorModeToolkit()
{
}

void FExampleEditorModeToolkit::Init(const TSharedPtr<IToolkitHost>& InitToolkitHost, TWeakObjectPtr<UEdMode> InOwningMode)
{
	FModeToolkit::Init(InitToolkitHost, InOwningMode);
}

void FExampleEditorModeToolkit::GetToolPaletteNames(TArray<FName>& PaletteNames) const
{
	PaletteNames.Add(NAME_Default);
}

FName FExampleEditorModeToolkit::GetToolkitFName() const
{
	return FName("ExampleEditorMode");
}

FText FExampleEditorModeToolkit::GetBaseToolkitName() const
{
	return LOCTEXT("DisplayName", "ExampleEditorMode Toolkit");
}

#undef LOCTEXT_NAMESPACE
```
## ExampleEditorMode.h
> This file will be auto generated during Plugin creation.
{: .prompt-warning }

Not much to do here, this is where you can implement custom logic as your module starts up or shuts down.

### OnPostEngineInit()
We are gonna add a delegate that we need for initializing the Style.\
This will get called after the Engine initializes.
```cpp
virtual void OnPostEngineInit();
```

## ExampleEditorMode.cpp
> This file will be auto generated during Plugin creation.
{: .prompt-warning }

## Includes
```cpp
#include "ExampleEditorMode.h"
#include "ExampleEditorModeStyle.h"
```
### OnPostEngineInit()
Here we have our implementations for the Startup and Shutdown of the module.\
We want to implement our `OnPostEngineInit()` and initialize the Style.
```cpp
void FExampleEditorMode::OnPostEngineInit()
{
	FExampleEditorModeStyle::Initialize();
}
```

### StartupModule()
Then we can bind the function to the Core Delegate OnPostEngineInit to call our custom logic.
```cpp
void FExampleEditorModeModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file per-module
	
	FCoreDelegates::OnPostEngineInit.AddRaw(this, &FExampleEditorModeModule::OnPostEngineInit);
}
```

## TO BE CONTINUED
> WORK IN PROGRESS, TOOL EXAMPLES COMING SOON
> <br>
> <br>
> FOR NOW IT'LL COMPILE AND YOU'LL HAVE ACCESS TO THE EDITOR MODE, BUT CLICKING IT WILL CRASH!
{: .prompt-danger }

> For an early preview and more in depth look into a proper implementation of those systems, please have a look at:
> <a href="https://github.com/Iannis623/UE-GridGen">https://github.com/Iannis623/UE-GridGen</a>
{: .prompt-tip }
