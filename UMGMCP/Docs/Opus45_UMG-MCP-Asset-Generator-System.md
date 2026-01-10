# UMG MCP Asset Generator System

## UMG MCP 에셋 생성 시스템 개발 문서

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [시스템 요구사항](#2-시스템-요구사항)
3. [시스템 아키텍처](#3-시스템-아키텍처)
4. [개발 디자인](#4-개발-디자인)
5. [개발 계획](#5-개발-계획)
6. [상세 구현 태스크](#6-상세-구현-태스크)
7. [API 명세](#7-api-명세)
8. [동적 위젯 트리 및 JSON 명세](#8-동적-위젯-트리-및-json-명세)
9. [테스트 계획](#9-테스트-계획)
10. [리스크 및 대응 방안](#10-리스크-및-대응-방안)

---

## 1. 프로젝트 개요

### 1.1 목적

Claude Code 등의 AI 에이전트가 Model Context Protocol(MCP)을 통해 언리얼 에디터에 접속하여, **JSON 명세를 기반으로 실제 위젯 블루프린트(.uasset)를 생성, 편집, 저장**할 수 있는 브릿지 시스템을 개발한다. 이를 통해 AI가 UI 레이아웃을 제안하고 즉시 프로젝트의 Content 폴더에 물리적인 에셋으로 저장하여 **프로토타이핑 속도를 혁신**한다.

### 1.2 주요 기능

- AI 에이전트의 JSON 명령을 UMG 위젯 블루프린트 생성 명령으로 변환
- JSON 명세 기반 위젯 계층 구조(Widget Tree) 재귀적 구성
- CanvasPanel, VerticalBox, HorizontalBox, GridPanel 등 컨테이너 위젯 생성
- Button, TextBlock, Image, ProgressBar 등 기본 위젯 생성
- UObject 리플렉션을 활용한 위젯 속성(Text, Color, Visibility 등) 자동 매핑
- 부모 위젯에 따른 Slot 속성(Anchors, Offsets, Padding, Alignment) 동적 처리
- **UWidgetBlueprintFactory를 통한 위젯 블루프린트 에셋 생성 및 저장**
- **에디터 타임(Editor-Time) 에셋 생성** (런타임이 아님)
- 사전 정의된 Base Widget Class 지정을 통한 이벤트 바인딩 우회 전략

### 1.3 기대 효과

- AI 기반 실시간 UI 프로토타이핑 가능
- 반복적인 UI 레이아웃 작업 시간 대폭 단축
- 버전 관리 가능한 실제 .uasset 에셋 생성

---

## 2. 시스템 요구사항

### 2.1 기능적 요구사항

#### FR-001: 컨테이너 위젯 생성
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-001-1 | CanvasPanel 생성 | 필수 |
| FR-001-2 | VerticalBox 생성 | 필수 |
| FR-001-3 | HorizontalBox 생성 | 필수 |
| FR-001-4 | GridPanel 생성 | 필수 |
| FR-001-5 | Overlay 생성 | 선택 |
| FR-001-6 | ScrollBox 생성 | 선택 |

#### FR-002: 리프 위젯 생성
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-002-1 | Button 생성 | 필수 |
| FR-002-2 | TextBlock 생성 | 필수 |
| FR-002-3 | Image 생성 | 필수 |
| FR-002-4 | ProgressBar 생성 | 필수 |
| FR-002-5 | EditableTextBox 생성 | 선택 |
| FR-002-6 | CheckBox 생성 | 선택 |

#### FR-003: 계층 구조 제어
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-003-1 | JSON 중첩 구조 해석 | 필수 |
| FR-003-2 | 재귀적 위젯 트리 구성 | 필수 |
| FR-003-3 | 부모-자식 관계 설정 | 필수 |
| FR-003-4 | 최대 10단계 깊이 지원 | 필수 |

#### FR-004: 속성 제어 (Reflection)
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-004-1 | Text 속성 설정 (FText) | 필수 |
| FR-004-2 | Color 속성 설정 (FLinearColor) | 필수 |
| FR-004-3 | Visibility 속성 설정 | 필수 |

#### FR-005: 슬롯 속성 제어
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-005-1 | CanvasPanelSlot - Anchors/Offsets | 필수 |
| FR-005-2 | VerticalBoxSlot - Padding/Alignment | 필수 |
| FR-005-3 | HorizontalBoxSlot - Padding/Alignment | 필수 |
| FR-005-4 | GridSlot - Row/Column | 필수 |

#### FR-006: 에셋 저장
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-006-1 | UWidgetBlueprintFactory 패키지 생성 | 필수 |
| FR-006-2 | FKismetEditorUtilities::CompileBlueprint | 필수 |
| FR-006-3 | UPackage::SavePackage .uasset 저장 | 필수 |
| FR-006-4 | AssetRegistry 등록 | 필수 |

#### FR-007: Base Class 및 이벤트 처리
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-007-1 | 사용자 지정 Base UserWidget 클래스 지정 | 필수 |
| FR-007-2 | BindWidget 메타 지원 | 필수 |
| FR-007-3 | C++ Base Class 이벤트 핸들링 우회 | 필수 |

### 2.2 비기능적 요구사항

| ID | 요구사항 | 목표값 |
|---|---|---|
| NFR-001 | 에디터 크래시 방지 | 잘못된 JSON 시 에러 반환 |
| NFR-002 | GameThread 안전성 | 에디터 스레드에서 에셋 작업 |
| NFR-003 | 언리얼 버전 호환 | 5.2 이상 |
| NFR-004 | 경로 검증 | /Game/ 내부만 허용 |

### 2.3 기술 스택

#### 언리얼 엔진 측 (C++)
```
- Unreal Engine 5.2+
- UMG, UMGEditor, UnrealEd Module
- Sockets, JsonUtilities Module
- WITH_EDITOR 전처리 블록 필수
```

#### MCP 서버 측 (Python)
```
- Python 3.10+
- mcp (Model Context Protocol SDK)
- asyncio, pydantic
```

---

## 3. 시스템 아키텍처

### 3.1 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        AI Agent Layer                           │
│  ┌─────────────────┐                                            │
│  │   Claude Code   │                                            │
│  └────────┬────────┘                                            │
│           │ MCP Protocol                                        │
└───────────┼─────────────────────────────────────────────────────┘
            │
┌───────────┼─────────────────────────────────────────────────────┐
│           │         MCP Server Layer (Python)                   │
│  ┌────────┴────────┐                                            │
│  │ UMG Asset MCP   │                                            │
│  │    Server       │                                            │
│  └────────┬────────┘                                            │
│           │ TCP                                                 │
└───────────┼─────────────────────────────────────────────────────┘
            │
┌───────────┼─────────────────────────────────────────────────────┐
│           │     Unreal Editor Layer (C++) [WITH_EDITOR]         │
│  ┌────────┴─────────────────────────────────────────────────┐   │
│  │           UUMGAssetGeneratorSubsystem                    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────┐   │   │
│  │  │Socket Server │ │WidgetTree   │ │ Property       │   │   │
│  │  │              │ │Builder      │ │ Reflector      │   │   │
│  │  └──────────────┘ └──────────────┘ └────────────────┘   │   │
│  │                                                          │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │        WidgetBlueprintCreator (.uasset 생성)     │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 데이터 흐름

```
AI Agent → MCP Tool Call (JSON UI Spec)
    ↓
MCP Server: JSON 검증
    ↓
TCP Message → Unreal Editor
    ↓
UWidgetTreeBuilder: 재귀적 위젯 트리 구성
    ↓
UPropertyReflector: 속성/슬롯 적용
    ↓
UWidgetBlueprintFactory: 블루프린트 생성
    ↓
CompileBlueprint → SavePackage → .uasset
    ↓
Response → AI Agent
```

### 3.3 모듈 구성

#### Unreal Engine 모듈
```
UMGMCPPlugin/
├── Source/UMGMCPBridge/
│   ├── Public/
│   │   ├── UMGAssetGeneratorSubsystem.h
│   │   ├── WidgetTreeBuilder.h
│   │   ├── PropertyReflector.h
│   │   ├── SlotPropertyResolver.h
│   │   └── WidgetBlueprintCreator.h
│   └── Private/
│       └── (구현 파일들)
└── UMGMCPBridge.Build.cs
```

#### Python MCP Server 모듈
```
umg_mcp_server/
├── src/
│   ├── server.py
│   ├── tools/
│   │   ├── asset_tools.py
│   │   └── widget_tools.py
│   ├── connection/
│   │   └── socket_client.py
│   └── models/
│       ├── commands.py
│       └── widget_spec.py
└── pyproject.toml
```

---

## 4. 개발 디자인

### 4.1 핵심 클래스 (C++)

```cpp
// UMGAssetGeneratorSubsystem - 에디터 서브시스템
#if WITH_EDITOR
UCLASS()
class UUMGAssetGeneratorSubsystem : public UEditorSubsystem
{
    GENERATED_BODY()

public:
    void StartServer(int32 Port = 9998);
    void StopServer();
    
    UFUNCTION(BlueprintCallable)
    FUMGAssetCreateResult CreateWidgetBlueprintFromJSON(
        const FString& JsonSpec,
        const FString& AssetPath,
        const FString& AssetName,
        TSubclassOf<UUserWidget> BaseClass = nullptr
    );

private:
    TUniquePtr<FUMGMCPSocketServer> SocketServer;
    UWidgetTreeBuilder* TreeBuilder;
    UWidgetBlueprintCreator* BlueprintCreator;
};
#endif

// UWidgetTreeBuilder - 위젯 트리 재귀 생성기
UCLASS()
class UWidgetTreeBuilder : public UObject
{
    GENERATED_BODY()

public:
    FWidgetBuildResult BuildWidgetTree(
        UPanelWidget* RootWidget,
        const FString& JsonSpec,
        UWidgetTree* WidgetTree
    );

private:
    UWidget* CreateWidgetRecursive(
        UPanelWidget* Parent,
        const TSharedPtr<FJsonObject>& WidgetJson,
        UWidgetTree* WidgetTree,
        int32 Depth
    );
    
    TSubclassOf<UWidget> ResolveWidgetClass(const FString& WidgetType);
    
    TMap<FString, TSubclassOf<UWidget>> WidgetClassMap;
    static constexpr int32 MaxDepth = 10;
};

// 데이터 타입
USTRUCT()
struct FUMGAssetCreateResult
{
    GENERATED_BODY()
    
    bool bSuccess = false;
    FString AssetPath;
    FString ErrorMessage;
    int32 WidgetCount = 0;
};
```

### 4.2 핵심 클래스 (Python)

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum
import uuid

class WidgetType(str, Enum):
    CANVAS_PANEL = "CanvasPanel"
    VERTICAL_BOX = "VerticalBox"
    HORIZONTAL_BOX = "HorizontalBox"
    BUTTON = "Button"
    TEXT_BLOCK = "TextBlock"
    IMAGE = "Image"

class WidgetSpec(BaseModel):
    type: WidgetType
    name: str
    properties: dict = Field(default_factory=dict)
    slot_properties: Optional[dict] = None
    children: list["WidgetSpec"] = Field(default_factory=list)

class CreateAssetCommand(BaseModel):
    command_type: str = "create_asset"
    request_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    asset_path: str
    asset_name: str
    widget_spec: dict
    base_class: str = ""

class UMGResponse(BaseModel):
    request_id: str
    success: bool
    message: str
    created_asset_path: Optional[str] = None
```

### 4.3 통신 프로토콜

**요청 (MCP Server → Unreal Editor)**
```json
{
    "request_id": "uuid-string",
    "command_type": "create_asset",
    "asset_path": "/Game/UI/Widgets",
    "asset_name": "WBP_MainMenu",
    "widget_spec": {
        "type": "CanvasPanel",
        "name": "RootCanvas",
        "children": [
            {
                "type": "Button",
                "name": "StartButton",
                "slot_properties": {"anchor_preset": "Center"}
            }
        ]
    }
}
```

**응답 (Unreal Editor → MCP Server)**
```json
{
    "request_id": "uuid-string",
    "success": true,
    "message": "위젯 블루프린트가 생성되었습니다",
    "created_asset_path": "/Game/UI/Widgets/WBP_MainMenu"
}
```

---

## 5. 개발 계획

### 5.1 프로젝트 일정

```
총 예상 기간: 6주

Phase 1: 기반 구축 (Week 1-2)
├── 언리얼 플러그인 프로젝트 생성 (WITH_EDITOR)
├── Python MCP 서버 프로젝트 생성
├── TCP 소켓 통신 구현
└── UWidgetBlueprintCreator 기본 구현

Phase 2: 핵심 기능 (Week 3-4)
├── UWidgetTreeBuilder 재귀 생성기
├── 컨테이너/리프 위젯 지원
├── UPropertyReflector 리플렉션 시스템
└── USlotPropertyResolver 슬롯 처리

Phase 3: 안정화 (Week 5-6)
├── 에러 핸들링 강화
├── 테스트
└── 문서화
```

### 5.2 마일스톤

| 마일스톤 | 목표 | 예정일 |
|---------|------|--------|
| M1 | MCP↔Unreal 메시지 왕복 | Week 1 |
| M2 | 빈 위젯 블루프린트 .uasset 생성 | Week 2 |
| M3 | JSON으로 위젯 트리 구성 | Week 3 |
| M4 | 리플렉션 기반 속성/슬롯 적용 | Week 4 |
| M5 | 에러처리 완료 | Week 5 |
| M6 | 테스트 통과 | Week 6 |

---

## 6. 상세 구현 태스크

### Phase 1: 기반 구축

| Task ID | 태스크 | 예상 시간 |
|---------|--------|----------|
| T1.1 | 언리얼 플러그인 생성, 모듈 설정 | 5h |
| T1.2 | Python 프로젝트 생성 | 2h |
| T1.3 | TCP 서버/클라이언트 구현 | 7h |
| T1.4 | 메시지 프로토콜 정의 | 2h |
| T1.5 | EditorSubsystem 구현 | 4h |
| T1.6 | BlueprintCreator 기본 | 4h |
| T1.7 | 패키지 생성/저장 로직 | 6h |

### Phase 2: 핵심 기능

| Task ID | 태스크 | 예상 시간 |
|---------|--------|----------|
| T2.1 | WidgetTreeBuilder 구현 | 4h |
| T2.2 | 위젯 클래스 매핑 | 3h |
| T2.3 | 컨테이너/리프 위젯 생성 | 8h |
| T2.4 | 재귀 트리 구성 | 4h |
| T2.5 | PropertyReflector 구현 | 4h |
| T2.6 | 타입 변환기 (FText, Color 등) | 4h |
| T2.7 | SlotPropertyResolver 구현 | 4h |
| T2.8 | 블루프린트 컴파일 연동 | 3h |
| T2.9 | MCP Tool 정의 | 3h |

---

## 7. API 명세

### 7.1 MCP Tools 목록

| Tool Name | 설명 | 파라미터 |
|-----------|------|----------|
| `umg_create_asset` | 위젯 블루프린트 생성 | asset_path, asset_name, widget_spec, base_class |
| `umg_add_widget` | 위젯 추가 | asset_path, parent_widget_name, widget_spec |
| `umg_save_asset` | 에셋 저장 | asset_path |
| `umg_remove_widget` | 위젯 제거 | asset_path, widget_name |
| `umg_list_widgets` | 위젯 목록 조회 | asset_path |

### 7.2 Tool 스키마 예시

```json
{
    "name": "umg_create_asset",
    "description": "JSON 명세 기반 위젯 블루프린트 생성",
    "inputSchema": {
        "type": "object",
        "properties": {
            "asset_path": {"type": "string", "description": "저장 경로"},
            "asset_name": {"type": "string", "description": "에셋 이름"},
            "widget_spec": {"type": "object", "description": "위젯 트리 명세"},
            "base_class": {"type": "string", "description": "Base 클래스 (선택)"}
        },
        "required": ["asset_path", "asset_name", "widget_spec"]
    }
}
```

---

## 8. 동적 위젯 트리 및 JSON 명세

### 8.1 핵심 워크플로우

```
AI (Claude Code)
    ↓ "메인 메뉴를 만들어줘. 버튼 3개가 세로로 배치"
    ↓
JSON 명세 생성
    ↓ {"type": "CanvasPanel", "children": [...]}
    ↓
Widget Tree 구성 → .uasset 저장
    ↓
Content Browser에서 확인 가능
```

### 8.2 JSON 스키마

```json
{
    "type": "WidgetType",
    "name": "WidgetInstanceName",
    "properties": {
        "Text": "텍스트 내용",
        "ColorAndOpacity": {"r": 1, "g": 1, "b": 1, "a": 1}
    },
    "slot_properties": {
        "anchor_preset": "Center",
        "padding": {"left": 10, "top": 5, "right": 10, "bottom": 5}
    },
    "children": []
}
```

### 8.3 슬롯 속성

**CanvasPanelSlot**
```json
{
    "anchor_preset": "Center",
    "offsets": {"left": -100, "top": -25, "right": 100, "bottom": 25},
    "alignment": {"x": 0.5, "y": 0.5}
}
```

**BoxSlot (VerticalBox/HorizontalBox)**
```json
{
    "padding": {"left": 10, "top": 5, "right": 10, "bottom": 5},
    "h_align": "Fill",
    "v_align": "Center",
    "size": {"rule": "Fill", "value": 1.0}
}
```

### 8.4 Anchor 프리셋

| 프리셋 | Min | Max |
|--------|-----|-----|
| TopLeft | (0,0) | (0,0) |
| Center | (0.5,0.5) | (0.5,0.5) |
| BottomRight | (1,1) | (1,1) |
| StretchFull | (0,0) | (1,1) |

### 8.5 이벤트 처리 전략: Base Class 상속

블루프린트 그래프 노드 생성이 복잡하므로 **C++ Base Class에서 이벤트 처리**:

```cpp
// MyBaseMenuWidget.h
UCLASS(Abstract)
class UMyBaseMenuWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    // AI가 생성하는 위젯 이름과 일치하면 자동 바인딩
    UPROPERTY(meta = (BindWidget))
    class UButton* StartButton;
    
    UPROPERTY(meta = (BindWidget))
    class UButton* ExitButton;

protected:
    virtual void NativeConstruct() override
    {
        if (StartButton)
            StartButton->OnClicked.AddDynamic(this, &UMyBaseMenuWidget::OnStartClicked);
    }
    
    UFUNCTION()
    virtual void OnStartClicked() {}
};
```

### 8.6 UI 템플릿 예제

```json
{
    "type": "CanvasPanel",
    "name": "RootCanvas",
    "children": [
        {
            "type": "VerticalBox",
            "name": "MenuContainer",
            "slot_properties": {"anchor_preset": "Center"},
            "children": [
                {"type": "TextBlock", "name": "TitleText", "properties": {"Text": "GAME TITLE"}},
                {"type": "Button", "name": "StartButton", "children": [
                    {"type": "TextBlock", "name": "StartText", "properties": {"Text": "START"}}
                ]},
                {"type": "Button", "name": "ExitButton", "children": [
                    {"type": "TextBlock", "name": "ExitText", "properties": {"Text": "EXIT"}}
                ]}
            ]
        }
    ]
}
```

---

## 9. 테스트 계획

### 9.1 기능 검증 테스트

| 테스트 ID | 테스트 항목 | 검증 내용 |
|-----------|-----------|----------|
| T-001 | MCP 통신 연결 | 서버-클라이언트 메시지 왕복 |
| T-002 | 빈 에셋 생성 | 빈 위젯 블루프린트 .uasset 생성 |
| T-003 | 단일 위젯 생성 | Button, TextBlock 등 단일 위젯 |
| T-004 | 위젯 트리 생성 | 3단계 이상 중첩 위젯 트리 |
| T-005 | 속성 적용 | Text, Color 속성 정상 적용 |
| T-006 | 슬롯 속성 적용 | Anchor, Padding 정상 적용 |
| T-007 | 에셋 저장 | Content Browser에서 확인 |
| T-008 | 에러 처리 | 잘못된 JSON에 대한 에러 반환 |

---

## 10. 리스크 및 대응 방안

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| 에디터 크래시 | 높음 | GameThread 보장, 예외 처리 강화 |
| 위젯 타입 미지원 | 중간 | 확장 가능한 클래스 매핑 구조 |
| 컴파일 오류 | 중간 | 사전 검증, 롤백 메커니즘 |
| 리플렉션 실패 | 중간 | 속성 화이트리스트 적용 |
| 언리얼 API 복잡성 | 중간 | 버퍼 기간 확보, MVP 우선 |

---

## 부록

### A. 환경 설정

#### Unreal Engine
```ini
; DefaultUMGMCP.ini
[/Script/UMGMCPBridge.UMGMCPSettings]
bAutoStartServer=true
ServerPort=9998
MaxWidgetDepth=10
```

#### Python MCP Server
```yaml
# settings.yaml
server:
  name: "umg-asset-generator"
unreal:
  host: "localhost"
  port: 9998
```

#### Claude Code MCP
```json
{
    "mcpServers": {
        "umg": {
            "command": "python",
            "args": ["-m", "umg_mcp_server"]
        }
    }
}
```

### B. 참고 자료

- [Unreal Engine UMG Documentation](https://docs.unrealengine.com/5.0/en-US/umg-ui-designer/)
- [MCP Specification](https://modelcontextprotocol.io/)

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|----------|
| 1.0 | 2024-XX-XX | 초안 작성 |

---

*이 문서는 UMG MCP Asset Generator System 개발의 기준 문서로 사용됩니다.*
