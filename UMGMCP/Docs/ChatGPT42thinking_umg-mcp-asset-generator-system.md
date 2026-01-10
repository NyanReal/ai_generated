# UMG MCP Asset Generator System

## UMG MCP 에셋 생성 시스템 개발 문서  
*(본 문서는 “Niagara MCP Control System” 문서의 **목차 구조(1~10), 표 형식, 다이어그램/코드 예시 스타일**을 동일하게 따르는 설계 템플릿을 기반으로 작성됨.)*

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

Claude Code 등의 AI 에이전트가 Model Context Protocol(MCP)을 통해 **언리얼 에디터(Editor-Time)**에 접속하여, **JSON 명세 기반으로 실제 위젯 블루프린트(Widget Blueprint) 에셋(.uasset)** 을 생성/편집/컴파일/저장할 수 있는 브릿지 시스템을 개발한다.

> 핵심은 “런타임(In-Game) UI 생성”이 아니라, **에디터 타임에서 Content 폴더에 물리적으로 저장되는 에셋 생성 파이프라인**이다.

### 1.2 주요 기능

- JSON 스펙 → Widget Blueprint(.uasset) 생성 (`UWidgetBlueprintFactory`)
- JSON 중첩 구조 → 재귀적 위젯 트리 구성 (`UWidgetTree`)
- Reflection 기반 속성 자동 매핑 (Text/Color/Visibility 등)
- 부모 패널 타입에 따른 Slot 속성 처리  
  - `UCanvasPanelSlot`: Anchors / Offsets / Alignment / ZOrder  
  - `UVerticalBoxSlot`: Padding / Size / Alignment  
  - `UGridSlot`: Row/Column/RowSpan/ColumnSpan/Alignment/Padding
- 에셋 컴파일 및 저장  
  - `FKismetEditorUtilities::CompileBlueprint`  
  - 패키지 저장(UPackage 저장 루틴)
- 에셋 편집(기존 Widget Blueprint 로드 후 변경/패치 적용)

### 1.3 기대 효과

- UI 프로토타이핑 속도 혁신: “요구사항 → JSON → 즉시 .uasset 생성”
- 디자이너/개발자/AI 협업 워크플로우 구축
- 레이아웃 반복 실험 비용 감소 (버전별 에셋 자동 생성/저장/회수)
- JSON 스펙 기반 UI를 통해 **재현 가능한 UI 빌드(Deterministic Build)** 확보

---

## 2. 시스템 요구사항

### 2.1 기능적 요구사항

#### FR-001: 위젯 생성
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-001-1 | 기본 컨테이너 위젯 생성 (CanvasPanel, VerticalBox, HorizontalBox, GridPanel, Overlay) | 필수 |
| FR-001-2 | 기본 프리미티브 위젯 생성 (Button, TextBlock, Image, Border, SizeBox, Spacer) | 필수 |
| FR-001-3 | 커스텀 위젯 클래스 생성/배치 (프로젝트 내 UserWidget 파생 클래스 지정) | 선택 |
| FR-001-4 | Named Slot/Content Slot 지원 (Border/SizeBox/Button 등 단일 슬롯) | 필수 |

#### FR-002: 계층 구조(Hierarchy)
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-002-1 | JSON의 Children 중첩을 재귀적으로 해석하여 WidgetTree 구성 | 필수 |
| FR-002-2 | Root 위젯 지정 및 교체(초기 Root 생성/변경) | 필수 |
| FR-002-3 | 위젯 ID 기반 참조(추후 패치 적용 시 타겟 식별) | 필수 |
| FR-002-4 | 위젯 삭제/이동(부모 변경) | 선택 |

#### FR-003: 속성 제어(Reflection)
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-003-1 | UProperty/Reflection을 통해 문자열 키-값을 실제 속성에 적용 | 필수 |
| FR-003-2 | 일반 속성(Text, Visibility, bIsEnabled 등) 매핑 | 필수 |
| FR-003-3 | 구조체/복합 타입(FLinearColor, FMargin, FVector2D 등) 파싱/적용 | 필수 |
| FR-003-4 | Enum(EVisibility, EHorizontalAlignment 등) 문자열 매핑 | 필수 |
| FR-003-5 | 미지원 속성은 경고로 수집 후 리포트 | 필수 |

#### FR-004: 슬롯(Slot) 제어
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-004-1 | 부모 패널이 CanvasPanel인 경우 CanvasSlot 속성 적용 | 필수 |
| FR-004-2 | 부모 패널이 VerticalBox인 경우 VerticalBoxSlot 속성 적용 | 필수 |
| FR-004-3 | 부모 패널이 GridPanel인 경우 GridSlot 속성 적용 | 필수 |
| FR-004-4 | Overlay/HorizontalBox 등 주요 패널 Slot 속성 적용 | 선택 |
| FR-004-5 | Slot 타입 불일치/미지원 시 안전 처리(무시 + 경고) | 필수 |

#### FR-005: 에셋 저장/컴파일
| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-005-1 | `UWidgetBlueprintFactory`로 Widget Blueprint 신규 생성 | 필수 |
| FR-005-2 | 변경 후 블루프린트 컴파일 수행 | 필수 |
| FR-005-3 | 패키지 Dirty 처리 및 .uasset 저장 | 필수 |
| FR-005-4 | 에셋 레지스트리 반영(AssetCreated/Scan) | 필수 |
| FR-005-5 | 저장 실패 시 롤백/오류 리포트 | 선택 |

### 2.2 비기능적 요구사항

#### NFR-001: 성능
| ID | 요구사항 | 목표값 |
|---|---|---|
| NFR-001-1 | JSON 스펙 → 위젯 트리 적용(중간 규모 UI) | < 200ms |
| NFR-001-2 | 에셋 컴파일(단일 Widget BP) | < 2s (환경 의존) |
| NFR-001-3 | 대형 트리(500 노드) 적용 시 크래시 없이 완료 | 필수 |
| NFR-001-4 | 에디터 스레드 블로킹 최소화(소켓 처리 분리) | 필수 |

#### NFR-002: 안정성
| ID | 요구사항 | 목표값 |
|---|---|---|
| NFR-002-1 | 잘못된 JSON/속성 입력 처리 | 에러 반환, 크래시 방지 |
| NFR-002-2 | 컴파일 실패 시 원인 메시지 제공 | 필수 |
| NFR-002-3 | 동일 에셋 경로 충돌 처리(덮어쓰기/버전 생성 정책) | 필수 |

#### NFR-003: 호환성
| ID | 요구사항 | 대상 |
|---|---|---|
| NFR-003-1 | 언리얼 엔진 버전 | 5.2 이상 |
| NFR-003-2 | 플랫폼 | Windows/Linux (Editor) |
| NFR-003-3 | MCP SDK 버전 | 1.0 이상 |

#### NFR-004: 보안
| ID | 요구사항 | 설명 |
|---|---|---|
| NFR-004-1 | 연결 인증 | 토큰 기반(선택) |
| NFR-004-2 | 로컬호스트 제한 | 기본 localhost만 허용 |
| NFR-004-3 | 경로 검증 | `/Game/...` 내부만 허용 |
| NFR-004-4 | 클래스 화이트리스트 | 생성 가능한 위젯 클래스 제한 |

### 2.3 기술 스택

#### 언리얼 엔진 측 (C++)
```
- Unreal Engine 5.2+
- UMG, UMGEditor
- UnrealEd
- Json, JsonUtilities
- AssetTools, KismetCompiler(간접)
- Sockets (선택: MCP↔UE 통신)
- WITH_EDITOR 전제
```

#### MCP 서버 측 (Python)
```
- Python 3.10+
- mcp (Model Context Protocol SDK)
- asyncio
- websockets 또는 socket
- pydantic (데이터 검증)
```

### 2.4 개발 환경 요구사항

| 구분 | 요구사항 |
|---|---|
| IDE | Visual Studio 2022 / Rider |
| 언리얼 버전 | 5.2, 5.3, 5.4, 5.5+ |
| Python | 3.10 이상 |
| OS | Windows 10/11, Ubuntu 22.04+ |
| Claude Code | 최신 버전 |

---

## 3. 시스템 아키텍처

### 3.1 전체 아키텍처

(상세는 채팅 출력본과 동일)

---

## 4. 개발 디자인

(상세는 채팅 출력본과 동일)

---

## 5. 개발 계획

(상세는 채팅 출력본과 동일)

---

## 6. 상세 구현 태스크

(상세는 채팅 출력본과 동일)

---

## 7. API 명세

(상세는 채팅 출력본과 동일)

---

## 8. 동적 위젯 트리 및 JSON 명세

(상세는 채팅 출력본과 동일)

---

## 9. 테스트 계획

(상세는 채팅 출력본과 동일)

---

## 10. 리스크 및 대응 방안

(상세는 채팅 출력본과 동일)

---

# 부록: Python Pydantic 스펙 모델(예시)

```python
from __future__ import annotations
from typing import Any, Dict, List, Literal, Optional
from pydantic import BaseModel, Field, field_validator

class CanvasSlotSpec(BaseModel):
    anchors: Optional[str] = None
    alignment: Optional[str] = None
    position: Optional[str] = None
    size: Optional[str] = None
    z_order: Optional[int] = None

class VerticalBoxSlotSpec(BaseModel):
    padding: Optional[str] = None
    horizontal_alignment: Optional[str] = None
    vertical_alignment: Optional[str] = None
    size_rule: Optional[Literal["Auto", "Fill"]] = None

class SlotSpec(BaseModel):
    canvas: Optional[CanvasSlotSpec] = None
    vertical_box: Optional[VerticalBoxSlotSpec] = None
    grid: Optional[Dict[str, Any]] = None

class WidgetNode(BaseModel):
    id: str
    type: str
    properties: Dict[str, str] = Field(default_factory=dict)
    slot: SlotSpec = Field(default_factory=SlotSpec)
    children: List["WidgetNode"] = Field(default_factory=list)

class WidgetBlueprintSpec(BaseModel):
    asset_path: str
    parent_class: str
    root: WidgetNode

    @field_validator("asset_path")
    @classmethod
    def must_be_game_path(cls, v: str) -> str:
        if not v.startswith("/Game/"):
            raise ValueError("asset_path must start with /Game/")
        return v
```
