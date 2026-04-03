---

tags: ["tool","c++","custom engine"]
image: images/Custom_Editor.png
date: '2020-02-16T10:52:19+01:00'
title: 'Custom Editor'

---

## Overview 

During my second year at The Game Assembly, me and my project team made our own game engine and editor using c++, nhloman json and Dear ImGui. We started of using the schools engine as a base, but after months of working together our editor has become so much more. From a editor with only a primitive scene view with basic prefabs, we now have a editor with

- Scenes with realtime lighting
- Perfabs that can contain multiple sub objects
- Material assets
- Particle System Editor
- UI Editor
- Animation Tree Editor
- Model and Texture preview

Working on the editor was something I enjoyed. Some of my features, like the **Particle System** and the **UI Editor** have their own pages. Below here are some other of my contributions that might also be interesting:

## Component Reflection

In our games, we use a component system to store all objects data. These component are added to prefabs in the editor (Simuraly to Unity). Since c++ does not have inbuild reflection like other languages (expcept recent experimental versions), making components editable in the editor was a tedious process for us programmers. Thankfully after multiple iterations, I ended up making a simple and fast pipline. Now we only have to register the component with two macros in one file...

```cpp
DECLARE_COMPONENT_TYPE(SpriteComponent) // Declares the component

IMPLEMENT_COMPONENT_TYPE(SpriteComponent, "Sprite Component",nullptr) // Defines the component with name and defualt value
```

... and write one function that defines what variables to expose to the editor.

```cpp
struct SpriteComponent
{
	MaterialReferenceProperty spriteMaterial{ Goose::NULL_MATERIAL };
	Tga::Color color;
	Tga::Vector2f size;
	Tga::Vector2f pivot;
};

template<>
inline void ECS::OnReflectComponent(SpriteComponent& aComponent)
{
	Goose::Reflection::ReflectProperty(aComponent.size, "Size");
	Goose::Reflection::ReflectProperty(aComponent.pivot, "Pivot");
	Goose::Reflection::ReflectProperty(aComponent.color, "Color");
	Goose::Reflection::ReflectProperty(aComponent.spriteMaterial, "Material");
}
```

This is all we need to write in order to expose these variables to the editor

*INSERT IMAGE*

This works because the ReflectProperty() function is templated and because it's used for saving, loading and displaying ImGui. We only have to write a Load, Save or display function if we wan't to expose a new variable data type.

```cpp
template<typename T>
void ReflectProperty(T& aValue, const char* aName, bool aSave = true, bool aDisplay = true, bool aOverride = false)
{
	const uint32_t addressDiff = reinterpret_cast<char*>(&aValue) - globalDataStartAddress;

	switch (globalReflectionMode)
	{
	case ReflectionMode::Load:
			if (aSave)
			{
				LoadProperty(aValue, *globalJsonSaveData, aName);
			}
		break;
	case ReflectionMode::Save:
			if (aSave)
			{
				SaveProperty(aValue, *globalJsonSaveData, aName);
			}
		break;
	case ReflectionMode::Display:
	{
		if (CanEdit(globalPropertyIndex) && aDisplay)
		{
			Goose::ImGuiUtil::PropertyTable::NextProperty();
			globalInfoFlag |= PreEdit(globalPropertyIndex);

			if (ImGui::GetIO().KeyCtrl)
			{
				ImGui::Indent(20.0f);
			}
			
			const uint8_t propertyInfoFlag = PropertyEdit::DisplayImGuiPropertyEdit(aValue, aName);

			if (ImGui::GetIO().KeyCtrl)
			{
				ImGui::Unindent(20.0f);
			}
			
			if (propertyInfoFlag & PropertyEdit::InfoFlag_EditEnded)
			{
				globalOverridesEditFlag |= (1u << globalPropertyIndex);
			}
			globalInfoFlag |= propertyInfoFlag;

			ImGui::PopStyleColor();
		}
	}
		break;
	case ReflectionMode::OverrideMerge:
	{
		if (((globalAllowedOverridesFlag & (1u << globalPropertyIndex)) && (globalOverridesEditFlag & (1u << globalPropertyIndex))) || aOverride)
		{
			aValue = *reinterpret_cast<const T*>(globalOverrideData + addressDiff);
		}
		else
		{
			aValue = *reinterpret_cast<const T*>(globalPrefabData + addressDiff);
		}
	}
	break;
	case ReflectionMode::RemoveOverride:
		{
			if (((globalAllowedOverridesFlag & (1u << globalPropertyIndex))) == 0u)
			{
				aValue = *reinterpret_cast<const T*>(globalOverrideData + addressDiff);
			}
			else
			{
				aValue = *reinterpret_cast<const T*>(globalPrefabData + addressDiff);
			}
		}
		break;
	}

	PropertyEdit::LocalInfo::currentAttribute = PropertyEdit::LocalInfo::defaultAttribute;
	globalPropertyIndex++;
}

```

## Game Instanse Editor



<!-- ## Asset Registry

We store all of our game assets in a **Asset Reistry** class. This class is generic, making it easy to apply to any kind of asset. It stores all assets, their names and their current file paths in map containers. Every asset has its own unsiged integer ID (typedef:ed to UniversalInstanceID) stored in its json data, and will be registered on start up and on creation.

There is also support for regestring assets copies ,either with a set id or a runtime generated id, skipping the need for json and allowing to directly define the asset data. We use this when registring default Models/Materials that will always exist and are defined in code.

```cpp

using AssetIndex = uint32_t;

constexpr UniversalInstanceID ASSET_RUNTIME_START = 121u << 24u;

template<typename T>
class AssetRegistry
{
public:
	void RegisterAsset(const std::filesystem::path& aFilePath);
	void RegisterAsset(const nlohmann::json& aAssetJson);
	void DeRegisterAsset(const std::filesystem::path& aFilePath);

	UniversalInstanceID RegisterAssetCopy(const T& aAsset);
	UniversalInstanceID RegisterAssetMove(T&& aAsset);
	void RegisterAssetCopy(const T& aAsset, UniversalInstanceID aID,const char* aName);

	UniversalInstanceID GetIDFromString(const char* aName) const;
	const char* GetStringFromID(UniversalInstanceID aID) const;
	T& AccessAsset(AssetIndex aIndex);
	T& AccessAssetFromID(UniversalInstanceID aId);
	AssetIndex GetIndexFromID(UniversalInstanceID aId);

	const FilePath& GetAssetFilePath(UniversalInstanceID aId);
	void UpdateAssetFilePath(const FilePath& aNewFilePath);

	bool AssetExists(UniversalInstanceID aId);
	bool HasRegisteredAsset(const char* aName) const;
	void UpdateRegisteredAsset(const char* aOldName, const char* aNewName);

	const MappedVector<UniversalInstanceID, Tga::StringId>& GetIdToNameMap() const;
	MappedVector<UniversalInstanceID, T, AssetIndex>& AccessAssetMap();
private:
	MappedVector<UniversalInstanceID, T, AssetIndex> myAssetsMap;
	MappedVector<UniversalInstanceID, Tga::StringId> myIdToName;
	std::unordered_map<UniversalInstanceID, FilePath> myIdToFilePath;
	
	std::unordered_map<Tga::StringId, UniversalInstanceID> myNameToId;

	UniversalInstanceID myCopyIdCounter = ASSET_RUNTIME_START;
};
```

When giving assigning an id to an asset on file creation, it's read from an interal counter that then in -->



