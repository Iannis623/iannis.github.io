---
title: Custom Material Output Node in UE5
description: Learn how to make a fully custom material output node with source edits
author: iannis
date: 2024-09-13 14:20:00 +0100
categories: [Gamedev, Materials]
tags: [gamedev, materials, source]     # TAG names should always be lowercase
image:
  path: /assets/img/posts/2024-09-13-material-output-node/exampleoutputtitle.png
---

# Introduction
In this post you'll learn to modify the unreal engine source to add a custom material output, technically called a `MaterialExpressionCustomOutput`, to your material graphs.\
Those can se extremely useful if you want to directly interface with the shader files while passing your own custom values inside of it.\
In this example we'll use it to directly replace the BaseColor and Normal of a Material.

> `SourceMod` are comments that indicate it's a Source Engine edits.\
> Sometimes may be included above snippets of code to showcase where a certain code snippet should be placed.\
> You only need to include code snippets with `SourceMod` comment\
> For example: 
> ```cpp
> // This is not a Source Edit
> void BarrelRoll();
>
> // SourceMod: This is a Source Edit
> void HelloWorld();
>```
{: .prompt-info }

## MaterialExpressionExampleOutput.h
A `MaterialExpressionCustomOutput` is file that defines the Input parameters of the Custom Output node.

You can find examples of this in `Engine/Source/Runtime/Engine/Classes/Materials/`\
We'll be creating our Custom Output node here.

For our example we'll create a `.h` file called `MaterialExpressionExampleOutput.h`

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Materials/MaterialExpressionCustomOutput.h"
#include "UObject/ObjectMacros.h"
#include "MaterialExpressionExampleOutput.generated.h"

/** Material output expression for writing BaseColor and Normal material properties overrides. */
UCLASS(MinimalAPI, collapsecategories, hidecategories = Object)
class UMaterialExpressionExampleOutput : public UMaterialExpressionCustomOutput
{
	GENERATED_UCLASS_BODY()

	/** The override BaseColor */
	UPROPERTY()
	FExpressionInput OverrideBaseColor;

	/** The override Normal */
	UPROPERTY()
	FExpressionInput OverrideNormal;

public:
#if WITH_EDITOR
	//~ Begin UMaterialExpression Interface
	virtual int32 Compile(class FMaterialCompiler* Compiler, int32 OutputIndex) override;
	virtual void GetCaption(TArray<FString>& OutCaptions) const override;
	//~ End UMaterialExpression Interface
	#endif

	//~ Begin UMaterialExpressionCustomOutput Interface
	virtual int32 GetNumOutputs() const override;
	virtual FString GetFunctionName() const override;
	virtual FString GetDisplayName() const override;
	//~ End UMaterialExpressionCustomOutput Interface
};
```

## MaterialCompiler.h
Register the new Custom Output into `MaterialCompiler.h`

```cpp
virtual int32 CustomExpression(class UMaterialExpressionCustom* Custom, int32 OutputIndex, TArray<int32>& CompiledInputs) = 0;
virtual int32 CustomOutput(class UMaterialExpressionCustomOutput* Custom, int32 OutputIndex, int32 OutputCode) = 0;
virtual int32 VirtualTextureOutput(uint8 AttributeMask) = 0;

// SourceMod: Our Example Output here
virtual int32 ExampleOutput() = 0;
```
```cpp
virtual int32 CustomExpression(class UMaterialExpressionCustom* Custom, int32 OutputIndex, TArray<int32>& CompiledInputs) override { return Compiler->CustomExpression(Custom, OutputIndex, CompiledInputs); }
virtual int32 CustomOutput(class UMaterialExpressionCustomOutput* Custom, int32 OutputIndex, int32 OutputCode) override { return Compiler->CustomOutput(Custom, OutputIndex, OutputCode); }
virtual int32 VirtualTextureOutput(uint8 AttributeMask) override { return Compiler->VirtualTextureOutput(AttributeMask); }

// SourceMod: Our Example Output implementation here
virtual int32 ExampleOutput() override { return Compiler->ExampleOutput(); }
```

## MaterialExpressions.cpp
Now we need to add its definition and implement the class in `MaterialExpressions.cpp`

### Header
Include the header of our Example Output
```cpp
#include "Materials/MaterialExpressionExampleOutput.h"
```

> To be on the safe side, all code snippets should go in order and under:
> ```cpp
>FString UMaterialExpressionSingleLayerWaterMaterialOutput::GetDisplayName() const
>{
>	return TEXT("Single Layer Water Material");
>}
>```
{: .prompt-warning }

### Initializer
The only thing you may wanna change is `Name_Example` and `LOCTEXT("ExampleOutput", "ExampleOutput")` for Category during search in Material Graph.
```cpp
UMaterialExpressionExampleOutput::UMaterialExpressionExampleOutput(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	// Structure to hold one-time initialization
	struct FConstructorStatics
	{
		FText NAME_Example;
		FConstructorStatics()
			: NAME_Example(LOCTEXT("ExampleOutput", "ExampleOutput"))
		{
		}
	};
	static FConstructorStatics ConstructorStatics;

#if WITH_EDITORONLY_DATA
	MenuCategories.Add(ConstructorStatics.NAME_Example);
#endif

#if WITH_EDITOR
	Outputs.Reset();
#endif
}
```

### Material Compiler function

We check if any pin is connected, you can see it happening with `!OverrideBaseColor.IsConnected()`, if not we throw a compiler error `Compiler->Error(TEXT("No inputs to Example Output."));`\
<br>
In the next step we generate indexes for each matching pin, so we can use it in something like `BasePixelShader.usf`, if nothing is connected to that pin we return a default value.\
<br>
Finally we instruct the Compiler to compile the Output node with `Compiler->ExampleOutput();`
```cpp
#if WITH_EDITOR

int32 UMaterialExpressionExampleOutput::Compile(class FMaterialCompiler* Compiler, int32 OutputIndex)
{
	int32 CodeInput = INDEX_NONE;

	static const auto CVarStrata = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.Strata"));
	const bool bStrata = CVarStrata ? CVarStrata->GetValueOnAnyThread() > 0 : false;

	// We check if any pin is connected
	if (!OverrideBaseColor.IsConnected()
		&& !OverrideNormal.IsConnected()
		&& !bStrata)
	{
		Compiler->Error(TEXT("No inputs to Example Output."));
	}

	// Generates function names GetExampleOutput{index} used in BasePixelShader.usf.
	if (OutputIndex == 0)
	{
		CodeInput = OverrideBaseColor.IsConnected() ? OverrideBaseColor.Compile(Compiler) : Compiler->Constant3(0.f, 0.f, 0.f);;
	}
	else if (OutputIndex == 1)
	{
		CodeInput = OverrideNormal.IsConnected() ? OverrideNormal.Compile(Compiler) : Compiler->Constant3(0.f, 0.f, 1.f);
	}
	

	// We compile our ExampleOutput
	Compiler->ExampleOutput();
	return Compiler->CustomOutput(this, OutputIndex, CodeInput);
}

#endif // WITH_EDITOR
```

### GetCaption
```cpp
#if WITH_EDITOR

// Name that shows inside the material editor
void UMaterialExpressionExampleOutput::GetCaption(TArray<FString>& OutCaptions) const
{
	OutCaptions.Add(FString(TEXT("Example")));
}

#endif // WITH_EDITOR
```

### GetNumOutputs
```cpp
// Should match the number of Inputs into the Custom Output
int32 UMaterialExpressionExampleOutput::GetNumOutputs() const
{
	return 2;
}
```

### GetFunctionName
```cpp
// Setup for using in something like BasePixelShader.usf
// Example usage: GetExampleOutput1(MaterialParameters)
FString UMaterialExpressionExampleOutput::GetFunctionName() const
{
	return TEXT("GetExampleOutput");
}
```

### GetDisplayName
```cpp
// Display name of the node
FString UMaterialExpressionExampleOutput::GetDisplayName() const
{
	return TEXT("Example Output");
}
```

## MaterialShared.h

We need to find a way to check the presence of the Custom Output node.\
Can be done in `MaterialShared.h`

``` cpp
/** Whether the material uses NNE. */
LAYOUT_BITFIELD(uint8, bUsedWithNeuralNetworks, 1);
	
// SourceMod: Checking for presence of example output custom node
/** true if the material writes to an example custom output node. */
LAYOUT_BITFIELD(uint8, bHasExampleOutputNode, 1);
```

```cpp
bUsedWithNeuralNetworks(false),

// SourceMod: Initializing example output node to false (not in graph)
// Make sure it's always last
bHasExampleOutputNode(false)
```

## Definitions.usf

In preparation for the Translator, we need a definition that will be used in the HLSL files to branch between different results based on the presence of the Custom Output node or not.\
<br>
You can put this at the end of the file.
```cpp
// SourceMod: Example Output Define in global Definitions
#ifndef EXAMPLE_OUTPUT
#define EXAMPLE_OUTPUT										0
#endif
```

## HLSLMaterialTranslator.h 

We need to register our Custom Output node into the HLSLMaterialTranslator

```cpp
virtual int32 VirtualTextureOutput(uint8 MaterialAttributeMask) override;
	
// SourceMod: We implement our Custom Output node in the Translator
virtual int32 ExampleOutput() override;
```

## HLSLMaterialTranslator.cpp

Now we implement the Translator

```cpp
if (EnvironmentDefines->bVirtualTextureOutput)
{
	OutEnvironment.SetDefine(TEXT("VIRTUAL_TEXTURE_OUTPUT"), 1);
}

// SourceMod: Setting #define of previously declared EXAMPLE_OUTPUT in Definitions.usf to 1
if (MaterialCompilationOutput.bHasExampleOutputNode)
{
	OutEnvironment.SetDefine(TEXT("EXAMPLE_OUTPUT"), 1);
}
```
```cpp
int32 FHLSLMaterialTranslator::VirtualTextureOutput(uint8 AttributeMask)
{
	MaterialCompilationOutput.bHasRuntimeVirtualTextureOutputNode |= AttributeMask != 0;
	MaterialCompilationOutput.RuntimeVirtualTextureOutputAttributeMask |= AttributeMask;

	// return value is not used
	return INDEX_NONE;
}

// SourceMod: Implementation of the function, we set the LAYOUT_BITFIELD to 1
int32 FHLSLMaterialTranslator::ExampleOutput()
{
	MaterialCompilationOutput.bHasExampleOutputNode = 1;

	// return value is not used
	return INDEX_NONE;
}
```

## Testing the code

Compile the Engine, create a new Material and search the name of the Custom Output node, in this case `ExampleOutput`.

![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputsearch.png){: width="988" height="420" }
![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputingraph.png){: width="952" height="737" }

![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputnoinputs.png){: width="280" height="61" .w-50 .right}
As you can see, we get the error for not having any inputs connected to the Example Output node.\
If we connect anything to `OverrideBaseColor` or to `OverrideNormal` it'll compile.

# Making it actually work
## BasePassPixelShader.usf

The part where the magic happens is this one.\
We will assume we don't have Substrate enabled.

```cpp
// Store the results in local variables and reuse instead of calling the functions multiple times.
half3 BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
half  Metallic = GetMaterialMetallic(PixelMaterialInputs);
half  Specular = GetMaterialSpecular(PixelMaterialInputs);

float Roughness = GetMaterialRoughness(PixelMaterialInputs);
float Anisotropy = GetMaterialAnisotropy(PixelMaterialInputs);
uint ShadingModel = GetMaterialShadingModel(PixelMaterialInputs);
half Opacity = GetMaterialOpacity(PixelMaterialInputs);
```

We see the BaseColor that we'll need to override, but not the Normal.\
Chose the Normal specifically beacuse it's in a different file, but the structure is similar.
<br>

The end result should look like this:
```cpp
	// SourceMod: OverrideBaseColor shader implementation
#if !EXAMPLE_OUTPUT
	half3 BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
#else
	// Get Index 0 which in this case is OverrideBaseColor
	half3 BaseColor = GetExampleOutput0(MaterialParameters);
#endif
```

![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputnooverride.png){: width="1247" height="629" }
![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputoverride.png){: width="1198" height="633" }

## MaterialTemplate.ush

It's time to do the same for the Normal, which is located in `MaterialTemplate.ush`.
```cpp
// Note that here MaterialNormal can be in world space or tangent space
float3 MaterialNormal = GetMaterialNormal(Parameters, PixelMaterialInputs);
```

Following the same we did above, the end result will look like this:
```cpp
	// Note that here MaterialNormal can be in world space or tangent space
	// SourceMod: Example material template branch implementation
#if !EXAMPLE_OUTPUT
	float3 MaterialNormal = GetMaterialNormal(Parameters, PixelMaterialInputs);
#else
	// Get Index 1 which in this case is OverrideNormal
	float3 MaterialNormal = GetExampleOutput1(Parameters);
#endif
```

![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputnonormal.png){: width="1259" height="771" }
![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputnormal.png){: width="1260" height="772" }

## Conclusions

In conclusion, you now have the full power to pass any parameter into the Material Graph and modify how the Engine compiles a Material in HLSL.\
<br>
This is a very basic example that assumes both nodes will be connected, you can of course create your own expecions and checks for connections, making so only connected pins are considered for example.
<br>
Or you can create fancier effects, like lerping between BaseColor and OvverideBase color:
```cpp
// We use the OverrideNormal as an Injection example, ideally would create another ExampleOutput
// but instead we use the x value (the first one) of the Normal Vector for the Distance for the quick and dirty example
float Distance =  GetExampleOutput1(MaterialParameters).x;
float InMin = 1000.f;
float InMax = 2000.f;
float OutMin = 1.f;
float OutMax = 0.f;

float LerpAlpha = (Distance - InMin) * (OutMax - OutMin) / (InMax - InMin) + OutMin;


half3 BaseColor = 0;

BRANCH
if (LerpAlpha <= 0)
{
	BaseColor = GetExampleOutput0(MaterialParameters);
}
else if (LerpAlpha > 0 && LerpAlpha < 1 )
{
	BaseColor = lerp(GetExampleOutput0(MaterialParameters), GetMaterialBaseColor(PixelMaterialInputs), LerpAlpha);
}
else
{
	BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
}
```

![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputnodedistance.png){: width="643" height="677" }
![Desktop View](/assets/img/posts/2024-09-13-material-output-node/exampleoutputdistance.png){: width="1084" height="897" }

For any doubts or clarifications, don't hesitate to contact me!
