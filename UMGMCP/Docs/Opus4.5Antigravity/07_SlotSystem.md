# 07. 슬롯 시스템 처리
## UMG MCP Asset Generator System

---

## 1. 개요

UMG에서 자식 위젯의 레이아웃은 부모 패널이 제공하는 **Slot**에 의해 결정됩니다. `USlotPropertyHandler`는 부모 타입에 따른 적절한 슬롯 속성을 자동으로 적용합니다.

### 1.1 부모-슬롯 매핑

| 부모 위젯 | 슬롯 클래스 | 주요 속성 |
|----------|------------|----------|
| CanvasPanel | UCanvasPanelSlot | Anchors, Offsets, Alignment, ZOrder |
| VerticalBox | UVerticalBoxSlot | Padding, Size, HorizontalAlignment |
| HorizontalBox | UHorizontalBoxSlot | Padding, Size, VerticalAlignment |
| GridPanel | UGridSlot | Row, Column, RowSpan, ColumnSpan |
| Overlay | UOverlaySlot | Padding, HAlign, VAlign |
| UniformGridPanel | UUniformGridSlot | Row, Column |
| SizeBox | USizeBoxSlot | (단일 자식) |
| ScrollBox | UScrollBoxSlot | Padding |

---

## 2. 슬롯 타입 감지

```cpp
EUMGSlotType USlotPropertyHandler::DetectSlotType(UPanelWidget* ParentWidget)
{
    if (!ParentWidget)
    {
        return EUMGSlotType::Unknown;
    }

    if (Cast<UCanvasPanel>(ParentWidget))
        return EUMGSlotType::CanvasPanelSlot;
    
    if (Cast<UVerticalBox>(ParentWidget))
        return EUMGSlotType::VerticalBoxSlot;
    
    if (Cast<UHorizontalBox>(ParentWidget))
        return EUMGSlotType::HorizontalBoxSlot;
    
    if (Cast<UGridPanel>(ParentWidget))
        return EUMGSlotType::GridSlot;
    
    if (Cast<UOverlay>(ParentWidget))
        return EUMGSlotType::OverlaySlot;
    
    if (Cast<UUniformGridPanel>(ParentWidget))
        return EUMGSlotType::UniformGridSlot;
    
    if (Cast<USizeBox>(ParentWidget))
        return EUMGSlotType::SizeBoxSlot;
    
    if (Cast<UScrollBox>(ParentWidget))
        return EUMGSlotType::ScrollBoxSlot;

    return EUMGSlotType::Unknown;
}
```

---

## 3. 슬롯 속성 적용

### 3.1 메인 함수

```cpp
bool USlotPropertyHandler::ApplySlotProperties(
    UWidget* ChildWidget,
    const TSharedPtr<FJsonObject>& SlotPropertiesJson,
    TArray<FString>& OutWarnings)
{
    if (!ChildWidget || !SlotPropertiesJson.IsValid())
    {
        return false;
    }

    UPanelSlot* Slot = ChildWidget->Slot;
    if (!Slot)
    {
        OutWarnings.Add(TEXT("Widget has no slot (not added to parent yet)"));
        return false;
    }

    // 앵커 프리셋 처리
    FString PresetName;
    if (SlotPropertiesJson->TryGetStringField(TEXT("AnchorPreset"), PresetName))
    {
        EUMGAnchorPreset Preset = ParseAnchorPreset(PresetName);
        if (Preset != EUMGAnchorPreset::Custom)
        {
            ApplyAnchorPreset(ChildWidget, Preset);
        }
    }

    // 슬롯 타입별 처리
    if (UCanvasPanelSlot* CanvasSlot = Cast<UCanvasPanelSlot>(Slot))
    {
        return ApplyCanvasSlotProperties(CanvasSlot, SlotPropertiesJson, OutWarnings);
    }
    else if (UVerticalBoxSlot* VBoxSlot = Cast<UVerticalBoxSlot>(Slot))
    {
        return ApplyBoxSlotProperties(VBoxSlot, SlotPropertiesJson, OutWarnings);
    }
    else if (UHorizontalBoxSlot* HBoxSlot = Cast<UHorizontalBoxSlot>(Slot))
    {
        return ApplyBoxSlotProperties(HBoxSlot, SlotPropertiesJson, OutWarnings);
    }
    else if (UGridSlot* GridSlot = Cast<UGridSlot>(Slot))
    {
        return ApplyGridSlotProperties(GridSlot, SlotPropertiesJson, OutWarnings);
    }
    else if (UOverlaySlot* OverlaySlot = Cast<UOverlaySlot>(Slot))
    {
        return ApplyOverlaySlotProperties(OverlaySlot, SlotPropertiesJson, OutWarnings);
    }

    return true;
}
```

---

## 4. CanvasPanelSlot 처리

### 4.1 구현

```cpp
bool USlotPropertyHandler::ApplyCanvasSlotProperties(
    UCanvasPanelSlot* Slot,
    const TSharedPtr<FJsonObject>& JsonObject,
    TArray<FString>& OutWarnings)
{
    // Anchors
    const TSharedPtr<FJsonObject>* AnchorsObj;
    if (JsonObject->TryGetObjectField(TEXT("Anchors"), AnchorsObj))
    {
        FAnchors Anchors;
        if (ParseAnchors(*AnchorsObj, Anchors))
        {
            Slot->SetAnchors(Anchors);
        }
    }

    // Offsets (Left, Top, Right, Bottom)
    const TSharedPtr<FJsonObject>* OffsetsObj;
    if (JsonObject->TryGetObjectField(TEXT("Offsets"), OffsetsObj))
    {
        FMargin Offsets;
        Offsets.Left = (*OffsetsObj)->GetNumberField(TEXT("Left"));
        Offsets.Top = (*OffsetsObj)->GetNumberField(TEXT("Top"));
        Offsets.Right = (*OffsetsObj)->GetNumberField(TEXT("Right"));
        Offsets.Bottom = (*OffsetsObj)->GetNumberField(TEXT("Bottom"));
        Slot->SetOffsets(Offsets);
    }

    // Alignment
    const TSharedPtr<FJsonObject>* AlignmentObj;
    if (JsonObject->TryGetObjectField(TEXT("Alignment"), AlignmentObj))
    {
        FVector2D Alignment;
        Alignment.X = (*AlignmentObj)->GetNumberField(TEXT("X"));
        Alignment.Y = (*AlignmentObj)->GetNumberField(TEXT("Y"));
        Slot->SetAlignment(Alignment);
    }

    // AutoSize
    bool bAutoSize;
    if (JsonObject->TryGetBoolField(TEXT("bAutoSize"), bAutoSize))
    {
        Slot->SetAutoSize(bAutoSize);
    }

    // ZOrder
    int32 ZOrder;
    if (JsonObject->TryGetNumberField(TEXT("ZOrder"), ZOrder))
    {
        Slot->SetZOrder(ZOrder);
    }

    return true;
}
```

### 4.2 앵커 파싱

```cpp
bool USlotPropertyHandler::ParseAnchors(
    const TSharedPtr<FJsonObject>& JsonObject,
    FAnchors& OutAnchors)
{
    const TSharedPtr<FJsonObject>* MinObj;
    const TSharedPtr<FJsonObject>* MaxObj;

    if (JsonObject->TryGetObjectField(TEXT("Minimum"), MinObj))
    {
        OutAnchors.Minimum.X = (*MinObj)->GetNumberField(TEXT("X"));
        OutAnchors.Minimum.Y = (*MinObj)->GetNumberField(TEXT("Y"));
    }

    if (JsonObject->TryGetObjectField(TEXT("Maximum"), MaxObj))
    {
        OutAnchors.Maximum.X = (*MaxObj)->GetNumberField(TEXT("X"));
        OutAnchors.Maximum.Y = (*MaxObj)->GetNumberField(TEXT("Y"));
    }

    return true;
}
```

---

## 5. Box Slot 처리 (Vertical/Horizontal)

```cpp
bool USlotPropertyHandler::ApplyBoxSlotProperties(
    UPanelSlot* Slot,
    const TSharedPtr<FJsonObject>& JsonObject,
    TArray<FString>& OutWarnings)
{
    // Padding
    const TSharedPtr<FJsonObject>* PaddingObj;
    if (JsonObject->TryGetObjectField(TEXT("Padding"), PaddingObj))
    {
        FMargin Padding;
        Padding.Left = (*PaddingObj)->GetNumberField(TEXT("Left"));
        Padding.Top = (*PaddingObj)->GetNumberField(TEXT("Top"));
        Padding.Right = (*PaddingObj)->GetNumberField(TEXT("Right"));
        Padding.Bottom = (*PaddingObj)->GetNumberField(TEXT("Bottom"));

        if (UVerticalBoxSlot* VSlot = Cast<UVerticalBoxSlot>(Slot))
        {
            VSlot->SetPadding(Padding);
        }
        else if (UHorizontalBoxSlot* HSlot = Cast<UHorizontalBoxSlot>(Slot))
        {
            HSlot->SetPadding(Padding);
        }
    }

    // Size
    const TSharedPtr<FJsonObject>* SizeObj;
    if (JsonObject->TryGetObjectField(TEXT("Size"), SizeObj))
    {
        FString SizeRule;
        float Value = 1.0f;
        (*SizeObj)->TryGetStringField(TEXT("SizeRule"), SizeRule);
        (*SizeObj)->TryGetNumberField(TEXT("Value"), Value);

        FSlateChildSize ChildSize;
        ChildSize.SizeRule = SizeRule == TEXT("Fill") ? ESlateSizeRule::Fill : ESlateSizeRule::Automatic;
        ChildSize.Value = Value;

        if (UVerticalBoxSlot* VSlot = Cast<UVerticalBoxSlot>(Slot))
        {
            VSlot->SetSize(ChildSize);
        }
        else if (UHorizontalBoxSlot* HSlot = Cast<UHorizontalBoxSlot>(Slot))
        {
            HSlot->SetSize(ChildSize);
        }
    }

    // Alignment
    FString HAlign, VAlign;
    JsonObject->TryGetStringField(TEXT("HorizontalAlignment"), HAlign);
    JsonObject->TryGetStringField(TEXT("VerticalAlignment"), VAlign);

    if (UVerticalBoxSlot* VSlot = Cast<UVerticalBoxSlot>(Slot))
    {
        if (!HAlign.IsEmpty())
            VSlot->SetHorizontalAlignment(ParseHorizontalAlignment(HAlign));
        if (!VAlign.IsEmpty())
            VSlot->SetVerticalAlignment(ParseVerticalAlignment(VAlign));
    }
    else if (UHorizontalBoxSlot* HSlot = Cast<UHorizontalBoxSlot>(Slot))
    {
        if (!HAlign.IsEmpty())
            HSlot->SetHorizontalAlignment(ParseHorizontalAlignment(HAlign));
        if (!VAlign.IsEmpty())
            HSlot->SetVerticalAlignment(ParseVerticalAlignment(VAlign));
    }

    return true;
}
```

---

## 6. GridSlot 처리

```cpp
bool USlotPropertyHandler::ApplyGridSlotProperties(
    UGridSlot* Slot,
    const TSharedPtr<FJsonObject>& JsonObject,
    TArray<FString>& OutWarnings)
{
    int32 Row, Column, RowSpan, ColumnSpan;

    if (JsonObject->TryGetNumberField(TEXT("Row"), Row))
        Slot->SetRow(Row);
    
    if (JsonObject->TryGetNumberField(TEXT("Column"), Column))
        Slot->SetColumn(Column);
    
    if (JsonObject->TryGetNumberField(TEXT("RowSpan"), RowSpan))
        Slot->SetRowSpan(RowSpan);
    
    if (JsonObject->TryGetNumberField(TEXT("ColumnSpan"), ColumnSpan))
        Slot->SetColumnSpan(ColumnSpan);

    // Alignment
    FString HAlign, VAlign;
    if (JsonObject->TryGetStringField(TEXT("HorizontalAlignment"), HAlign))
        Slot->SetHorizontalAlignment(ParseHorizontalAlignment(HAlign));
    
    if (JsonObject->TryGetStringField(TEXT("VerticalAlignment"), VAlign))
        Slot->SetVerticalAlignment(ParseVerticalAlignment(VAlign));

    return true;
}
```

---

## 7. 앵커 프리셋

### 7.1 프리셋 파싱

```cpp
EUMGAnchorPreset USlotPropertyHandler::ParseAnchorPreset(const FString& PresetName)
{
    static TMap<FString, EUMGAnchorPreset> PresetMap = {
        {TEXT("TopLeft"), EUMGAnchorPreset::TopLeft},
        {TEXT("TopCenter"), EUMGAnchorPreset::TopCenter},
        {TEXT("TopRight"), EUMGAnchorPreset::TopRight},
        {TEXT("CenterLeft"), EUMGAnchorPreset::CenterLeft},
        {TEXT("Center"), EUMGAnchorPreset::Center},
        {TEXT("CenterRight"), EUMGAnchorPreset::CenterRight},
        {TEXT("BottomLeft"), EUMGAnchorPreset::BottomLeft},
        {TEXT("BottomCenter"), EUMGAnchorPreset::BottomCenter},
        {TEXT("BottomRight"), EUMGAnchorPreset::BottomRight},
        {TEXT("StretchAll"), EUMGAnchorPreset::StretchAll}
    };

    if (EUMGAnchorPreset* Found = PresetMap.Find(PresetName))
    {
        return *Found;
    }
    return EUMGAnchorPreset::Custom;
}
```

### 7.2 프리셋 앵커 값

```cpp
FAnchors USlotPropertyHandler::GetAnchorsForPreset(EUMGAnchorPreset Preset)
{
    switch (Preset)
    {
    case EUMGAnchorPreset::TopLeft:
        return FAnchors(0, 0, 0, 0);
    case EUMGAnchorPreset::TopCenter:
        return FAnchors(0.5f, 0, 0.5f, 0);
    case EUMGAnchorPreset::TopRight:
        return FAnchors(1, 0, 1, 0);
    case EUMGAnchorPreset::CenterLeft:
        return FAnchors(0, 0.5f, 0, 0.5f);
    case EUMGAnchorPreset::Center:
        return FAnchors(0.5f, 0.5f, 0.5f, 0.5f);
    case EUMGAnchorPreset::CenterRight:
        return FAnchors(1, 0.5f, 1, 0.5f);
    case EUMGAnchorPreset::BottomLeft:
        return FAnchors(0, 1, 0, 1);
    case EUMGAnchorPreset::BottomCenter:
        return FAnchors(0.5f, 1, 0.5f, 1);
    case EUMGAnchorPreset::BottomRight:
        return FAnchors(1, 1, 1, 1);
    case EUMGAnchorPreset::StretchAll:
        return FAnchors(0, 0, 1, 1);
    default:
        return FAnchors();
    }
}
```

### 7.3 프리셋 적용

```cpp
bool USlotPropertyHandler::ApplyAnchorPreset(UWidget* ChildWidget, EUMGAnchorPreset Preset)
{
    UCanvasPanelSlot* CanvasSlot = Cast<UCanvasPanelSlot>(ChildWidget->Slot);
    if (!CanvasSlot)
    {
        return false;
    }

    FAnchors Anchors = GetAnchorsForPreset(Preset);
    CanvasSlot->SetAnchors(Anchors);
    return true;
}
```

---

## 8. 정렬 파싱 헬퍼

```cpp
EHorizontalAlignment ParseHorizontalAlignment(const FString& Value)
{
    if (Value == TEXT("Left")) return EHorizontalAlignment::HAlign_Left;
    if (Value == TEXT("Center")) return EHorizontalAlignment::HAlign_Center;
    if (Value == TEXT("Right")) return EHorizontalAlignment::HAlign_Right;
    return EHorizontalAlignment::HAlign_Fill;
}

EVerticalAlignment ParseVerticalAlignment(const FString& Value)
{
    if (Value == TEXT("Top")) return EVerticalAlignment::VAlign_Top;
    if (Value == TEXT("Center")) return EVerticalAlignment::VAlign_Center;
    if (Value == TEXT("Bottom")) return EVerticalAlignment::VAlign_Bottom;
    return EVerticalAlignment::VAlign_Fill;
}
```
