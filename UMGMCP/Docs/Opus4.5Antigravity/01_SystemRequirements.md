# 01. 시스템 요구사항
## UMG MCP Asset Generator System

---

## 1. 기능 요구사항

### 1.1 위젯 생성 (FR-001 ~ FR-010)

| ID | 요구사항 | 우선순위 | 상태 |
|----|----------|----------|------|
| FR-001 | 시스템은 JSON 명세를 받아 CanvasPanel을 루트로 하는 위젯 블루프린트를 생성해야 한다 | 필수 | 설계완료 |
| FR-002 | 시스템은 VerticalBox, HorizontalBox, GridPanel 등 컨테이너 위젯을 생성할 수 있어야 한다 | 필수 | 설계완료 |
| FR-003 | 시스템은 Button, TextBlock, Image 등 기본 위젯을 생성할 수 있어야 한다 | 필수 | 설계완료 |
| FR-004 | 시스템은 중첩된 JSON 구조를 해석하여 재귀적으로 위젯 트리를 구성해야 한다 | 필수 | 설계완료 |
| FR-005 | 시스템은 각 위젯에 고유한 이름을 부여하고 관리해야 한다 | 필수 | 설계완료 |
| FR-006 | 시스템은 기존 위젯 블루프린트에 위젯을 추가할 수 있어야 한다 | 권장 | 설계완료 |
| FR-007 | 시스템은 특정 위젯을 삭제할 수 있어야 한다 | 권장 | 설계완료 |
| FR-008 | 시스템은 위젯의 부모를 변경할 수 있어야 한다 | 선택 | 미정 |
| FR-009 | 시스템은 SizeBox, ScaleBox, ScrollBox 등 특수 컨테이너를 지원해야 한다 | 권장 | 설계완료 |
| FR-010 | 시스템은 커스텀 UserWidget 클래스를 부모로 지정할 수 있어야 한다 | 필수 | 설계완료 |

### 1.2 속성 제어 (FR-011 ~ FR-020)

| ID | 요구사항 | 우선순위 | 상태 |
|----|----------|----------|------|
| FR-011 | 시스템은 UObject 리플렉션을 사용하여 위젯 속성을 동적으로 설정해야 한다 | 필수 | 설계완료 |
| FR-012 | 시스템은 Text, Color, Visibility 등 기본 속성을 지원해야 한다 | 필수 | 설계완료 |
| FR-013 | 시스템은 Font 관련 속성(FontFamily, FontSize, FontStyle)을 지원해야 한다 | 필수 | 설계완료 |
| FR-014 | 시스템은 이미지 브러시 속성(ImageSize, Tint, DrawAs)을 지원해야 한다 | 필수 | 설계완료 |
| FR-015 | 시스템은 RenderTransform(위치, 회전, 스케일)을 지원해야 한다 | 권장 | 설계완료 |
| FR-016 | 시스템은 잘못된 속성명에 대해 경고를 반환해야 한다 | 필수 | 설계완료 |
| FR-017 | 시스템은 속성 타입 불일치 시 자동 변환을 시도해야 한다 | 권장 | 설계완료 |
| FR-018 | 시스템은 에셋 참조 속성(텍스처, 폰트 등)을 경로로 지정할 수 있어야 한다 | 필수 | 설계완료 |
| FR-019 | 시스템은 열거형 속성을 문자열로 지정할 수 있어야 한다 | 필수 | 설계완료 |
| FR-020 | 시스템은 구조체 속성(FMargin, FLinearColor 등)을 JSON 객체로 지정할 수 있어야 한다 | 필수 | 설계완료 |

### 1.3 슬롯 제어 (FR-021 ~ FR-030)

| ID | 요구사항 | 우선순위 | 상태 |
|----|----------|----------|------|
| FR-021 | 시스템은 부모 위젯 타입에 따른 슬롯 클래스를 자동 감지해야 한다 | 필수 | 설계완료 |
| FR-022 | 시스템은 CanvasPanelSlot의 Anchors, Offsets, Alignment를 지원해야 한다 | 필수 | 설계완료 |
| FR-023 | 시스템은 VerticalBoxSlot/HorizontalBoxSlot의 Padding, Size, Alignment를 지원해야 한다 | 필수 | 설계완료 |
| FR-024 | 시스템은 GridSlot의 Row, Column, RowSpan, ColumnSpan을 지원해야 한다 | 필수 | 설계완료 |
| FR-025 | 시스템은 OverlaySlot의 HorizontalAlignment, VerticalAlignment를 지원해야 한다 | 필수 | 설계완료 |
| FR-026 | 시스템은 UniformGridSlot의 Row, Column을 지원해야 한다 | 권장 | 설계완료 |
| FR-027 | 시스템은 SizeBoxSlot의 제약 속성을 지원해야 한다 | 권장 | 설계완료 |
| FR-028 | 시스템은 슬롯 속성 오류 시 기본값을 적용하고 경고를 반환해야 한다 | 필수 | 설계완료 |
| FR-029 | 시스템은 프리셋 앵커(TopLeft, Center, Stretch 등)를 지원해야 한다 | 권장 | 설계완료 |
| FR-030 | 시스템은 SlotProperties를 JSON에서 별도 객체로 분리하여 관리해야 한다 | 필수 | 설계완료 |

### 1.4 에셋 관리 (FR-031 ~ FR-040)

| ID | 요구사항 | 우선순위 | 상태 |
|----|----------|----------|------|
| FR-031 | 시스템은 UWidgetBlueprintFactory를 사용하여 위젯 블루프린트 패키지를 생성해야 한다 | 필수 | 설계완료 |
| FR-032 | 시스템은 FKismetEditorUtilities::CompileBlueprint로 블루프린트를 컴파일해야 한다 | 필수 | 설계완료 |
| FR-033 | 시스템은 UPackage::SavePackage로 .uasset 파일을 저장해야 한다 | 필수 | 설계완료 |
| FR-034 | 시스템은 저장 경로를 "/Game/" 형식의 언리얼 경로로 받아야 한다 | 필수 | 설계완료 |
| FR-035 | 시스템은 동일 경로에 이미 에셋이 존재할 경우 덮어쓰기 옵션을 제공해야 한다 | 필수 | 설계완료 |
| FR-036 | 시스템은 에셋 저장 후 Content Browser를 자동 새로고침해야 한다 | 권장 | 설계완료 |
| FR-037 | 시스템은 저장 실패 시 상세한 오류 메시지를 반환해야 한다 | 필수 | 설계완료 |
| FR-038 | 시스템은 기존 위젯 블루프린트를 열어 수정할 수 있어야 한다 | 필수 | 설계완료 |
| FR-039 | 시스템은 위젯 블루프린트의 부모 클래스를 지정할 수 있어야 한다 | 필수 | 설계완료 |
| FR-040 | 시스템은 썸네일 자동 생성을 지원해야 한다 | 선택 | 미정 |

---

## 2. 비기능 요구사항

### 2.1 성능 요구사항

| ID | 요구사항 | 목표값 | 측정 방법 |
|----|----------|--------|----------|
| NFR-001 | 단일 위젯 블루프린트 생성 시간 | < 500ms | 에디터 로그 타임스탬프 |
| NFR-002 | 100개 위젯 계층 구조 생성 시간 | < 2s | 벤치마크 테스트 |
| NFR-003 | 에셋 저장 시간 | < 1s | 파일 시스템 타임스탬프 |
| NFR-004 | MCP 응답 시간 (생성+저장) | < 3s | MCP 라운드트립 측정 |
| NFR-005 | 메모리 사용량 증가 | < 50MB | 에디터 메모리 프로파일러 |

### 2.2 안정성 요구사항

| ID | 요구사항 | 설명 |
|----|----------|------|
| NFR-011 | 에디터 크래시 방지 | 잘못된 입력에도 에디터가 크래시되지 않아야 함 |
| NFR-012 | 트랜잭션 지원 | 실패 시 부분 생성된 에셋을 정리해야 함 |
| NFR-013 | 동시 요청 처리 | 순차적 요청 처리, 동시성 이슈 방지 |
| NFR-014 | 에러 복구 | 컴파일 실패 시에도 에셋 참조 무결성 유지 |

### 2.3 호환성 요구사항

| ID | 요구사항 | 대상 버전 |
|----|----------|----------|
| NFR-021 | Unreal Engine 버전 | 5.3, 5.4, 5.5 |
| NFR-022 | Python 버전 | 3.10+ |
| NFR-023 | MCP SDK 버전 | 1.0+ |
| NFR-024 | 운영체제 | Windows 10/11 (에디터 지원) |

### 2.4 유지보수성 요구사항

| ID | 요구사항 | 설명 |
|----|----------|------|
| NFR-031 | 모듈 분리 | 에디터 모듈로 분리하여 런타임 의존성 제거 |
| NFR-032 | 로깅 | 모든 주요 작업에 대한 상세 로그 출력 |
| NFR-033 | 문서화 | 모든 퍼블릭 API에 대한 주석 필수 |
| NFR-034 | 테스트 가능성 | 단위 테스트 및 통합 테스트 지원 |

---

## 3. 인터페이스 요구사항

### 3.1 MCP Tool 인터페이스

```python
# MCP Tool 정의
tools = [
    {
        "name": "umg_create_asset",
        "description": "새로운 위젯 블루프린트를 생성합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string", "description": "저장 경로 (예: /Game/UI/MyWidget)"},
                "parent_class": {"type": "string", "description": "부모 UserWidget 클래스 경로"},
                "root_widget": {"type": "object", "description": "루트 위젯 JSON 명세"},
                "overwrite": {"type": "boolean", "default": False}
            },
            "required": ["asset_path", "root_widget"]
        }
    },
    {
        "name": "umg_add_widget",
        "description": "기존 위젯 블루프린트에 위젯을 추가합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string"},
                "parent_widget_name": {"type": "string"},
                "widget_spec": {"type": "object"}
            },
            "required": ["asset_path", "parent_widget_name", "widget_spec"]
        }
    },
    {
        "name": "umg_modify_widget",
        "description": "기존 위젯의 속성을 수정합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string"},
                "widget_name": {"type": "string"},
                "properties": {"type": "object"},
                "slot_properties": {"type": "object"}
            },
            "required": ["asset_path", "widget_name"]
        }
    },
    {
        "name": "umg_remove_widget",
        "description": "위젯을 삭제합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string"},
                "widget_name": {"type": "string"}
            },
            "required": ["asset_path", "widget_name"]
        }
    },
    {
        "name": "umg_save_asset",
        "description": "위젯 블루프린트를 저장합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string"},
                "compile": {"type": "boolean", "default": True}
            },
            "required": ["asset_path"]
        }
    },
    {
        "name": "umg_get_widget_tree",
        "description": "위젯 블루프린트의 위젯 트리 구조를 조회합니다",
        "inputSchema": {
            "type": "object",
            "properties": {
                "asset_path": {"type": "string"}
            },
            "required": ["asset_path"]
        }
    }
]
```

### 3.2 C++ 서브시스템 인터페이스

```cpp
UCLASS()
class UMGMCPEDITOR_API UUMGAssetGeneratorSubsystem : public UEditorSubsystem
{
    GENERATED_BODY()

public:
    // 에셋 생성/수정
    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult CreateWidgetBlueprint(
        const FString& AssetPath,
        const FString& ParentClass,
        const FString& RootWidgetJson,
        bool bOverwrite = false
    );

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult AddWidget(
        const FString& AssetPath,
        const FString& ParentWidgetName,
        const FString& WidgetSpecJson
    );

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult ModifyWidget(
        const FString& AssetPath,
        const FString& WidgetName,
        const FString& PropertiesJson,
        const FString& SlotPropertiesJson
    );

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult RemoveWidget(
        const FString& AssetPath,
        const FString& WidgetName
    );

    // 에셋 관리
    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGOperationResult SaveAsset(
        const FString& AssetPath,
        bool bCompile = true
    );

    UFUNCTION(BlueprintCallable, Category = "UMG MCP")
    FUMGWidgetTreeInfo GetWidgetTree(const FString& AssetPath);
};
```

---

## 4. 데이터 요구사항

### 4.1 입력 데이터 형식

#### JSON 위젯 명세 구조
```json
{
    "Type": "CanvasPanel",
    "Name": "RootCanvas",
    "Properties": {
        "Visibility": "Visible"
    },
    "Children": [
        {
            "Type": "TextBlock",
            "Name": "TitleText",
            "Properties": {
                "Text": "Hello World",
                "ColorAndOpacity": {"R": 1.0, "G": 1.0, "B": 1.0, "A": 1.0},
                "Font": {
                    "FontObject": "/Game/Fonts/MyFont",
                    "Size": 24
                }
            },
            "SlotProperties": {
                "Anchors": {"Minimum": {"X": 0.5, "Y": 0}, "Maximum": {"X": 0.5, "Y": 0}},
                "Offsets": {"Left": 0, "Top": 50, "Right": 0, "Bottom": 0},
                "Alignment": {"X": 0.5, "Y": 0}
            }
        }
    ]
}
```

### 4.2 출력 데이터 형식

#### 작업 결과 응답
```json
{
    "success": true,
    "asset_path": "/Game/UI/MyWidget",
    "message": "Widget blueprint created successfully",
    "widget_count": 5,
    "warnings": [],
    "compile_status": "Success"
}
```

#### 에러 응답
```json
{
    "success": false,
    "error_code": "INVALID_WIDGET_TYPE",
    "error_message": "Unknown widget type: InvalidType",
    "details": {
        "line": 15,
        "widget_name": "BadWidget"
    }
}
```

---

## 5. 제약 조건

### 5.1 기술적 제약

| 구분 | 제약 | 영향 | 대응 방안 |
|------|------|------|----------|
| **환경** | 에디터 전용 | 런타임 사용 불가 | Editor 모듈 분리 |
| **API** | WITH_EDITOR 필수 | 패키지 빌드 제외 | 조건부 컴파일 |
| **이벤트** | BP 그래프 생성 복잡 | 이벤트 자동화 한계 | Base Class 상속 |
| **애니메이션** | UMG 애니메이션 복잡 | 애니메이션 자동화 한계 | 사전 정의 참조 |
| **데이터 바인딩** | MVVM 자동화 복잡 | 동적 바인딩 한계 | 수동 설정 또는 Base Class |

### 5.2 비즈니스 제약

| 구분 | 제약 | 설명 |
|------|------|------|
| **라이선스** | Unreal EULA | 언리얼 엔진 라이선스 준수 필요 |
| **배포** | Editor Only Plugin | Marketplace 배포 시 Editor 전용 명시 |

---

## 6. 검증 기준

### 6.1 단위 테스트 케이스

| TC-ID | 테스트 케이스 | 입력 | 예상 결과 |
|-------|--------------|------|----------|
| TC-001 | 빈 위젯 생성 | 최소 JSON (Type만) | 성공, 빈 위젯 생성 |
| TC-002 | 중첩 위젯 생성 | 3단계 중첩 JSON | 성공, 계층 구조 일치 |
| TC-003 | 속성 적용 | Text, Color 속성 | 성공, 속성값 일치 |
| TC-004 | 슬롯 속성 적용 | Anchor, Offset | 성공, 슬롯 속성 일치 |
| TC-005 | 잘못된 위젯 타입 | "InvalidType" | 실패, 에러 메시지 |
| TC-006 | 잘못된 속성명 | "InvalidProp" | 경고, 무시됨 |
| TC-007 | 덮어쓰기 비활성 | 기존 경로, false | 실패, 에러 메시지 |
| TC-008 | 덮어쓰기 활성 | 기존 경로, true | 성공, 에셋 갱신 |

### 6.2 통합 테스트 시나리오

| IS-ID | 시나리오 | 단계 | 예상 결과 |
|-------|---------|------|----------|
| IS-001 | 전체 생성 플로우 | MCP 호출 → 생성 → 저장 | .uasset 파일 생성됨 |
| IS-002 | 수정 플로우 | 열기 → 수정 → 저장 | 속성 변경 반영됨 |
| IS-003 | 삭제 플로우 | 열기 → 삭제 → 저장 | 위젯 제거됨 |
| IS-004 | 에러 복구 | 잘못된 입력 → 복구 | 에디터 안정 유지 |
