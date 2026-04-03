---

tags: ["tool","ui","c++","custom engine"]
image: images/UI_Animated_Thumbnail.gif
date: '2025-02-16T10:52:19+01:00'
title: 'UI Editor'
subtitle: "a UI editor made in custom engine"
github: https://github.com/WonderfulK-JGithub/UI-Editor
---

## Overview 

I made this editor for our teams engine while working on *Spite: Curse of Tzalozel*. During previous game project in the school engine, UI element placements were often hardcoded and could not be changed in the editor. My goal with this UI editor was to allow our artists to set the layout for our UI.

## UI Elements

All UI elements have properties for position, scale and pivot.

![](../../images/UI_Position_Size_Pivot.png "position scale pivot")

Every element also has a name that can be edited in a inputfeild, and can be anchored to a part of the screen. 

![](../../images/UI_AnchorType.png "name and anchor")

To view how the anchors affect the elements, it is possible to change the reference resolution.

![](../../images/UI_Anchoring.gif "anchor")

All elements exists in layers, allowing controll over draw order of elements.

![](../../images/UI_Layers.png "anchor")

## Image & Text

Images have properties for sprite material and tint color

![](../../images/UI_Image.png "image")

Text elements have properties for display text, font, fontsize, alignment and tint color. The rendered text will automatically wrap to fit the width of the text element.

![](../../images/UI_Text.png "Text")

## Button

Buttons have properties for both image and text. Additionaly, there is a extra color tint for when the button is hovered, a click event property dictating what the button does and a property for what sfx to play.

![](../../images/UI_Button.png "Button")

## Slider & Checkbox

Sliders have image properties for both the backdrop and the handle. The handle can also be scaled independently from the backdrop.

![](../../images/UI_Slider.png "slider")

Checkboxes have properties for the backdrop material, default color and hover color. There is also properties for the checkmark sprite.

![](../../images/UI_Checkbox.png "checkbox")

## Editing in code

When programming the UI, I often needed access to the element in code. In the beggining, this was done by simply looping through all layers in a specifik canvas, looking for an element with a hardcoded name, and saving the layer index and ID, like this:

```cpp
const Tga::StringId sliderNameToCheck = "Slider"_tgaid;
Goose::UIID sliderId{};
uint32_t sliderLayerIndex{};

for (uint32_t layerIndex = 0; layerIndex < canvas.layers.size(); ++layerIndex)
{
	auto& layer = canvas.layers[layerIndex];
	for (uint32_t i = 0; i < layer.sliders.size(); ++i)
	{
		if (layer.sliders.data()[i].baseData.name == sliderNameToCheck)
		{
			sliderId         = layer.sliders.dataKeys()[i];
			sliderLayerIndex = layerIndex;
		}
	}
}
```

This method quickly became messy once I needed more references to elements. I improved this by creating a Reference class and from the editor generating constant references of elements in a header file. This would be done for all UI element marked as exposed

![](../../images/UI_Exposed.png "exposed")

This made everything more readable and also had the benefit of giving compile errors when elements were renamed or removed, making it easier to find and fix.

```cpp
namespace UI_VARIABLES
{
	namespace UI_KJ
	{
		constexpr Goose::UIReference Slider_To_Expose{ 7,1,3,218103821 };

	}
}

// Example of how to access
void PrintSliderValue()
{
	std::cout << UI_VARIABLES::UI_KJ::Slider_To_Expose.AccessSlider().value << "\n";
}
```

The idea of generating constants in a header file is something I got from working with the Wwise audio engine, which simularly creates constants for every audio event. How I generate the file is by continusly adding text to a string. I wrap every canvas in it's own namespace and creating variables for every exposed element based on their names.

```cpp
void Goose::GenerateCanvasHeader()
{
	const std::string tabString = "    ";
	const std::string doubleTabString = "        ";
	std::string fileContent = "#pragma once\n#include \"UIReference.h\"\nnamespace UI_VARIABLES\n{\n" + tabString;

	auto& allCanvases = GetCanvasRegistry().AccessAssetMap();

	for (uint32_t canvasIndex = 0; canvasIndex < allCanvases.size(); ++canvasIndex)
	{
		const UICanvas& canvas = allCanvases.data()[canvasIndex];

		const CanvasID canvasId = allCanvases.dataKeys()[canvasIndex];

		FilePath canvasName = GetCanvasRegistry().GetStringFromID(canvasId);
		canvasName.replace_extension("");

		fileContent += "namespace " + canvasName.string() + "\n" + tabString + "{\n" + doubleTabString;

		int layerIndex = 0;
		for (auto& layer : canvas.layers)
		{
			auto exposeToFile = [&](const UIBaseData& aUIObject, const UniversalInstanceID aId, UIType aType)
			{
				String64 objectName = aUIObject.name.GetString();
				objectName.replace(' ', '_');

				fileContent += "constexpr Goose::UIReference " + std::string(objectName.c_str()) + "{"
					+ std::to_string(aId) + "," + std::to_string(layerIndex) + "," + std::to_string(static_cast<int>(aType))
				 + "," + std::to_string(canvasId) + "};\n" + doubleTabString;
			};

			for (int i = 0; i < static_cast<int>(layer.images.size()); ++i)
			{
				const UIBaseData& uiObject = layer.images.data()[i].baseData;
				
				if (uiObject.exposeToHeader == false)
				{
					continue;
				}

				const UniversalInstanceID id = layer.images.dataKeys()[i];

				exposeToFile(uiObject, id, UIType::Image);
			}
			for (int i = 0; i < static_cast<int>(layer.texts.size()); ++i)
			{
				const UIBaseData& uiObject = layer.texts.data()[i].baseData;
				const UniversalInstanceID id = layer.texts.dataKeys()[i];

				if (uiObject.exposeToHeader == false)
				{
					continue;
				}

				exposeToFile(uiObject, id, UIType::Text);
			}

			for (int i = 0; i < static_cast<int>(layer.buttons.size()); ++i)
			{
				const UIBaseData& uiObject = layer.buttons.data()[i].baseData;
				const UniversalInstanceID id = layer.buttons.dataKeys()[i];

				if (uiObject.exposeToHeader == false)
				{
					continue;
				}

				exposeToFile(uiObject, id, UIType::Button);
			}

			for (int i = 0; i < static_cast<int>(layer.sliders.size()); ++i)
			{
				const UIBaseData& uiObject = layer.sliders.data()[i].baseData;
				const UniversalInstanceID id = layer.sliders.dataKeys()[i];

				if (uiObject.exposeToHeader == false)
				{
					continue;
				}

				exposeToFile(uiObject, id, UIType::Slider);
			}

			for (int i = 0; i < static_cast<int>(layer.checkboxes.size()); ++i)
			{
				const UIBaseData& uiObject = layer.checkboxes.data()[i].baseData;
				const UniversalInstanceID id = layer.checkboxes.dataKeys()[i];

				if (uiObject.exposeToHeader == false)
				{
					continue;
				}

				exposeToFile(uiObject, id, UIType::Checkbox);
			}

			layerIndex++;
		}

		fileContent += "\n" + tabString + "}\n" + tabString;
	}

	fileContent += "\n}";

	const FilePath headerPath = Tga::Settings::SourceRoot() / "Goose" / "UI" / "CanvasReferences.h";
	std::ofstream outPut(headerPath, std::ios::out | std::ios::trunc);
	outPut << fileContent.c_str();
	outPut.close();
}
```