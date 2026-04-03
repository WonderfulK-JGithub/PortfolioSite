---

tags: ["tool","c++","custom engine"]
image: images/Terrain_Animated_Thumbnail.gif
date: '2026-02-16T10:52:19+01:00'
title: 'Terrain Editor'
subtitle: "3D Terrain Editor using marching cubes algorithm"
github: https://github.com/WonderfulK-JGithub/Terrain-Editor
---

## Overview 

This was my specialization project at The Game Assembly. The terrain editor lets the user create terrain both by rasing and lowering terrain, but also by sculpting in 3D. Instead of using a heightmap, the terrain is generated with the Marching Cubes algorithm. This makes it possible to sculpt things like tunnels and overhangs, something that would not be possible using a heightmap.

Hear is a in order breakdown of how I created this terrain editor:

## Creating a mesh

My first step was to implement the marching cubes algorithm. The algorithm works by taking a 3D grid of values, in my case ranging from 0 to 1. It then goes through all the values. For each point it checks the neigbouring 7 points, making a local cube out of all the points. The algorithm then creates triangles inside this cube depending on the values of the points.
 
![](../../images/Terrain_MarchingShowcase.gif "Visual representation of the marching cubes algorithm")


I started of with a fixed grid that would create a wave like patern

```cpp
constexpr uint32_t size = 17;
std::vector<float> testValues;
testValues.resize(size * size * size);
for (uint32_t z = 0; z < size; ++z)
{
	for (uint32_t y = 0; y < size; ++y)
	{
		for (uint32_t x = 0; x < size; ++x)
		{
			testValues[size * size * z + size * y + x] = 
			FMath::Clamp((static_cast<float>(y) + sin(static_cast<float>(x) * 0.6f) 
			* cos(static_cast<float>(z) * 0.6f)) / 16.f, 0.f, 1.f);
		}
	}
}
```

I then made my first implementation of the algorithm.  

```cpp
constexpr float surfaceLevel = 0.5f;

void TerrainCreation::GenerateMesh(Goose::ModelAsset::MeshData& aMeshData, const std::vector<float>& aVoxelValues, 
const uint32_t aSize)
{
	std::vector<Tga::Vector3f> vertexPositions;
	const uint32_t squaredSize = aSize * aSize;
	const uint32_t cellSize    = aSize - 1;

	for (uint32_t z = 0; z < cellSize; ++z)
	{
		for (uint32_t y = 0; y < cellSize; ++y)
		{
			for (uint32_t x = 0; x < cellSize; ++x)
			{
				uint32_t cubeIndex{};
				float dataCube[8]{};

				for (uint32_t i = 0; i < std::size(localCornerChecks); i++)
				{
					const Tga::Vector3ui& check = localCornerChecks[i];
					dataCube[i]                 = aVoxelValues[(z + check.z) * squaredSize + 
					(y + check.y) * aSize + x + check.x];
					if (dataCube[i] > surfaceLevel)
					{
						cubeIndex |= 1 << i;
					}
				}

				const Tga::Vector3f origin(static_cast<float>(x), 
				static_cast<float>(y), 
				static_cast<float>(z));

				for (int a = 0; localTriangleTable[cubeIndex][a] != -1; a += 3)
				{
					const int startEdgeIndex0 = 
					localEdgeIndices[localTriangleTable[cubeIndex][a]][0];
					const int endEdgeIndex0   = 
					localEdgeIndices[localTriangleTable[cubeIndex][a]][1];

					const int startEdgeIndex1 = 
					localEdgeIndices[localTriangleTable[cubeIndex][a + 1]][0];
					const int endEdgeIndex1   = 
					localEdgeIndices[localTriangleTable[cubeIndex][a + 1]][1];

					const int startEdgeIndex2 = 
					localEdgeIndices[localTriangleTable[cubeIndex][a + 2]][0];
					const int endEdgeIndex2   = 
					localEdgeIndices[localTriangleTable[cubeIndex][a + 2]][1];

					vertexPositions.emplace_back(origin + FMath::Lerp(cornerTable[startEdgeIndex0], 
					cornerTable[endEdgeIndex0],
					(surfaceLevel - dataCube[startEdgeIndex0]) / (
					dataCube[endEdgeIndex0] - dataCube[startEdgeIndex0])));

					vertexPositions.emplace_back(origin + FMath::Lerp(cornerTable[startEdgeIndex1], 
					cornerTable[endEdgeIndex1],
					(surfaceLevel - dataCube[startEdgeIndex1]) / (
					dataCube[endEdgeIndex1] - dataCube[startEdgeIndex1])));

					vertexPositions.emplace_back(origin + FMath::Lerp(cornerTable[startEdgeIndex2], 
					cornerTable[endEdgeIndex2],
					(surfaceLevel - dataCube[startEdgeIndex2]) / (
					dataCube[endEdgeIndex2] - dataCube[startEdgeIndex2])));
				}
			}
		}
	}
}
```

To verify that the algorithm was working I first rendered triangles with debug lines.
![](../../images/Terrain_DebugMesh.png "terrain debug lines")

When the algorithm worked correctly I then generated a mesh with the vertecies and rendered it with a basic material.

![](../../images/Terrain_Hard.png "terrain mesh")

## Smooth shading

So far the terrain mesh did not share vertices between triangles, making the terrain flat shaded. This is not uncommon for marching cubes terrain in general but I wanted my terrain to be smooth. To do this, I needed to create a VertexID and a map going from VertexID to index inside the vertexPosition vector.

```cpp
struct VertexID
{
	Tga::Vector3<TerrainCreation::ChunkSizeType> pos1;
	Tga::Vector3<TerrainCreation::ChunkSizeType> pos2;

	bool operator==(const VertexID& aId) const
	{
		return pos1 == aId.pos1 && pos2 == aId.pos2;
	}
};
```
```cpp

std::vector<Tga::Vector3f> vertexPositions;
std::unordered_map<VertexID, uint32_t> idToIndex;
```
```cpp

const Tga::Vector3f origin(x, y, z);
const Tga::Vector3 baseId{ x,y,z };
for (int a = 0; localTriangleTable[cubeIndex][a] != -1; a++)
{
	const int startEdgeIndex0 = localEdgeIndices[localTriangleTable[cubeIndex][a]][0];
	const int endEdgeIndex0   = localEdgeIndices[localTriangleTable[cubeIndex][a]][1];

	VertexID id{ .pos1 = baseId + cornerChecks[startEdgeIndex0],.pos2 = baseId + cornerChecks[endEdgeIndex0] };

	const auto [iterator, result] = idToIndex.insert({ id, static_cast<uint32_t>(vertexPositions.size()) });

	if (result)
	{
		vertexPositions.emplace_back(origin + FMath::Lerp(cornerTable[startEdgeIndex0], cornerTable[endEdgeIndex0],
			(surfaceLevel - dataCube[startEdgeIndex0]) / (
				dataCube[endEdgeIndex0] - dataCube[startEdgeIndex0])));
	}

	aMeshData.Indices.emplace_back(iterator->second);
}

```

![](../../images/Terrain_SmoothShaded.png "smooth terrain")

## The first tools

When I had a mesh to use as reference I moved on to creating the editing tools. The first one I made was a 3D sculpting tool that would raise the value of nearby points.

![](../../images/Terrain_3DSculpt.gif "sculpt in 3d")

I then created a tool that would lower and raise terrain as if it was a heightmap.

![](../../images/Terrain_2DSculpt.gif "reaise and lower terrain")

## Multiple chunks

The next step was to increase the area you could edit in. Since the mesh has to be generated every time a value in the grid is changed, the terrains mesh needed to be divided into different chunks to make it more performant.

With chunks, a mesh would only be generaed if the changed values were in that chunk (represented by the greeen debug lines in the image bellow).

![](../../images/Terrain_Chunks2.png "chunks")

## Erase, Flatten and Smooth

With a bigger terrain to edit on it was time to implement more tools to edit with.

I made a tool to erase sculpting done..

![](../../images/Terrain_Erase.gif "erase")

.., a Tool to flatten terrain..

![](../../images/Terrain_Flatten.gif "flatten")

.., and a tool to make the terrain more smooth.

![](../../images/Terrain_Smoothing.gif "smoothing")

I also made a 3D varaint of the smoothing tool that is better to use on more 3D sculpted terrain.

![](../../images/Terrain_Smooth3D.gif "smoothing 3D")


## Ramp and Tunnel

I then moved on to making more complex tools, starting of with a ramp tool. The tool allows the user to select two points on the terrain, move those points if necisary, and create a ramp based on the line between the points

![](../../images/Terrain_Ramp.gif "ramp")

From the ramp tool I reused the point selection functionality to make a new toot. This one removes terrain between the points, creating a tunnel

![](../../images/Terrain_Tunnel.gif "tunnel")

## Textures

To calculate the UVs for the terrain, I check what axis the normal is most aligned to using the dot product. From this you can use world position as uv, and get a accurate tangent and binormal by using the cross product.

```hlsl
float2 uv;

float3 tangent;
float3 binormal;

if (dotX > dotY && dotX > dotZ)
{
    uv = input.worldPosition.zy;
    tangent = normalize(cross(input.normal, float3(0.f, 1.f, 0.f)));
    binormal = cross(tangent, input.normal);
}
else
{
	if (dotY > dotX && dotY > dotZ)
	{
        uv = input.worldPosition.zx;
        tangent = normalize(cross(input.normal, float3(1.f, 0.f, 0.f)));
        binormal = cross(tangent, input.normal);
    }
	else
	{
        uv = input.worldPosition.xy;
        tangent = normalize(cross(input.normal, float3(0.f, 1.f, 0.f)));
        binormal = cross(tangent, input.normal);
    }
}
```

I choose to do this inside the pixel shader, instead of pre calculating when genereting the mesh, since vertices are shared which made this approach look weird

![](../../images/Terrain_UV.png "textured terrain")


## Chunk bugfix

One problem I had when seperating the terrain into chunks was that vertecies on the chunk's edge had incorrect normals, causing lighting issues like this:

![](../../images/Terrain_ChunkFail.png "epic chunk fail")

I solved this by adding a extra loop along the points on the edge when generating the terrain, checking with points 1 step outside the chunk. These checks would not add new triangles, but would affect the normals of the vertices if they existed in the chunk.

![](../../images/Terrain_ChunkFix.png "epic chunk fix")

## Optimizations

Editing many chunks at once is very performance heavy. To increase performance I seperated chunk generations into different tasks that would be executed on multiple threads in a thread pool, massivly increasing performance.

```cpp
void Goose::TerrainDocument::GenerateChunks(const std::vector<Tga::Vector3<uint32_t>>& aChunks)
{
	TerrainCreation::GenerationContext context;
	context.voxelValues = myTerrain.AccessVoxelValues().data();
	context.chunkSize = myTerrain.GetChunkSize();
	context.voxelDimensions = myTerrain.GetVoxelDimensions();
	context.unitsPerVoxel = myTerrain.GetVoxelsPerUnitInverse();

	constexpr uint32_t chunksPerTask = 4;

	uint32_t currentChunkStart = 0;
	
	while (currentChunkStart + chunksPerTask < aChunks.size())
	{
		auto task = [context,currentChunkStart, this,&aChunks]()
		{
			const uint32_t zChunkSize = myTerrain.GetChunkDimensions().y * myTerrain.GetChunkDimensions().x;
			TerrainCreation::GenerationContext localContext = context;
			for (uint32_t i = currentChunkStart; i < currentChunkStart + chunksPerTask; ++i)
			{
				auto& chunk = aChunks[i];
				
				localContext.voxelStart = Tga::Vector3<uint32_t>{ chunk.x * myTerrain.GetChunkSize(), chunk.y * myTerrain.GetChunkSize(), chunk.z * myTerrain.GetChunkSize() };
				const uint32_t chunkIndex = zChunkSize * chunk.z + chunk.y * myTerrain.GetChunkDimensions().x + chunk.x;
				GenerateChunk(myTerrain.AccessChunks()[chunkIndex], localContext);
			}
		};

		ThreadPool::GetGlobalInstance().ScheduleTask(task);

		currentChunkStart += chunksPerTask;
	}

	uint32_t chunksLeft = aChunks.size() - currentChunkStart;

	if (chunksLeft > 0)
	{
		auto task = [context, currentChunkStart, chunksLeft, this, &aChunks]()
		{
			const uint32_t zChunkSize = myTerrain.GetChunkDimensions().y * myTerrain.GetChunkDimensions().x;
			TerrainCreation::GenerationContext localContext = context;
			for (uint32_t i = currentChunkStart; i < currentChunkStart + chunksLeft; ++i)
			{
				auto& chunk = aChunks[i];

				localContext.voxelStart = Tga::Vector3<uint32_t>{ chunk.x * myTerrain.GetChunkSize(), chunk.y * myTerrain.GetChunkSize(), chunk.z * myTerrain.GetChunkSize() };
				const uint32_t chunkIndex = zChunkSize * chunk.z + chunk.y * myTerrain.GetChunkDimensions().x + chunk.x;
				GenerateChunk(myTerrain.AccessChunks()[chunkIndex], localContext);
			}
		};

		ThreadPool::GetGlobalInstance().ScheduleTask(task);
	}

	ThreadPool::GetGlobalInstance().RunUntilQueEmpty();
}
```

I also made a small performance increase by changing the grid layout from z-y-x to z-x-y. When looping through the values, the inner for loop would be changed to y instead:
```cpp
for (uint32_t z = 0; z < size; ++z)
{
	for (uint32_t x = 0; x < size; ++x)
	{
		for (uint32_t y = 0; y < size; ++y)
		uint32_t index = size * size * z + size * y + x;


```
Having the grid this way ensures the merory is aligned along the y axis. This makes the code for checking and adding height to the terrain more cache friendly, and made the 2d tools noticably faster.