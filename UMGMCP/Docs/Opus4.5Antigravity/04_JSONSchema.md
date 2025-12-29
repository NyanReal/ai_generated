# 04. JSON 명세 및 스키마 정의
## UMG MCP Asset Generator System

---

## 1. JSON 스키마 개요

AI 에이전트가 생성하는 UI 레이아웃 명세의 JSON 스키마를 정의합니다.

### 1.1 최상위 구조

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "UMG Widget Specification",
    "type": "object",
    "required": ["Type"],
    "properties": {
        "Type": {
            "type": "string",
            "description": "위젯 타입명"
        },
        "Name": {
            "type": "string",
            "description": "위젯 고유 이름 (선택, 미지정 시 자동 생성)"
        },
        "Properties": {
            "type": "object",
            "description": "위젯 속성"
        },
        "SlotProperties": {
            "type": "object",
            "description": "부모 슬롯 속성"
        },
        "Children": {
            "type": "array",
            "description": "자식 위젯 배열",
            "items": { "$ref": "#" }
        }
    }
}
```

---

## 2. 지원 위젯 타입

### 2.1 컨테이너 위젯

| 타입명 | 클래스 | 설명 | 자식 허용 |
|--------|--------|------|----------|
| `CanvasPanel` | UCanvasPanel | 절대 좌표 배치 | O |
| `VerticalBox` | UVerticalBox | 수직 정렬 | O |
| `HorizontalBox` | UHorizontalBox | 수평 정렬 | O |
| `GridPanel` | UGridPanel | 그리드 배치 | O |
| `Overlay` | UOverlay | 오버레이 스택 | O |
| `SizeBox` | USizeBox | 크기 제약 | O (1개) |
| `ScaleBox` | UScaleBox | 스케일링 | O (1개) |
| `ScrollBox` | UScrollBox | 스크롤 영역 | O |
| `WidgetSwitcher` | UWidgetSwitcher | 위젯 전환 | O |

### 2.2 기본 위젯

| 타입명 | 클래스 | 설명 | 자식 허용 |
|--------|--------|------|----------|
| `TextBlock` | UTextBlock | 텍스트 표시 | X |
| `Image` | UImage | 이미지 표시 | X |
| `Button` | UButton | 버튼 | O (1개) |
| `Border` | UBorder | 테두리/배경 | O (1개) |
| `ProgressBar` | UProgressBar | 진행률 | X |
| `Slider` | USlider | 슬라이더 | X |
| `CheckBox` | UCheckBox | 체크박스 | X |
| `EditableText` | UEditableText | 편집 텍스트 | X |
| `Spacer` | USpacer | 공간 확보 | X |
| `Throbber` | UThrobber | 로딩 인디케이터 | X |

---

## 3. Properties 스키마

### 3.1 공통 속성

```json
{
    "Visibility": "Visible | Hidden | Collapsed | HitTestInvisible | SelfHitTestInvisible",
    "RenderOpacity": 1.0,
    "IsEnabled": true,
    "ToolTipText": "툴팁 텍스트",
    "Cursor": "Default | Hand | TextEditBeam | ...",
    "RenderTransform": {
        "Translation": {"X": 0, "Y": 0},
        "Scale": {"X": 1, "Y": 1},
        "Shear": {"X": 0, "Y": 0},
        "Angle": 0
    },
    "RenderTransformPivot": {"X": 0.5, "Y": 0.5}
}
```

### 3.2 TextBlock 속성

```json
{
    "Text": "표시할 텍스트",
    "ColorAndOpacity": {"R": 1.0, "G": 1.0, "B": 1.0, "A": 1.0},
    "Font": {
        "FontObject": "/Game/Fonts/MyFont",
        "TypefaceFontName": "Regular",
        "Size": 24,
        "OutlineSettings": {
            "OutlineSize": 0,
            "OutlineColor": {"R": 0, "G": 0, "B": 0, "A": 1}
        }
    },
    "Justification": "Left | Center | Right",
    "AutoWrapText": false,
    "WrapTextAt": 0,
    "TextTransformPolicy": "None | ToLower | ToUpper",
    "StrikeBrush": { ... },
    "ShadowOffset": {"X": 0, "Y": 0},
    "ShadowColorAndOpacity": {"R": 0, "G": 0, "B": 0, "A": 0.5}
}
```

### 3.3 Image 속성

```json
{
    "Brush": {
        "ImageSize": {"X": 64, "Y": 64},
        "DrawAs": "Image | Border | Box | RoundedBox",
        "Tiling": "NoTile | Horizontal | Vertical | Both",
        "TintColor": {"R": 1, "G": 1, "B": 1, "A": 1},
        "ResourceObject": "/Game/Textures/MyTexture"
    },
    "ColorAndOpacity": {"R": 1, "G": 1, "B": 1, "A": 1},
    "FlipForRightToLeftFlowDirection": false
}
```

### 3.4 Button 속성

```json
{
    "WidgetStyle": {
        "Normal": { "TintColor": {...} },
        "Hovered": { "TintColor": {...} },
        "Pressed": { "TintColor": {...} },
        "Disabled": { "TintColor": {...} }
    },
    "ColorAndOpacity": {"R": 1, "G": 1, "B": 1, "A": 1},
    "BackgroundColor": {"R": 1, "G": 1, "B": 1, "A": 1},
    "IsFocusable": true,
    "ClickMethod": "DownAndUp | MouseDown | MouseUp | PreciseClick"
}
```

### 3.5 ProgressBar 속성

```json
{
    "Percent": 0.5,
    "FillType": "LeftToRight | RightToLeft | TopToBottom | BottomToTop | FillFromCenter",
    "BarFillType": "HorizontalFill | VerticalFill",
    "FillColorAndOpacity": {"R": 0.2, "G": 0.8, "B": 0.2, "A": 1},
    "BorderPadding": {"Left": 0, "Top": 0, "Right": 0, "Bottom": 0}
}
```

---

## 4. SlotProperties 스키마

### 4.1 CanvasPanelSlot

```json
{
    "SlotProperties": {
        "AnchorPreset": "TopLeft | Center | StretchAll | ...",
        "Anchors": {
            "Minimum": {"X": 0, "Y": 0},
            "Maximum": {"X": 0, "Y": 0}
        },
        "Offsets": {
            "Left": 0,
            "Top": 0,
            "Right": 100,
            "Bottom": 50
        },
        "Alignment": {"X": 0, "Y": 0},
        "bAutoSize": false,
        "ZOrder": 0
    }
}
```

### 4.2 VerticalBoxSlot / HorizontalBoxSlot

```json
{
    "SlotProperties": {
        "Padding": {"Left": 0, "Top": 0, "Right": 0, "Bottom": 0},
        "Size": {
            "SizeRule": "Auto | Fill",
            "Value": 1.0
        },
        "HorizontalAlignment": "Fill | Left | Center | Right",
        "VerticalAlignment": "Fill | Top | Center | Bottom"
    }
}
```

### 4.3 GridSlot

```json
{
    "SlotProperties": {
        "Row": 0,
        "Column": 0,
        "RowSpan": 1,
        "ColumnSpan": 1,
        "Layer": 0,
        "Nudge": {"X": 0, "Y": 0},
        "HorizontalAlignment": "Fill | Left | Center | Right",
        "VerticalAlignment": "Fill | Top | Center | Bottom"
    }
}
```

### 4.4 OverlaySlot

```json
{
    "SlotProperties": {
        "Padding": {"Left": 0, "Top": 0, "Right": 0, "Bottom": 0},
        "HorizontalAlignment": "Fill | Left | Center | Right",
        "VerticalAlignment": "Fill | Top | Center | Bottom"
    }
}
```

---

## 5. 앵커 프리셋

| 프리셋 | Minimum | Maximum | 설명 |
|--------|---------|---------|------|
| `TopLeft` | (0, 0) | (0, 0) | 좌상단 고정 |
| `TopCenter` | (0.5, 0) | (0.5, 0) | 상단 중앙 |
| `TopRight` | (1, 0) | (1, 0) | 우상단 고정 |
| `CenterLeft` | (0, 0.5) | (0, 0.5) | 좌측 중앙 |
| `Center` | (0.5, 0.5) | (0.5, 0.5) | 중앙 |
| `CenterRight` | (1, 0.5) | (1, 0.5) | 우측 중앙 |
| `BottomLeft` | (0, 1) | (0, 1) | 좌하단 고정 |
| `BottomCenter` | (0.5, 1) | (0.5, 1) | 하단 중앙 |
| `BottomRight` | (1, 1) | (1, 1) | 우하단 고정 |
| `StretchHorizontal` | (0, 0.5) | (1, 0.5) | 가로 스트레치 |
| `StretchVertical` | (0.5, 0) | (0.5, 1) | 세로 스트레치 |
| `StretchAll` | (0, 0) | (1, 1) | 전체 스트레치 |

---

## 6. 전체 예제

### 6.1 메인 메뉴 UI

```json
{
    "Type": "CanvasPanel",
    "Name": "RootCanvas",
    "Children": [
        {
            "Type": "Image",
            "Name": "BackgroundImage",
            "Properties": {
                "Brush": {
                    "ResourceObject": "/Game/UI/Textures/MenuBG",
                    "DrawAs": "Image"
                }
            },
            "SlotProperties": {
                "AnchorPreset": "StretchAll",
                "Offsets": {"Left": 0, "Top": 0, "Right": 0, "Bottom": 0}
            }
        },
        {
            "Type": "VerticalBox",
            "Name": "MenuContainer",
            "SlotProperties": {
                "AnchorPreset": "Center",
                "Alignment": {"X": 0.5, "Y": 0.5},
                "Offsets": {"Left": -150, "Top": -100, "Right": 150, "Bottom": 100}
            },
            "Children": [
                {
                    "Type": "TextBlock",
                    "Name": "TitleText",
                    "Properties": {
                        "Text": "GAME TITLE",
                        "Font": {"Size": 48},
                        "Justification": "Center"
                    },
                    "SlotProperties": {
                        "Padding": {"Bottom": 40},
                        "HorizontalAlignment": "Center"
                    }
                },
                {
                    "Type": "Button",
                    "Name": "PlayButton",
                    "SlotProperties": {
                        "Padding": {"Bottom": 10},
                        "HorizontalAlignment": "Fill"
                    },
                    "Children": [
                        {
                            "Type": "TextBlock",
                            "Name": "PlayButtonText",
                            "Properties": {
                                "Text": "Play",
                                "Justification": "Center"
                            }
                        }
                    ]
                }
            ]
        }
    ]
}
```

---

## 7. 이벤트 처리 전략

블루프린트 그래프 노드 자동 생성은 복잡하므로, 다음 우회 전략을 사용합니다:

### 7.1 Base Class 상속 방식

```json
{
    "asset_path": "/Game/UI/MyMenu",
    "parent_class": "/Game/UI/BP_MenuWidgetBase",
    "root_widget": { ... }
}
```

### 7.2 이벤트 핸들러 명명 규칙

위젯 이름 기반으로 Base Class에서 이벤트 함수를 미리 정의:

```cpp
// BP_MenuWidgetBase.h
UFUNCTION(BlueprintImplementableEvent)
void OnPlayButtonClicked();

UFUNCTION(BlueprintImplementableEvent)
void OnOptionsButtonClicked();
```

### 7.3 Python Pydantic 모델

```python
from pydantic import BaseModel
from typing import Optional, List, Dict, Any

class Vector2D(BaseModel):
    X: float = 0.0
    Y: float = 0.0

class LinearColor(BaseModel):
    R: float = 1.0
    G: float = 1.0
    B: float = 1.0
    A: float = 1.0

class Margin(BaseModel):
    Left: float = 0.0
    Top: float = 0.0
    Right: float = 0.0
    Bottom: float = 0.0

class Anchors(BaseModel):
    Minimum: Vector2D = Vector2D()
    Maximum: Vector2D = Vector2D()

class CanvasSlotProperties(BaseModel):
    AnchorPreset: Optional[str] = None
    Anchors: Optional[Anchors] = None
    Offsets: Optional[Margin] = None
    Alignment: Optional[Vector2D] = None
    bAutoSize: bool = False
    ZOrder: int = 0

class BoxSlotProperties(BaseModel):
    Padding: Optional[Margin] = None
    HorizontalAlignment: str = "Fill"
    VerticalAlignment: str = "Fill"

class WidgetSpec(BaseModel):
    Type: str
    Name: Optional[str] = None
    Properties: Optional[Dict[str, Any]] = None
    SlotProperties: Optional[Dict[str, Any]] = None
    Children: Optional[List["WidgetSpec"]] = None

WidgetSpec.model_rebuild()
```
