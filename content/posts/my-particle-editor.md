---

tags: ["tool","c++"]
image: images/Particle_Animated_Thumbnail.gif
date: '2024-02-16T10:52:19+01:00'
title: 'Particle System Editor'
subtitle: "A particle system editor made from the ground up in custom engine"
github: https://github.com/WonderfulK-JGithub/Particle-Editor
---

## Overview 

I made this particle system for my teams game engine. There was no way for our procedural artist to create partciles in our schools game engine, so I made this completly from scratch. When creating this tool I took inspiration from Unitys particle system when deciding what to add.

## Shape & Emission

All particles always spawn with a set lifetime. This value can be the same for every particle, or randomly generated at spawn. Every property in the particle system that can be random has a options to choose between a fixed **Value**, a random value in a **Range** or a randomly chosen value of **Either** left or right value.

![white kitten](../../images/Particle_Lifetime.png "")

To spawn the particles I have settings for when to spawn them in **Emission** and where to spawn them in **Shape**. In emission, you can edit how many particles to spawn per seconds, how many per distance (Used to spawn when physically moving the particle system) and bursts. For burst emissions you can set how many burst in total, how many particles in every burst and the time between every burst.

If **Emit World** is checked particles will move independetly once spawned. Otherwise they will move locally to the transform of the particle system.

![white kitten](../../images/Particle_Emission.png "")

In Shape, you can choose between emitting in a sphere, cone or box.There are more settings for every shape type. The cone has settings for radius and height, and a arc setting that can controll angle thing. You can also controll thickness, which when smaller will make particles spawn more to the edge of the shape. 

The direction the particles spawn with is determined by their spawn position and the emitors shape type At the request of our procedural artists, I also added an option to spawn particles with a fixed direction.

![white kitten](../../images/Particle_Shape.png "")

## Speed, Size & Rotation

Speed, size and rotation can all be edited for both start values and over time change. For the overtime values, I added an option to make a **curve** for the value. The curve can have points moved, added and removed. You can also choose if interpolation between points is linear or hermite.

![white kitten](../../images/Particle_Curve.gif "")

This is the code for the Curve, both how it displays in editor and how it evalutes a value:

```cpp
namespace Goose
{
	struct CurvePoint
	{
		float value   = 0.f;
		float tangent = 0.f;
		float time    = 0.f;
	};

	class FunctionCurve
	{
	public:
		float Evaluate(float aTime) const;

		std::vector<CurvePoint>& AccessPoints();
		const std::vector<CurvePoint>& AccessPoints() const;

		bool IsLinear() const;
		void SetIsLinear(bool aLinear);

		uint8_t DisplayImGuiEdit(const char* aId);
	protected:
		void Sort();
		uint8_t DisplayCurveEdit();

		std::vector<CurvePoint> myPoints{{}, {.value = 1.f, .time = 1.f}};
		float myMinValue{ 0.f };
		float myMaxValue{ 1.f };
		bool myIsLinear = false;
	};
}


static float HermiteInterpolate(const Goose::CurvePoint& k0, const Goose::CurvePoint& k1, const float t)
{
	const float dt = k1.time - k0.time;
	const float m0 = -k0.tangent * dt;
	const float m1 = -k1.tangent * dt;
	const float u  = (t - k0.time) / dt;

	const float u2 = u * u;
	const float u3 = u2 * u;

	// Cubic Hermite spline basis
	const float h00 = 2 * u3 - 3 * u2 + 1;
	const float h10 = u3 - 2 * u2 + u;
	const float h01 = -2 * u3 + 3 * u2;
	const float h11 = u3 - u2;

	const float result = h00 * k0.value + h10 * m0 + h01 * k1.value + h11 * m1;
	return result;
}

float Goose::FunctionCurve::Evaluate(const float aTime) const
{
	if (aTime <= myPoints.front().time)
	{
		return myPoints.front().value;
	}
	if (aTime >= myPoints.back().time)
	{
		return myPoints.back().value;
	}

	CurvePoint prevCurvePoint = myPoints.front();
	float returnValue{};

	for (const CurvePoint& curvePoint : myPoints)
	{
		returnValue = curvePoint.value;
		if (curvePoint.time > aTime)
		{
			if (aTime == 0.f)
			{
				break;
			}

			if (myIsLinear)
			{
				const float deltaT = aTime - prevCurvePoint.time;
				returnValue = prevCurvePoint.value + ((curvePoint.value - prevCurvePoint.value) / (curvePoint.time - prevCurvePoint.time)) *
				              deltaT;
			}
			else
			{
				returnValue = HermiteInterpolate(prevCurvePoint, curvePoint, aTime);
			}
			break;
		}
		prevCurvePoint = curvePoint;
	}

	return returnValue;
}

std::vector<Goose::CurvePoint>& Goose::FunctionCurve::AccessPoints()
{
	return myPoints;
}

const std::vector<Goose::CurvePoint>& Goose::FunctionCurve::AccessPoints() const
{
	return myPoints;
}

bool Goose::FunctionCurve::IsLinear() const
{
	return myIsLinear;
}

void Goose::FunctionCurve::SetIsLinear(const bool aLinear)
{
	myIsLinear = aLinear;
}

#pragma region ImGuiSection

constexpr ImU32 BACKGROUND_COLOR         = IM_COL32(130, 130, 130, 255);
constexpr ImU32 GRAPH_COLOR              = IM_COL32(30, 230, 30, 255);
constexpr ImU32 POINT_COLOR              = IM_COL32(60, 190, 60, 255);
constexpr ImU32 POINT_HOVER_COLOR        = IM_COL32(80, 200, 80, 255);
constexpr ImU32 POINT_DELETE_COLOR       = IM_COL32(190, 40, 40, 255);
constexpr ImU32 POINT_DELETE_HOVER_COLOR = IM_COL32(210, 60, 60, 255);
constexpr ImU32 TANGENT_COLOR            = IM_COL32(60, 160, 120, 255);
constexpr ImU32 TANGENT_HOVER_COLOR      = IM_COL32(800, 180, 180, 255);

constexpr char CURVE_POP_UP[] = "Curve Popup";

constexpr float MAX_TANGENT = 320.f;

struct CurveImGuiInfo
{
	ImVec2 myMouseTracking;
	int mySelectedPoint = -1;
	bool myDragTangent  = false;
};

static CurveImGuiInfo localInfo;

uint8_t Goose::FunctionCurve::DisplayImGuiEdit(const char* aId)
{
	ImGui::PushID(aId);

	if (globalIsUsingNodeEditor) // This does not display correctly in node editor and is disabled
	{
		ImGui::Text("_Variable_Only_");
		return 0u;
	}
	
	constexpr float spacingX = 10.f;
	constexpr float graphHeight = 30.f;

	const ImVec2 origin = ImGui::GetCursorScreenPos();
	const float totalWidth = ImGui::GetContentRegionAvail().x * 0.7f - spacingX * 2.f;

	const ImVec2 graphSize = ImVec2(totalWidth, graphHeight);

	ImDrawList* drawList = ImGui::GetWindowDrawList();

	drawList->AddRectFilled(origin, origin + graphSize, BACKGROUND_COLOR);

	for (size_t i = 0; i + 1 < myPoints.size(); ++i)
	{
		ImVec2 point1 = origin + ImVec2(totalWidth * myPoints[i].time,
			FMath::InverseLerp(myMaxValue, myMinValue, myPoints[i].value) * graphHeight);
		ImVec2 point2 = origin + ImVec2(totalWidth * myPoints[i + 1].time,
			FMath::InverseLerp(myMaxValue, myMinValue, myPoints[i + 1].value) * graphHeight);
		drawList->AddLine(point1, point2, GRAPH_COLOR, 1.f);
	}

	ImGui::Dummy(graphSize);

	

	const bool hovered = ImGui::IsItemHovered();
	if (hovered && ImGui::IsMouseClicked(ImGuiMouseButton_Left))
	{
		ImGui::OpenPopup(CURVE_POP_UP);
		ImGui::SetNextWindowPos(ImGui::GetCursorScreenPos());
		localInfo = CurveImGuiInfo();
	}
	uint8_t info = 0;
	/*ImGui::SetNextWindowFocus();*/
	if (ImGui::BeginPopup(CURVE_POP_UP, ImGuiWindowFlags_NoMove))
	{
		info |= DisplayCurveEdit();
		ImGui::EndPopup();
	}

	ImGui::PopID();

	return info;
}

void Goose::FunctionCurve::Sort()
{
	for (int i = 0; i < static_cast<int>(myPoints.size()); i++)
	{
		int bestIndex    = i;
		float lowestTime = myPoints[i].time;
		for (int j = i + 1; j < static_cast<int>(myPoints.size()); ++j)
		{
			if (myPoints[j].time < lowestTime)
			{
				bestIndex  = j;
				lowestTime = myPoints[j].time;
			}
		}

		std::swap(myPoints[i], myPoints[bestIndex]);
		if (bestIndex == localInfo.mySelectedPoint)
		{
			localInfo.mySelectedPoint = i;
		}
		else if (i == localInfo.mySelectedPoint)
		{
			localInfo.mySelectedPoint = bestIndex;
		}
	}
}

uint8_t Goose::FunctionCurve::DisplayCurveEdit()
{
	constexpr ImVec2 graphOffset    = ImVec2(20, 20);
	constexpr ImVec2 graphSize      = {300.f, 100.f};
	constexpr int graphCurveDetail  = 200;
	constexpr float pointSize       = 5.f;
	constexpr float tangentDistance = pointSize * 4.f;

	constexpr float sideItemsWidth = 50.f;
	constexpr float sideOffset     = 20.f;

	constexpr ImVec2 totalSize = {sideItemsWidth + sideItemsWidth + graphSize.x + graphOffset.x * 2.f, graphSize.y + graphOffset.y * 2.f};

	const ImVec2 origin           = ImGui::GetWindowPos() + graphOffset;
	const ImVec2 sideCursorOrigin = graphOffset + ImVec2(graphSize.x + sideOffset, 0.f);

	const bool deleteMode = ImGui::GetIO().KeyShift && myPoints.size() > 2;

	uint8_t info = 0;

	//Drawing (and deleting)
	bool hasHovered = false;
	{
		ImDrawList* drawList = ImGui::GetWindowDrawList();

		drawList->AddRectFilled(origin, origin + graphSize, BACKGROUND_COLOR);

		if (myIsLinear)
		{
			for (size_t i = 0; i + 1 < myPoints.size(); ++i)
			{
				ImVec2 point1 = origin + ImVec2(graphSize.x * myPoints[i].time,
				                                FMath::InverseLerp(myMaxValue, myMinValue, myPoints[i].value) *
				                                graphSize.y);
				ImVec2 point2 = origin + ImVec2(graphSize.x * myPoints[i + 1].time,
				                                FMath::InverseLerp(myMaxValue, myMinValue,
				                                                   myPoints[i + 1].value) * graphSize.y);
				drawList->AddLine(point1, point2, GRAPH_COLOR, 2.f);
			}

			const float firstY = FMath::InverseLerp(myMaxValue, myMinValue, myPoints.front().value) * graphSize.
			                     y;
			const float lastY = FMath::InverseLerp(myMaxValue, myMinValue, myPoints.back().value) * graphSize.y;

			drawList->AddLine(origin + ImVec2(0.f, firstY), origin + ImVec2(myPoints.front().time * graphSize.x, firstY), GRAPH_COLOR, 2.f);
			drawList->AddLine(origin + ImVec2(graphSize.x, lastY), origin + ImVec2(myPoints.back().time * graphSize.x, lastY), GRAPH_COLOR,
			                  2.f);
		}
		else
		{
			ImVec2 points[graphCurveDetail]{};

			constexpr float timeIncrement = 1.f / static_cast<float>(graphCurveDetail);
			float t                       = timeIncrement;

			for (int i = 0; i < graphCurveDetail; ++i)
			{
				points[i] = origin + ImVec2{
					            t,
					            FMath::InverseLerp(myMaxValue, myMinValue, Evaluate(t))
				            } * graphSize;
				t += timeIncrement;
			}

			drawList->AddPolyline(points, graphCurveDetail, GRAPH_COLOR, ImDrawFlags_None, 2.f);
		}

		const ImU32 defaultColor = deleteMode ? POINT_DELETE_COLOR : POINT_COLOR;
		const ImU32 hoverColor   = deleteMode ? POINT_DELETE_HOVER_COLOR : POINT_HOVER_COLOR;

		for (int i = 0; i < static_cast<int>(myPoints.size()); ++i)
		{
			CurvePoint& point = myPoints[i];

			ImVec2 drawPosition = origin + ImVec2(point.time, FMath::InverseLerp(myMaxValue, myMinValue,
			                                                                     point.value)) * graphSize;

			const bool selected = localInfo.mySelectedPoint == i;
			bool hovered        = false;
			if (hasHovered == false && ImGuiUtil::IsMouseOverCircle(drawPosition, pointSize))
			{
				hovered    = true;
				hasHovered = true;
			}

			drawList->AddCircleFilled(drawPosition, pointSize, hovered ? hoverColor : defaultColor);

			if (hovered && ImGui::IsMouseClicked(ImGuiMouseButton_Left))
			{
				if (deleteMode)
				{
					localInfo.mySelectedPoint = -1;
					myPoints.erase(myPoints.begin() + i);
					break;
				}
				localInfo.mySelectedPoint = i;
				localInfo.myMouseTracking = drawPosition;
				localInfo.myDragTangent   = false;
			}

			if (selected && myIsLinear == false)
			{
				const float angle = atanf(point.tangent);

				const ImVec2 tangentVector = {std::cosf(angle), std::sinf(angle)};

				ImVec2 tangentDrawPos = drawPosition + tangentVector * tangentDistance;

				const bool tangentHovered = ImGuiUtil::IsMouseOverCircle(tangentDrawPos, pointSize);
				hasHovered |= tangentHovered;

				drawList->AddCircleFilled(tangentDrawPos, pointSize, tangentHovered ? TANGENT_HOVER_COLOR : TANGENT_COLOR);

				if (tangentHovered && ImGui::IsMouseClicked(ImGuiMouseButton_Left))
				{
					localInfo.myMouseTracking = tangentDrawPos;
					localInfo.myDragTangent   = true;
				}
			}
		}
	}

	// Edit Points
	{
		if (ImGui::IsMouseClicked(ImGuiMouseButton_Left) && hasHovered == false)
		{
			if (localInfo.mySelectedPoint != -1)
			{
				info |= PropertyEdit::InfoFlag_EditEnded | PropertyEdit::InfoFlag_ChangedValue;
			}

			localInfo.mySelectedPoint = -1;
		}

		localInfo.myMouseTracking += ImGui::GetIO().MouseDelta;

		if (localInfo.mySelectedPoint != -1 && ImGui::IsMouseDown(ImGuiMouseButton_Left))
		{
			if ((ImGui::GetIO().MouseDelta.x != 0.f) || (ImGui::GetIO().MouseDelta.y != 0.f))
			{
				info |= PropertyEdit::InfoFlag_ChangedValue;
			}

			if (localInfo.myDragTangent)
			{
				const ImVec2 pointPosition = origin + ImVec2(myPoints[localInfo.mySelectedPoint].time,
				                                             FMath::InverseLerp(myMaxValue, myMinValue,
				                                                                myPoints[localInfo.mySelectedPoint].value)) * graphSize;

				ImVec2 directionVector = localInfo.myMouseTracking - pointPosition;
				directionVector /= sqrtf(directionVector.x * directionVector.x + directionVector.y * directionVector.y);
				const float angle     = FMath::Clamp(acosf(directionVector.x), -FMath::Pi_Almost_Half, FMath::Pi_Almost_Half);
				const float angleSign = directionVector.y < 0.f ? -1.f : 1.f;

				myPoints[localInfo.mySelectedPoint].tangent = tanf(angle * angleSign);
			}
			else
			{
				myPoints[localInfo.mySelectedPoint].time = FMath::Clamp(FMath::InverseLerp(origin.x, origin.x + graphSize.x,
				                                                                           localInfo.myMouseTracking.x), 0.f, 1.f);
				myPoints[localInfo.mySelectedPoint].value = FMath::Lerp(myMinValue, myMaxValue,
				                                                        FMath::Clamp(FMath::InverseLerp(origin.y + graphSize.y, origin.y,
				                                                                      localInfo.myMouseTracking.y), 0.f, 1.f));

				Sort();
			}
		}

		if (ImGui::IsMouseDoubleClicked(ImGuiMouseButton_Left) && localInfo.mySelectedPoint == -1 && hasHovered == false)
		{
			constexpr float tolerance = 6.f;

			const float mouseT = FMath::InverseLerp(origin.x, origin.x + graphSize.x, ImGui::GetMousePos().x);

			if (mouseT >= 0.f && mouseT <= 1.f)
			{
				const float mouseY = ImGui::GetMousePos().y;

				const float graphValue = Evaluate(mouseT);
				const float graphY = origin.y + FMath::InverseLerp(myMaxValue, myMinValue, Evaluate(mouseT)) *
				                     graphSize.y;

				if (fabsf(mouseY - graphY) < tolerance)
				{
					CurvePoint newPoint;
					newPoint.time    = mouseT;
					newPoint.tangent = 0.f;
					newPoint.value   = FMath::Clamp(graphValue, myMinValue, myMaxValue);

					myPoints.emplace_back(newPoint);

					Sort();

					info |= PropertyEdit::InfoFlag_EditEnded | PropertyEdit::InfoFlag_ChangedValue;
				}
			}
		}
	}

	// Side items
	{
		const float initialMax = myMaxValue;
		const float initialMin = myMinValue;

		ImGui::PushItemWidth(sideItemsWidth);

		ImGui::SetCursorPos(sideCursorOrigin);
		info |= ImGuiUtil::DisplayDragFloat("##MaxValue", &myMaxValue, 0.01f);
		if (ImGui::IsItemHovered())
		{
			ImGui::SetTooltip("Max");
		}

		ImGui::SetCursorPos(ImVec2(sideCursorOrigin.x, ImGui::GetCursorPos().y));
		info |= ImGuiUtil::DisplayDragFloat("##MinValue", &myMinValue, 0.01f);
		if (ImGui::IsItemHovered())
		{
			ImGui::SetTooltip("Min");
		}

		ImGui::PopItemWidth();

		if (myMinValue != initialMin || myMaxValue != initialMax)
		{
			if (myMinValue >= myMaxValue)
			{
				myMinValue = initialMin;
				myMaxValue = initialMax;
			}
			else
			{
				info |= PropertyEdit::InfoFlag_ChangedValue;

				for (auto& point : myPoints)
				{
					point.value = FMath::Remap(point.value, initialMin, initialMax, myMinValue, myMaxValue);
				}
			}
		}

		ImGui::SetCursorPos(ImVec2(sideCursorOrigin.x, ImGui::GetCursorPos().y));
		if (ImGui::Checkbox("Linear", &myIsLinear))
		{
			info |= PropertyEdit::InfoFlag_EditEnded | PropertyEdit::InfoFlag_ChangedValue;
		}
	}

	ImGui::SetCursorPos({0.f, 0.f});
	ImGui::Dummy(totalSize);

	return info;
}

#pragma endregion
```

## Color

Particle color can also be edited for both start values and over time change, both using a gradient. The start value will be a random color in the first gradient, and the second gradient is a overtime multiplier.

![white kitten](../../images/Particle_Color.gif "")

The gradient property was also custom made and has simular code to the curve:

```cpp

namespace Goose
{
	struct ColorKey
	{
		Tga::Color color = {1.f, 1.f, 1.f, 1.f};
		float time       = {0.f};
	};

	class Gradient
	{
	public:
		void Sort();
		Tga::Color Evaluate(float aTime) const;

		std::vector<ColorKey>& AccessColorKeys();
		const std::vector<ColorKey>& AccessColorKeys() const;

		uint32_t DisplayImGuiEdit();

	private:
		uint32_t DisplayGradientEdit();

		void AddGradientToDrawList(const ImVec2& aStart, const ImVec2& aEnd);

		std::vector<ColorKey> myColorKeys = {{}};

		int mySelectedColorKey = -1;
		float myTrackingX      = 0.f;
	};
}

static constexpr ImColor HOVERED_COLOR     = IM_COL32(240, 240, 240, 255);
static constexpr ImColor DEFAULT_COLOR     = IM_COL32(170, 160, 170, 255);
static constexpr ImColor DELETE_COLOR      = IM_COL32(190, 40, 40, 255);
static constexpr ImColor DELETE_HOVER_COLOR = IM_COL32(190, 90, 90, 255);


constexpr const char* POPUP_NAME = "Curve Popup";

static ImU32 TgaColorConvertToU32(const Tga::Color& in)
{
	ImU32 out = static_cast<ImU32>(IM_F32_TO_INT8_SAT(in.r)) << IM_COL32_R_SHIFT;
	out |= static_cast<ImU32>(IM_F32_TO_INT8_SAT(in.g)) << IM_COL32_G_SHIFT;
	out |= static_cast<ImU32>(IM_F32_TO_INT8_SAT(in.b)) << IM_COL32_B_SHIFT;
	out |= static_cast<ImU32>(IM_F32_TO_INT8_SAT(in.a)) << IM_COL32_A_SHIFT;
	return out;
}
void Goose::Gradient::Sort()
{
	for (int i = 0; i < static_cast<int>(myColorKeys.size()); i++)
	{
		int bestIndex    = i;
		float lowestTime = myColorKeys[i].time;
		for (int j = i + 1; j < static_cast<int>(myColorKeys.size()); ++j)
		{
			if (myColorKeys[j].time < lowestTime)
			{
				
				bestIndex  = j;
				lowestTime = myColorKeys[j].time;
			}
		}

		std::swap(myColorKeys[i], myColorKeys[bestIndex]);
		if (bestIndex == mySelectedColorKey)
		{
			mySelectedColorKey = i;
		}
		else if (i == mySelectedColorKey)
		{
			mySelectedColorKey = bestIndex;
		}
	}
}

Tga::Color Goose::Gradient::Evaluate(float aTime) const
{
	if (myColorKeys.size() == 1)
	{
		return myColorKeys.front().color;
	}

	aTime = FMath::Clamp(aTime, 0.f, 1.f);

	Tga::Color previousColor = myColorKeys.front().color;
	float previousTime       = 0.f;
	Tga::Color returnColor;
	

	for (const ColorKey& colorKey : myColorKeys)
	{
		returnColor = colorKey.color;
		if (colorKey.time > aTime)
		{
			if (aTime == 0.f)
			{
				break;
			}

			const float lerpTime = (aTime - previousTime) / (colorKey.time - previousTime);
			returnColor          = FMath::Lerp(previousColor, colorKey.color, lerpTime);
			break;
		}
		previousTime  = colorKey.time;
		previousColor = colorKey.color;
	}

	return returnColor;
}

std::vector<Goose::ColorKey>& Goose::Gradient::AccessColorKeys()
{
	return myColorKeys;
}

const std::vector<Goose::ColorKey>& Goose::Gradient::AccessColorKeys() const
{
	return myColorKeys;
}

uint32_t Goose::Gradient::DisplayImGuiEdit()
{
	uint8_t info = 0;

	

	constexpr float spacingX = 10.f;
	constexpr float graphHeight = 30.f;

	if (globalIsUsingNodeEditor) // This does not display correctly in node editor and is disabled
	{
		ImGui::Text("_Variable_Only_");
		return 0u;
	}

	const ImVec2 origin = ImGui::GetCursorScreenPos();
	const float totalWidth = FMath::Min(ImGui::GetContentRegionAvail().x * 0.7f - spacingX * 2.f, 100.f);
	const ImVec2 graphSize = ImVec2(totalWidth, graphHeight);

	AddGradientToDrawList(origin, origin + graphSize);

	ImGui::Dummy(graphSize);

	const bool hovered = ImGui::IsItemHovered();
	if (hovered && ImGui::IsMouseClicked(ImGuiMouseButton_Left))
	{
		ImGui::OpenPopup(POPUP_NAME);
		ImGui::SetNextWindowPos(ImGui::GetCursorScreenPos());
	}

	if (ImGui::BeginPopup(POPUP_NAME, ImGuiWindowFlags_NoMove))
	{
		info |= DisplayGradientEdit();
		ImGui::EndPopup();
	}

	return info;
}

uint32_t Goose::Gradient::DisplayGradientEdit()
{
	uint32_t info = 0;

	const bool deleteMode = ImGui::GetIO().KeyShift && myColorKeys.size() > 1;

	constexpr float spacingX = 10.f;
	constexpr float spacingY = 10.f;
	constexpr float keySize = 30.f;

	
	const ImVec2 origin        = ImGui::GetWindowPos() + ImVec2{0.f,spacingY };
	constexpr float totalWidth = 300.f;
	
	const float startX         = origin.x + spacingX;
	const float endX           = startX + totalWidth;
	constexpr float gradientShowcaseHeight = 30.f;

	ImDrawList* drawList = ImGui::GetWindowDrawList();
	constexpr ImVec2 totalSize{ totalWidth,gradientShowcaseHeight + keySize + spacingY * 2.f };

	bool hasBeenHovered = false;

	const ImVec2 lineStartPoint = origin + ImVec2(0.f, keySize);
	const ImVec2 lineEndPoint = origin + ImVec2(totalWidth, keySize);

	drawList->AddLine(lineStartPoint, lineEndPoint, IM_COL32(200, 200, 200, 254), 2.f);

	const ImColor noHoverColor = deleteMode ? DELETE_COLOR : DEFAULT_COLOR;
	const ImColor hoverColor = deleteMode ? DELETE_HOVER_COLOR : HOVERED_COLOR;

	const int initialSelected = mySelectedColorKey;

	for (int i = 0; i < static_cast<int>(myColorKeys.size()); ++i)
	{
		ImGui::PushID(i);

		const float xPosition = FMath::Lerp(startX, endX, myColorKeys[i].time);

		ImVec2 drawPos = ImVec2(xPosition, keySize) + ImVec2{0.f,origin.y};

		const Tga::Color initialColor = myColorKeys[i].color;

		ImGui::SetCursorScreenPos(ImVec2(drawPos.x - 12.5f, drawPos.y - 30.f)); // 4
		ImGui::ColorEdit4("##ColorEdit", myColorKeys[i].color.myValues, ImGuiColorEditFlags_NoInputs);

		if (initialColor != myColorKeys[i].color)
		{
			info |= PropertyEdit::InfoFlag_ChangedValue;
		}
		if (ImGui::IsItemDeactivatedAfterEdit())
		{
			info |= PropertyEdit::InfoFlag_EditEnded;
		}

		bool hovered = false;
		if (hasBeenHovered == false)
		{
			hovered = ImGui::IsMouseHoveringRect(drawPos - ImVec2(5.f, 5.f), drawPos + ImVec2(5.f, 5.f));
			hasBeenHovered |= hovered;
		}

		drawList->AddCircleFilled(drawPos, 5.f, hovered ? hoverColor : noHoverColor);

		if (ImGui::IsMouseClicked(ImGuiMouseButton_Left) && hovered)
		{
			if (deleteMode)
			{
				myColorKeys.erase(myColorKeys.begin() + i);
				i--;
				info |= PropertyEdit::InfoFlag_EditEnded;
				info |= PropertyEdit::InfoFlag_ChangedValue;
			}
			else
			{
				mySelectedColorKey = static_cast<int>(i);
				myTrackingX = xPosition + origin.x;
			}

		}

		ImGui::PopID();
	}

	if (hasBeenHovered == false && ImGui::IsMouseDoubleClicked(ImGuiMouseButton_Left))
	{
		if (ImGui::IsMouseHoveringRect(lineStartPoint - ImVec2(2.f, 2.f), lineEndPoint + ImVec2(2.f, 2.f)))
		{
			ColorKey newColorKey;
			newColorKey.time = FMath::Clamp(FMath::InverseLerp(startX, endX, ImGui::GetMousePos().x), 0.f, 1.f);

			myColorKeys.emplace_back(newColorKey);

			info |= PropertyEdit::InfoFlag_EditEnded;
			info |= PropertyEdit::InfoFlag_ChangedValue;

			Sort();
		}
	}

	
	const ImVec2 gradientShowcaseRoot = lineStartPoint + ImVec2(0.f, 10.f);

	AddGradientToDrawList(gradientShowcaseRoot, gradientShowcaseRoot + ImVec2{ totalWidth,gradientShowcaseHeight });

	ImGui::SetCursorScreenPos(origin + ImVec2(0.f, gradientShowcaseHeight + 20.f));

	if (ImGui::IsMouseReleased(ImGuiMouseButton_Left))
	{
		if (mySelectedColorKey != -1)
		{
			info |= PropertyEdit::InfoFlag_EditEnded;
		}
		mySelectedColorKey = -1;
	}

	myTrackingX += ImGui::GetIO().MouseDelta.x;

	if (mySelectedColorKey != -1)
	{
		if (ImGui::GetIO().MouseDelta.x != 0.f)
		{
			info |= PropertyEdit::InfoFlag_ChangedValue;
		}

		myColorKeys[mySelectedColorKey].time = FMath::InverseLerp(startX, endX, myTrackingX - origin.x);
		myColorKeys[mySelectedColorKey].time = FMath::Clamp(myColorKeys[mySelectedColorKey].time, 0.f, 1.f);

		Sort();
	}

	if (initialSelected != mySelectedColorKey)
	{
		info |= PropertyEdit::InfoFlag_ChangedValue;
	}

	ImGui::SetCursorPos({ 0.f, 0.f });
	ImGui::Dummy(totalSize);

	return info;
}

void Goose::Gradient::AddGradientToDrawList(const ImVec2& aStart, const ImVec2& aEnd)
{
	const float gradientShowcaseHeight = aEnd.y - aStart.y;
	float lastX = aStart.x;
	ImColor lastColor = TgaColorConvertToU32(myColorKeys.front().color);

	ImDrawList* drawList = ImGui::GetWindowDrawList();
	for (size_t i = 0; i < myColorKeys.size(); ++i)
	{
		ImColor currentColor = TgaColorConvertToU32(myColorKeys[i].color);
		const float xPosition = FMath::Lerp(aStart.x, aEnd.x, myColorKeys[i].time);

		const ImVec2 subSize = ImVec2(xPosition - lastX, gradientShowcaseHeight);
		const ImVec2 subCorner = ImVec2{ 0.f,aStart.y } + ImVec2(lastX, 0.f);

		drawList->AddRectFilledMultiColor(subCorner, subCorner + subSize, lastColor, currentColor, currentColor, lastColor);

		lastColor = currentColor;
		lastX = xPosition;
	}
	{
		ImColor currentColor = TgaColorConvertToU32(myColorKeys.back().color);
		const float xPosition = aEnd.x;

		const ImVec2 subSize = ImVec2(xPosition - lastX, gradientShowcaseHeight);
		const ImVec2 subCorner = ImVec2{ 0.f,aStart.y } + ImVec2(lastX, 0.f);

		drawList->AddRectFilledMultiColor(subCorner, subCorner + subSize, lastColor, currentColor, currentColor, lastColor);
	}
}

```

## Rendering

In the rendering tab, you can assign material to render with and what type of particle to render. The default type is a **Billboarded** sprite. Another sprite option is **Velocity Sprite**, which will align the sprite's
y-axis to the direction it travels in and then rotate it along the y-axis towards the camera.

There are also different UV options. You can choose to scale the UV and render with random parts.

![white kitten](../../images/Particle_VelocitySprite.png "")

The last particle type is **Mesh**. Having this option on will also enable rotation-options around the x and y axis.

![white kitten](../../images/Particle_Mesh.png "")

## Multiple Emitors

A particle system supports multiple emitors, aLlowing for multiple particle types in one system.

Additionaly, you can add **events**, that executes on particle birth or death/, that can trigger a specifik emitor.

![white kitten](../../images/Particle_OnDeath.gif "")

## Gravity & Noise

This was the last feature I added to the particle editor. Gravity will apply a force on the particle overtime, and you can choose if the gravity direction is down or towards a certain point.

![Particle gravity gif](../../images/Particle_Gravity.gif "Particle gravity gif")

Noise will psuedo-randomly offet the particle depending on its position. Strength controlls how big the offset is, frequency how big the variation is and update time how often the noise is recalculated

![Particle noise gif](../../images/Particle_Noise.gif "Particle noise gif")

