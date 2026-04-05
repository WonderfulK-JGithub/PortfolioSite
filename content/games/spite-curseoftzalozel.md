---
date: '2025-02-16T10:52:19+01:00'

tags: ["c++","custom engine"]
video: images/Tzalozel_Video.mp4
image: images/TCOT_Thumbnail2.png
title: 'Spite: The Curse of Tzalozel'
subtitle: "A desktop game made in custom game engine in 14 weeks with a team of 17. I mainly worked on UI functionality and Scene loading."
github: https://github.com/WonderfulK-JGithub/The-Curse-of-Tzalozel
#hideThumbnail: true

---

🎮 Link to game: [https://there-goes-the-goose.itch.io/spite-the-curse-of-tzalozel](https://there-goes-the-goose.itch.io/spite-the-curse-of-tzalozel)

My Contributions:
- **UI**
- **Scene Loading & Transitions**
- **Engine Maintenance**

This was my 5th game at The Game Assembly. It is a Diablo III inspired hack and slash where you traverse the jungle and defeat an evil goddess.

Between previous projects we were assigned to new teams, but starting with this project we would have the same team throughout the second year. We would also make our games in our own custom engine. This was challenging since we had to create tools for the engine and develop the game at the same time. My contributions were mainly on the engine side, such as creating our *UI editor* and *particle system editor*. On the game side I worked on the UI functionality, scene loading, and various minor elements such as the arena combat encounters.


## Scene loading

In our scene editor, every scene object is saved in a separate json file. This allows multiple designers to work on different parts of the levels without getting merge conflicts.

It has a drawback though and that is **loading times**. Loading thousands of different files takes a lot of time, especially if the filepaths are not loaded in windows cache. To solve this problem, without removing the benefits of multiple object files, I added a build process for our release version of the game. This process would go through every scene, and combine all scene objects into 1 larger json file.

![](../../images/TCOT_Build.png "build!")

This alone sped up loading times on average from 5 seconds to 1 second. I would iterate on this even more by saving the build file in a binary format instead of json. To achieve this I made a binary buffer class that was easy to read from and write to.

```cpp
namespace Goose
{
	class BinaryBuffer
	{
	public:
		void InputFromStream(std::ifstream& aIfStream);
		void OutputToStream(std::ofstream& aOfStream) const;

		char* data();
		const char* data() const;

		char* GetCurrentByteAddress();
		const char* GetCurrentByteAddress() const;

		bool HasEnoughBytes(size_t aSize) const;
		void EnsureBytes(size_t aSize);
		void IncrementPosition(size_t aCount);

		void ResetPosition();
		size_t GetCurrentPosition() const;
	private:
		std::vector<char> myData;
		size_t myCurrentPosition = 0;
	};
}
```

With this buffer, it was easy to serialize the scene data by using templated read & write functions.

```cpp
template<typename T>
void WriteData(BinaryBuffer& aBinaryBuffer, const T& aData);
template<typename T>
void WriteArrayData(BinaryBuffer& aBinaryBuffer, const T* aData, size_t aCount);

template<typename T>
void WriteData(BinaryBuffer& aBinaryBuffer, const T& aData)
{
	aBinaryBuffer.EnsureBytes(sizeof(T));
	std::copy_n(reinterpret_cast<const char*>(&aData), sizeof(T), aBinaryBuffer.GetCurrentByteAddress());
	aBinaryBuffer.IncrementPosition(sizeof(T));
}

template<typename T>
void WriteArrayData(BinaryBuffer& aBinaryBuffer, const T* aData, const size_t aCount)
{
	const size_t byteSize = aCount * sizeof(T);
	aBinaryBuffer.EnsureBytes(byteSize);
	std::copy_n(reinterpret_cast<const char*>(aData), byteSize, aBinaryBuffer.GetCurrentByteAddress());
	aBinaryBuffer.IncrementPosition(byteSize);
}

void Goose::Serialization::WriteData(BinaryBuffer& aBinaryBuffer, const SimpleMap<SceneInstanceID, GameSceneObjectInstance>& aData)
{
	const SmallSizeType mapSize = static_cast<uint16_t>(aData.size());

	Base::WriteData(aBinaryBuffer, mapSize);
	Base::WriteArrayData(aBinaryBuffer, aData.dataKeys(), aData.size());

	for (int i = 0; i < static_cast<int>(aData.size()); ++i)
	{
		const GameSceneObjectInstance& instance = aData.data()[i];

		Base::WriteData(aBinaryBuffer, instance.transform);
		Base::WriteData(aBinaryBuffer, instance.prefabId);

		const SmallSizeType nameLength = static_cast<uint16_t>(instance.name.size());
		Base::WriteData(aBinaryBuffer, nameLength);
		Base::WriteArrayData(aBinaryBuffer, instance.name.c_str(), instance.name.size());
	}
}
```

Storing the scene data in binary reduced load times even further, from 1 second to around 0.05 (almost instant). This way of serializing was also easy to write which later allowed me to store models and animations in binary as well, which was also significantly faster than loading from fbx.

## Trailer

{{< youtube id=kpPr7Tqi7X4   >}}