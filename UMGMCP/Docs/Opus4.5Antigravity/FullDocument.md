# UMG MCP Asset Generator System
## 통합 기술 설계 문서 v1.0

---

# 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [시스템 요구사항](#2-시스템-요구사항)
3. [시스템 아키텍처](#3-시스템-아키텍처)
4. [클래스 설계](#4-클래스-설계)
5. [JSON 명세 및 스키마](#5-json-명세-및-스키마)
6. [위젯 트리 빌더](#6-위젯-트리-빌더)
7. [속성 리플렉션 시스템](#7-속성-리플렉션-시스템)
8. [슬롯 시스템](#8-슬롯-시스템)
9. [에셋 관리](#9-에셋-관리)
10. [MCP 연동](#10-mcp-연동)
11. [구현 가이드](#11-구현-가이드)

---

# 문서 정보

| 항목 | 내용 |
|------|------|
| **문서 버전** | 1.0 (통합본) |
| **작성일** | 2024-12-29 |
| **대상 엔진** | Unreal Engine 5.3+ |
| **대상 플랫폼** | Windows (Editor Only) |

---

# 1. 프로젝트 개요

## 1.1 목적

UMG MCP Asset Generator System은 **AI 에이전트가 MCP(Model Context Protocol)를 통해 언리얼 에디터에 접속**하여, JSON 명세를 기반으로 **실제 위젯 블루프린트(.uasset)를 생성, 편집, 저장**할 수 있도록 지원하는 에디터 확장 시스템입니다.

> **핵심 가치**: AI가 UI 레이아웃을 제안하고, 이를 즉시 프로젝트의 Content 폴더에 `.uasset` 형태로 물리적으로 저장하여 UI 프로토타이핑 속도를 혁신합니다.

## 1.2 핵심 차별점

| 구분 | 기존 방식 | UMG MCP 방식 |
|------|----------|--------------|
| **생성 시점** | 런타임(In-Game) | 에디터 타임(Editor-Time) |
| **결과물** | 임시 위젯 객체 | 영구 저장 .uasset 파일 |
| **AI 연동** | 불가능 | MCP 프로토콜 지원 |

## 1.3 기술 스택

### 언리얼 엔진 측

| 구성 요소 | 기술/모듈 |
|----------|----------|
| **기반 클래스** | `UEditorSubsystem` |
| **필수 모듈** | `UnrealEd`, `UMG`, `UMGEditor`, `Slate`, `SlateCore` |
| **에셋 생성** | `UWidgetBlueprintFactory` |
| **컴파일** | `FKismetEditorUtilities::CompileBlueprint` |
| **빌드 조건** | `WITH_EDITOR` 매크로 필수 |

### Python MCP 측

| 구성 요소 | 기술/라이브러리 |
|----------|----------------|
| **프로토콜** | Model Context Protocol (MCP) |
| **데이터 검증** | Pydantic v2 |
| **SDK** | mcp-python-sdk |

## 1.4 지원 위젯 타입

**컨테이너:** CanvasPanel, VerticalBox, HorizontalBox, GridPanel, Overlay, SizeBox, ScaleBox, ScrollBox

**기본:** TextBlock, Image, Button, Border, ProgressBar, Slider, CheckBox, EditableText, Spacer

## 1.5 제약 사항

| 제약 사항 | 설명 | 우회 방안 |
|----------|------|----------|
| **에디터 전용** | 런타임/패키지 빌드에서 동작 불가 | Editor 모듈로 분리 |
| **이벤트 바인딩** | 블루프린트 그래프 노드 생성 복잡 | Base Class 상속 방식 |
| **애니메이션** | UMG 애니메이션 자동 생성 미지원 | 사전 정의된 애니메이션 참조 |

---

# 2. 시스템 요구사항

## 2.1 기능 요구사항

### 위젯 생성 (FR-001 ~ FR-010)

| ID | 요구사항 | 우선순위 |
|----|----------|----------|
| FR-001 | JSON 명세로 CanvasPanel 루트 위젯 블루프린트 생성 | 필수 |
| FR-002 | VerticalBox, HorizontalBox, GridPanel 등 컨테이너 생성 | 필수 |
| FR-003 | Button, TextBlock, Image 등 기본 위젯 생성 | 필수 |
| FR-004 | 중첩 JSON 구조를 재귀적으로 위젯 트리 구성 | 필수 |
| FR-005 | 각 위젯에 고유 이름 부여 및 관리 | 필수 |
| FR-010 | 커스텀 UserWidget 클래스를 부모로 지정 | 필수 |

### 속성 제어 (FR-011 ~ FR-020)

| ID | 요구사항 | 우선순위 |
|----|----------|----------|
| FR-011 | UObject 리플렉션으로 위젯 속성 동적 설정 | 필수 |
| FR-012 | Text, Color, Visibility 등 기본 속성 지원 | 필수 |
| FR-018 | 에셋 참조 속성을 경로로 지정 | 필수 |
| FR-019 | 열거형 속성을 문자열로 지정 | 필수 |
| FR-020 | 구조체 속성을 JSON 객체로 지정 | 필수 |

### 슬롯 제어 (FR-021 ~ FR-030)

| ID | 요구사항 | 우선순위 |
|----|----------|----------|
| FR-021 | 부모 위젯 타입에 따른 슬롯 클래스 자동 감지 | 필수 |
| FR-022 | CanvasPanelSlot의 Anchors, Offsets, Alignment 지원 | 필수 |
| FR-023 | BoxSlot의 Padding, Size, Alignment 지원 | 필수 |
| FR-024 | GridSlot의 Row, Column, Span 지원 | 필수 |

### 에셋 관리 (FR-031 ~ FR-040)

| ID | 요구사항 | 우선순위 |
|----|----------|----------|
| FR-031 | UWidgetBlueprintFactory로 패키지 생성 | 필수 |
| FR-032 | FKismetEditorUtilities::CompileBlueprint로 컴파일 | 필수 |
| FR-033 | UPackage::SavePackage로 .uasset 저장 | 필수 |
| FR-035 | 기존 에셋 덮어쓰기 옵션 제공 | 필수 |

## 2.2 비기능 요구사항

### 성능

| ID | 요구사항 | 목표값 |
|----|----------|--------|
| NFR-001 | 단일 위젯 블루프린트 생성 시간 | < 500ms |
| NFR-002 | 100개 위젯 계층 구조 생성 시간 | < 2s |
| NFR-004 | MCP 응답 시간 (생성+저장) | < 3s |

### 호환성

| ID | 요구사항 | 대상 버전 |
|----|----------|----------|
| NFR-021 | Unreal Engine 버전 | 5.3, 5.4, 5.5 |
| NFR-022 | Python 버전 | 3.10+ |
| NFR-023 | MCP SDK 버전 | 1.0+ |

---

# 3. 시스템 아키텍처

## 3.1 시스템 구성도

```
┌───────────────────────────────────────────────────────────────┐
│                     AI 에이전트 (Claude)                        │
└──────────────────────────┬────────────────────────────────────┘
                           │ MCP Protocol (JSON-RPC)
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                    MCP Server (Python)                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │Tool Handler│  │  Pydantic  │  │ UE Client  │               │
│  └────────────┘  └────────────┘  └────────────┘               │
└──────────────────────────┬────────────────────────────────────┘
                           │ HTTP (JSON)
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   Unreal Editor (C++)                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │           UUMGAssetGeneratorSubsystem                    │  │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐   │  │
│  │  │WidgetTreeBuilder│  │    PropertyReflector        │   │  │
│  │  └─────────────────┘  └─────────────────────────────┘   │  │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐   │  │
│  │  │SlotPropertyHandler│ │   UMGBlueprintFactory      │   │  │
│  │  └─────────────────┘  └─────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              │                                 │
│                              ▼                                 │
│                    ┌─────────────────┐                        │
│                    │  File System    │                        │
│                    │  (.uasset 저장) │                        │
│                    └─────────────────┘                        │
└───────────────────────────────────────────────────────────────┘
```

## 3.2 플러그인 구조

```
UMGMCP/
├── UMGMCP.uplugin
├── Source/
│   ├── UMGMCPRuntime/           # 런타임 모듈 (공유 타입)
│   │   ├── Public/
│   │   │   └── UMGMCPTypes.h
│   │   └── Private/
│   │       └── UMGMCPRuntimeModule.cpp
│   │
│   └── UMGMCPEditor/            # 에디터 모듈 (핵심 기능)
│       ├── Public/
│       │   ├── UMGAssetGeneratorSubsystem.h
│       │   ├── WidgetTreeBuilder.h
│       │   ├── PropertyReflector.h
│       │   ├── SlotPropertyHandler.h
│       │   └── UMGMCPHttpServer.h
│       └── Private/
│           └── *.cpp
│
├── Python/                       # MCP 서버
│   ├── requirements.txt
│   └── umg_mcp_server/
│       ├── server.py
│       ├── tools.py
│       ├── models.py
│       └── ue_client.py
│
└── Docs/                         # 문서
```

## 3.3 HTTP API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/umg/create` | 위젯 블루프린트 생성 |
| POST | `/api/umg/add` | 위젯 추가 |
| PUT | `/api/umg/modify` | 위젯 수정 |
| DELETE | `/api/umg/remove` | 위젯 삭제 |
| POST | `/api/umg/save` | 에셋 저장 |
| GET | `/api/umg/tree` | 위젯 트리 조회 |

---

# 4. 클래스 설계

## 4.1 핵심 클래스

### UUMGAssetGeneratorSubsystem

```cpp
UCLASS()
class UMGMCPEDITOR_API UUMGAssetGeneratorSubsystem : public UEditorSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    static UUMGAssetGeneratorSubsystem* Get();

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult CreateWidgetBlueprint(
        const FString& AssetPath,
        const FString& ParentClass,
        const FString& RootWidgetJson,
        bool bOverwrite = false);

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult AddWidget(
        const FString& AssetPath,
        const FString& ParentWidgetName,
        const FString& WidgetSpecJson);

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult ModifyWidget(
        const FString& AssetPath,
        const FString& WidgetName,
        const FString& PropertiesJson,
        const FString& SlotPropertiesJson);

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult SaveAsset(const FString& AssetPath, bool bCompile = true);

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGWidgetTreeInfo GetWidgetTree(const FString& AssetPath);

private:
    UPROPERTY()
    TObjectPtr<UWidgetTreeBuilder> TreeBuilder;

    UPROPERTY()
    TObjectPtr<UPropertyReflector> PropertyReflector;

    UPROPERTY()
    TObjectPtr<USlotPropertyHandler> SlotHandler;
};
```

### UWidgetTreeBuilder

```cpp
UCLASS()
class UMGMCPEDITOR_API UWidgetTreeBuilder : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(UPropertyReflector* InReflector, USlotPropertyHandler* InSlotHandler);

    UWidget* ParseJsonToWidgetTree(
        const FString& JsonString,
        UWidgetTree* WidgetTree,
        FString& OutError);

    UWidget* CreateWidget(
        const FString& TypeName,
        const FString& WidgetName,
        UWidgetTree* WidgetTree,
        FString& OutError);

    bool AddChildToParent(UPanelWidget* Parent, UWidget* Child);

private:
    UWidget* ProcessWidgetNode(
        const TSharedPtr<FJsonObject>& JsonObject,
        UWidgetTree* WidgetTree,
        UPanelWidget* Parent,
        int32 CurrentDepth,
        FString& OutError);

    UClass* FindWidgetClass(const FString& TypeName);

    TMap<FName, UClass*> WidgetClassCache;
    int32 MaxRecursionDepth = 50;
    TArray<FString> Warnings;
};
```

## 4.2 데이터 구조체

```cpp
USTRUCT(BlueprintType)
struct UMGMCPRUNTIME_API FUMGOperationResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly) bool bSuccess = false;
    UPROPERTY(BlueprintReadOnly) int32 ErrorCode = 0;
    UPROPERTY(BlueprintReadOnly) FString ErrorMessage;
    UPROPERTY(BlueprintReadOnly) FString AssetPath;
    UPROPERTY(BlueprintReadOnly) int32 WidgetCount = 0;
    UPROPERTY(BlueprintReadOnly) TArray<FString> Warnings;

    static FUMGOperationResult Success(const FString& Path, int32 Count);
    static FUMGOperationResult Failure(int32 Code, const FString& Message);
};

USTRUCT(BlueprintType)
struct UMGMCPRUNTIME_API FUMGWidgetInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly) FString Name;
    UPROPERTY(BlueprintReadOnly) FString Type;
    UPROPERTY(BlueprintReadOnly) FString ParentName;
    UPROPERTY(BlueprintReadOnly) TArray<FString> ChildrenNames;
};
```

## 4.3 에러 코드

```cpp
namespace UMGMCPError
{
    // 입력 검증 (1000-1099)
    constexpr int32 InvalidJson = 1001;
    constexpr int32 MissingRequiredField = 1002;

    // 위젯 생성 (2000-2099)
    constexpr int32 UnknownWidgetType = 2001;
    constexpr int32 MaxDepthExceeded = 2004;

    // 속성 적용 (3000-3099)
    constexpr int32 PropertyNotFound = 3001;

    // 에셋 관리 (5000-5099)
    constexpr int32 InvalidAssetPath = 5001;
    constexpr int32 CompileFailed = 5004;
    constexpr int32 SaveFailed = 5005;
}
```

---

# 5. JSON 명세 및 스키마

## 5.1 기본 구조

```json
{
    "Type": "WidgetTypeName",
    "Name": "OptionalWidgetName",
    "Properties": {
        "PropertyName": "Value"
    },
    "SlotProperties": {
        "SlotPropertyName": "Value"
    },
    "Children": [
        { "Type": "ChildWidget", ... }
    ]
}
```

## 5.2 Properties 예시

### TextBlock

```json
{
    "Text": "표시할 텍스트",
    "ColorAndOpacity": {"R": 1.0, "G": 1.0, "B": 1.0, "A": 1.0},
    "Font": {
        "FontObject": "/Game/Fonts/MyFont",
        "Size": 24
    },
    "Justification": "Center"
}
```

### Image

```json
{
    "Brush": {
        "ResourceObject": "/Game/Textures/MyTexture",
        "ImageSize": {"X": 64, "Y": 64},
        "DrawAs": "Image",
        "TintColor": {"R": 1, "G": 1, "B": 1, "A": 1}
    }
}
```

## 5.3 SlotProperties 예시

### CanvasPanelSlot

```json
{
    "AnchorPreset": "Center",
    "Anchors": {
        "Minimum": {"X": 0.5, "Y": 0.5},
        "Maximum": {"X": 0.5, "Y": 0.5}
    },
    "Offsets": {"Left": -100, "Top": -50, "Right": 100, "Bottom": 50},
    "Alignment": {"X": 0.5, "Y": 0.5}
}
```

### VerticalBoxSlot

```json
{
    "Padding": {"Left": 0, "Top": 10, "Right": 0, "Bottom": 10},
    "Size": {"SizeRule": "Fill", "Value": 1.0},
    "HorizontalAlignment": "Center"
}
```

## 5.4 앵커 프리셋

| 프리셋 | Minimum | Maximum |
|--------|---------|---------|
| TopLeft | (0, 0) | (0, 0) |
| Center | (0.5, 0.5) | (0.5, 0.5) |
| BottomRight | (1, 1) | (1, 1) |
| StretchAll | (0, 0) | (1, 1) |

## 5.5 전체 예제

```json
{
    "Type": "CanvasPanel",
    "Name": "RootCanvas",
    "Children": [
        {
            "Type": "Image",
            "Name": "Background",
            "Properties": {
                "Brush": {"ResourceObject": "/Game/UI/Textures/BG"}
            },
            "SlotProperties": {
                "AnchorPreset": "StretchAll"
            }
        },
        {
            "Type": "VerticalBox",
            "Name": "MenuContainer",
            "SlotProperties": {
                "AnchorPreset": "Center",
                "Alignment": {"X": 0.5, "Y": 0.5}
            },
            "Children": [
                {
                    "Type": "TextBlock",
                    "Name": "Title",
                    "Properties": {
                        "Text": "GAME TITLE",
                        "Font": {"Size": 48}
                    },
                    "SlotProperties": {
                        "Padding": {"Bottom": 40}
                    }
                },
                {
                    "Type": "Button",
                    "Name": "PlayButton",
                    "Children": [
                        {
                            "Type": "TextBlock",
                            "Name": "PlayText",
                            "Properties": {"Text": "Play"}
                        }
                    ]
                }
            ]
        }
    ]
}
```

---

# 6. 위젯 트리 빌더

## 6.1 생성 흐름

1. JSON 입력 → FJsonObject 파싱
2. Type 필드 추출 → 위젯 클래스 조회
3. UWidget 인스턴스 생성 → 이름 설정
4. Properties 적용 (PropertyReflector)
5. 부모에 추가 (있는 경우)
6. SlotProperties 적용 (SlotPropertyHandler)
7. Children 재귀 처리
8. 위젯 반환

## 6.2 핵심 알고리즘

```cpp
UWidget* UWidgetTreeBuilder::ProcessWidgetNode(
    const TSharedPtr<FJsonObject>& JsonObject,
    UWidgetTree* WidgetTree,
    UPanelWidget* Parent,
    int32 CurrentDepth,
    FString& OutError)
{
    // 깊이 제한 검사
    if (CurrentDepth > MaxRecursionDepth)
    {
        OutError = TEXT("Maximum recursion depth exceeded");
        return nullptr;
    }

    // Type 추출
    FString WidgetType;
    if (!JsonObject->TryGetStringField(TEXT("Type"), WidgetType))
    {
        OutError = TEXT("Missing required field: Type");
        return nullptr;
    }

    // 위젯 생성
    FString WidgetName = GetOrGenerateName(JsonObject, WidgetType);
    UWidget* NewWidget = CreateWidget(WidgetType, WidgetName, WidgetTree, OutError);
    if (!NewWidget) return nullptr;

    // Properties 적용
    ApplyProperties(NewWidget, JsonObject);

    // 부모에 추가
    if (Parent)
    {
        Parent->AddChild(NewWidget);
        ApplySlotProperties(NewWidget, JsonObject);
    }

    // Children 재귀 처리
    ProcessChildren(NewWidget, JsonObject, WidgetTree, CurrentDepth);

    return NewWidget;
}
```

## 6.3 위젯 클래스 매핑

```cpp
static TMap<FName, FName> TypeToClassName = {
    {TEXT("CanvasPanel"), TEXT("CanvasPanel")},
    {TEXT("VerticalBox"), TEXT("VerticalBox")},
    {TEXT("HorizontalBox"), TEXT("HorizontalBox")},
    {TEXT("TextBlock"), TEXT("TextBlock")},
    {TEXT("Image"), TEXT("Image")},
    {TEXT("Button"), TEXT("Button")},
    // ...
};
```

---

# 7. 속성 리플렉션 시스템

## 7.1 지원 타입

| 타입 | C++ | JSON 형식 |
|------|-----|----------|
| Bool | bool | boolean |
| Int | int32 | number |
| Float | float | number |
| String | FString | string |
| Text | FText | string |
| Enum | UEnum | string |
| Struct | UScriptStruct | object |
| Object | UObject* | string (경로) |

## 7.2 구조체 처리

| 구조체 | 필드 |
|--------|------|
| FLinearColor | R, G, B, A |
| FVector2D | X, Y |
| FMargin | Left, Top, Right, Bottom |
| FAnchors | Minimum, Maximum |

## 7.3 속성 적용 코드

```cpp
bool UPropertyReflector::SetPropertyValue(
    UObject* Target,
    const FString& PropertyName,
    const TSharedPtr<FJsonValue>& JsonValue,
    FString& OutError)
{
    FProperty* Property = FindProperty(Target->GetClass(), PropertyName);
    if (!Property)
    {
        OutError = TEXT("Property not found");
        return false;
    }

    void* ValuePtr = Property->ContainerPtrToValuePtr<void>(Target);

    if (FBoolProperty* BoolProp = CastField<FBoolProperty>(Property))
    {
        bool Value;
        if (JsonValue->TryGetBool(Value))
        {
            BoolProp->SetPropertyValue(ValuePtr, Value);
            return true;
        }
    }
    else if (FTextProperty* TextProp = CastField<FTextProperty>(Property))
    {
        FString Value;
        if (JsonValue->TryGetString(Value))
        {
            TextProp->SetPropertyValue(ValuePtr, FText::FromString(Value));
            return true;
        }
    }
    // ... 다른 타입 처리

    return false;
}
```

---

# 8. 슬롯 시스템

## 8.1 부모-슬롯 매핑

| 부모 위젯 | 슬롯 클래스 | 주요 속성 |
|----------|------------|----------|
| CanvasPanel | UCanvasPanelSlot | Anchors, Offsets, Alignment |
| VerticalBox | UVerticalBoxSlot | Padding, Size, HAlign |
| HorizontalBox | UHorizontalBoxSlot | Padding, Size, VAlign |
| GridPanel | UGridSlot | Row, Column, Span |
| Overlay | UOverlaySlot | Padding, Alignment |

## 8.2 슬롯 타입 감지

```cpp
EUMGSlotType USlotPropertyHandler::DetectSlotType(UPanelWidget* ParentWidget)
{
    if (Cast<UCanvasPanel>(ParentWidget))
        return EUMGSlotType::CanvasPanelSlot;
    if (Cast<UVerticalBox>(ParentWidget))
        return EUMGSlotType::VerticalBoxSlot;
    if (Cast<UHorizontalBox>(ParentWidget))
        return EUMGSlotType::HorizontalBoxSlot;
    // ...
    return EUMGSlotType::Unknown;
}
```

## 8.3 CanvasSlot 적용

```cpp
bool ApplyCanvasSlotProperties(UCanvasPanelSlot* Slot, const TSharedPtr<FJsonObject>& Json)
{
    // 앵커 프리셋
    FString PresetName;
    if (Json->TryGetStringField(TEXT("AnchorPreset"), PresetName))
    {
        FAnchors Anchors = GetAnchorsForPreset(PresetName);
        Slot->SetAnchors(Anchors);
    }

    // Offsets
    if (const TSharedPtr<FJsonObject>* OffsetsObj; Json->TryGetObjectField(TEXT("Offsets"), OffsetsObj))
    {
        FMargin Offsets;
        Offsets.Left = (*OffsetsObj)->GetNumberField(TEXT("Left"));
        Offsets.Top = (*OffsetsObj)->GetNumberField(TEXT("Top"));
        Offsets.Right = (*OffsetsObj)->GetNumberField(TEXT("Right"));
        Offsets.Bottom = (*OffsetsObj)->GetNumberField(TEXT("Bottom"));
        Slot->SetOffsets(Offsets);
    }

    // Alignment
    if (const TSharedPtr<FJsonObject>* AlignObj; Json->TryGetObjectField(TEXT("Alignment"), AlignObj))
    {
        FVector2D Alignment;
        Alignment.X = (*AlignObj)->GetNumberField(TEXT("X"));
        Alignment.Y = (*AlignObj)->GetNumberField(TEXT("Y"));
        Slot->SetAlignment(Alignment);
    }

    return true;
}
```

---

# 9. 에셋 관리

## 9.1 블루프린트 생성

```cpp
UWidgetBlueprint* FUMGBlueprintFactory::CreateWidgetBlueprint(
    const FString& AssetPath,
    UClass* ParentClass,
    bool bOverwrite,
    FString& OutError)
{
    // 패키지 생성
    UPackage* Package = CreatePackage(*AssetPath);
    Package->FullyLoad();

    // 부모 클래스 결정
    if (!ParentClass)
        ParentClass = UUserWidget::StaticClass();

    // 위젯 블루프린트 팩토리 사용
    UWidgetBlueprintFactory* Factory = NewObject<UWidgetBlueprintFactory>();
    Factory->ParentClass = ParentClass;

    FString AssetName = FPaths::GetBaseFilename(AssetPath);
    UWidgetBlueprint* Blueprint = Cast<UWidgetBlueprint>(
        Factory->FactoryCreateNew(
            UWidgetBlueprint::StaticClass(),
            Package,
            FName(*AssetName),
            RF_Public | RF_Standalone,
            nullptr,
            GWarn
        )
    );

    Blueprint->WidgetTree = NewObject<UWidgetTree>(Blueprint, TEXT("WidgetTree"));
    return Blueprint;
}
```

## 9.2 컴파일

```cpp
bool FUMGBlueprintFactory::CompileBlueprint(UWidgetBlueprint* Blueprint, FString& OutError)
{
    FCompilerResultsLog Results;
    FKismetEditorUtilities::CompileBlueprint(
        Blueprint,
        EBlueprintCompileOptions::SkipGarbageCollection,
        &Results
    );

    if (Results.NumErrors > 0)
    {
        OutError = FString::Printf(TEXT("Compile failed with %d errors"), Results.NumErrors);
        return false;
    }
    return true;
}
```

## 9.3 저장

```cpp
bool FUMGBlueprintFactory::SaveBlueprint(UWidgetBlueprint* Blueprint, FString& OutError)
{
    UPackage* Package = Blueprint->GetOutermost();
    Package->MarkPackageDirty();

    FString PackageFilename = FPackageName::LongPackageNameToFilename(
        Package->GetName(),
        FPackageName::GetAssetPackageExtension()
    );

    FSavePackageArgs SaveArgs;
    SaveArgs.TopLevelFlags = RF_Public | RF_Standalone;

    FSavePackageResultStruct Result = UPackage::Save(Package, Blueprint, *PackageFilename, SaveArgs);

    if (Result.Result != ESavePackageResult::Success)
    {
        OutError = TEXT("Failed to save package");
        return false;
    }

    FAssetRegistryModule::AssetCreated(Blueprint);
    return true;
}
```

---

# 10. MCP 연동

## 10.1 Pydantic 모델

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any

class WidgetSpec(BaseModel):
    Type: str
    Name: Optional[str] = None
    Properties: Optional[Dict[str, Any]] = None
    SlotProperties: Optional[Dict[str, Any]] = None
    Children: Optional[List["WidgetSpec"]] = None

class CreateWidgetRequest(BaseModel):
    asset_path: str
    parent_class: Optional[str] = None
    root_widget: WidgetSpec
    overwrite: bool = False

class OperationResult(BaseModel):
    success: bool
    asset_path: Optional[str] = None
    widget_count: int = 0
    error_code: Optional[int] = None
    error_message: Optional[str] = None
    warnings: List[str] = []
```

## 10.2 MCP 도구

```python
@server.tool()
async def umg_create_asset(
    asset_path: str,
    root_widget: dict,
    parent_class: str = None,
    overwrite: bool = False
) -> list[TextContent]:
    """새로운 위젯 블루프린트를 생성합니다."""
    widget_spec = WidgetSpec(**root_widget)
    request = CreateWidgetRequest(
        asset_path=asset_path,
        parent_class=parent_class,
        root_widget=widget_spec,
        overwrite=overwrite
    )
    result = ue_client.create_widget_blueprint(request)
    
    if result.success:
        return [TextContent(text=f"✅ Created: {result.asset_path}")]
    else:
        return [TextContent(text=f"❌ Failed: {result.error_message}")]

@server.tool()
async def umg_modify_widget(
    asset_path: str,
    widget_name: str,
    properties: dict = None,
    slot_properties: dict = None
) -> list[TextContent]:
    """위젯 속성을 수정합니다."""
    # ...

@server.tool()
async def umg_save_asset(asset_path: str, compile: bool = True) -> list[TextContent]:
    """위젯 블루프린트를 저장합니다."""
    # ...
```

## 10.3 Claude Desktop 설정

```json
{
    "mcpServers": {
        "umg-mcp": {
            "command": "python",
            "args": ["-m", "umg_mcp_server.server"],
            "cwd": "D:/GitHub/ai_generated/UMGMCP/Python",
            "env": {
                "UMG_MCP_UE_ENDPOINT": "http://localhost:8080"
            }
        }
    }
}
```

---

# 11. 구현 가이드

## 11.1 플러그인 설정

### UMGMCP.uplugin

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "UMG MCP Asset Generator",
    "Description": "AI-driven UMG widget blueprint generation via MCP",
    "Category": "Editor",
    "Modules": [
        {
            "Name": "UMGMCPRuntime",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        },
        {
            "Name": "UMGMCPEditor",
            "Type": "Editor",
            "LoadingPhase": "Default"
        }
    ]
}
```

### Build.cs

```csharp
// UMGMCPEditor.Build.cs
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core", "CoreUObject", "Engine", "UMG", "Slate", "SlateCore", "UMGMCPRuntime"
});

PrivateDependencyModuleNames.AddRange(new string[]
{
    "UnrealEd", "UMGEditor", "Kismet", "BlueprintGraph",
    "HTTP", "Json", "JsonUtilities", "AssetTools",
    "ContentBrowser", "EditorSubsystem"
});
```

## 11.2 사용 예제

### AI 에이전트에서

```python
await umg_create_asset(
    asset_path="/Game/UI/MainMenu",
    root_widget={
        "Type": "CanvasPanel",
        "Children": [
            {
                "Type": "VerticalBox",
                "SlotProperties": {"AnchorPreset": "Center"},
                "Children": [
                    {"Type": "TextBlock", "Properties": {"Text": "Game Title"}},
                    {"Type": "Button", "Children": [
                        {"Type": "TextBlock", "Properties": {"Text": "Play"}}
                    ]}
                ]
            }
        ]
    },
    overwrite=True
)
```

### C++ 직접 호출

```cpp
UUMGAssetGeneratorSubsystem* Subsystem = UUMGAssetGeneratorSubsystem::Get();

FString Json = R"({
    "Type": "CanvasPanel",
    "Children": [{"Type": "TextBlock", "Properties": {"Text": "Hello"}}]
})";

FUMGOperationResult Result = Subsystem->CreateWidgetBlueprint(
    TEXT("/Game/UI/TestWidget"),
    TEXT(""),
    Json,
    true
);

if (Result.bSuccess)
{
    UE_LOG(LogTemp, Log, TEXT("Created with %d widgets"), Result.WidgetCount);
}
```

## 11.3 문제 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| 에셋 저장 실패 | 경로 오류 | /Game/ 접두사 확인 |
| 위젯 미표시 | 컴파일 누락 | compile=true로 저장 |
| 속성 미적용 | 속성명 오타 | 경고 로그 확인 |
| HTTP 연결 실패 | 포트 충돌 | 다른 포트 사용 |

---

# 부록: 참고 자료

- [Unreal Engine UMG Documentation](https://docs.unrealengine.com/5.3/en-US/umg-ui-designer-for-unreal-engine/)
- [Model Context Protocol Specification](https://github.com/anthropics/model-context-protocol)
- [UWidgetBlueprint API Reference](https://docs.unrealengine.com/5.3/en-US/API/Editor/UMGEditor/UWidgetBlueprint/)

---

**문서 버전**: 1.0 (통합본)  
**최종 수정일**: 2024-12-29
