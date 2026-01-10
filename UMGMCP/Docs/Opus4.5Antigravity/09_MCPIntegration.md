# 09. MCP 연동 및 Python 도구
## UMG MCP Asset Generator System

---

## 1. 개요

MCP(Model Context Protocol)를 통해 AI 에이전트가 언리얼 에디터의 UMG 에셋 생성 기능에 접근할 수 있도록 하는 Python 서버입니다.

### 1.1 아키텍처

```
┌─────────────────┐     MCP Protocol     ┌─────────────────┐
│   AI Agent      │◄───────────────────►│   MCP Server    │
│   (Claude)      │     (stdio/SSE)      │   (Python)      │
└─────────────────┘                      └────────┬────────┘
                                                  │ HTTP
                                                  ▼
                                         ┌─────────────────┐
                                         │  Unreal Editor  │
                                         │  (HTTP Server)  │
                                         └─────────────────┘
```

---

## 2. Python 프로젝트 구조

```
Python/
├── requirements.txt
├── pyproject.toml
├── umg_mcp_server/
│   ├── __init__.py
│   ├── server.py          # MCP 서버 메인
│   ├── tools.py           # MCP 도구 정의
│   ├── models.py          # Pydantic 모델
│   ├── ue_client.py       # UE HTTP 클라이언트
│   └── config.py          # 설정
└── tests/
    ├── __init__.py
    ├── test_models.py
    └── test_tools.py
```

---

## 3. Pydantic 모델

### 3.1 models.py

```python
"""UMG MCP Pydantic 모델 정의"""

from typing import Optional, List, Dict, Any, Union
from pydantic import BaseModel, Field, field_validator
from enum import Enum


class Vector2D(BaseModel):
    """2D 벡터"""
    X: float = 0.0
    Y: float = 0.0


class LinearColor(BaseModel):
    """선형 색상 (RGBA)"""
    R: float = Field(default=1.0, ge=0.0, le=1.0)
    G: float = Field(default=1.0, ge=0.0, le=1.0)
    B: float = Field(default=1.0, ge=0.0, le=1.0)
    A: float = Field(default=1.0, ge=0.0, le=1.0)


class Margin(BaseModel):
    """여백"""
    Left: float = 0.0
    Top: float = 0.0
    Right: float = 0.0
    Bottom: float = 0.0


class Anchors(BaseModel):
    """앵커 설정"""
    Minimum: Vector2D = Field(default_factory=Vector2D)
    Maximum: Vector2D = Field(default_factory=Vector2D)


class CanvasSlotProperties(BaseModel):
    """CanvasPanel 슬롯 속성"""
    AnchorPreset: Optional[str] = None
    Anchors: Optional[Anchors] = None
    Offsets: Optional[Margin] = None
    Alignment: Optional[Vector2D] = None
    bAutoSize: bool = False
    ZOrder: int = 0


class BoxSlotProperties(BaseModel):
    """Vertical/HorizontalBox 슬롯 속성"""
    Padding: Optional[Margin] = None
    HorizontalAlignment: str = "Fill"
    VerticalAlignment: str = "Fill"
    SizeRule: str = "Auto"
    SizeValue: float = 1.0


class GridSlotProperties(BaseModel):
    """GridPanel 슬롯 속성"""
    Row: int = 0
    Column: int = 0
    RowSpan: int = 1
    ColumnSpan: int = 1
    HorizontalAlignment: str = "Fill"
    VerticalAlignment: str = "Fill"


class WidgetSpec(BaseModel):
    """위젯 명세"""
    Type: str = Field(..., description="위젯 타입명")
    Name: Optional[str] = Field(None, description="위젯 이름 (자동 생성 가능)")
    Properties: Optional[Dict[str, Any]] = Field(None, description="위젯 속성")
    SlotProperties: Optional[Dict[str, Any]] = Field(None, description="슬롯 속성")
    Children: Optional[List["WidgetSpec"]] = Field(None, description="자식 위젯")

    @field_validator('Type')
    @classmethod
    def validate_type(cls, v: str) -> str:
        valid_types = {
            'CanvasPanel', 'VerticalBox', 'HorizontalBox', 'GridPanel',
            'Overlay', 'SizeBox', 'ScaleBox', 'ScrollBox', 'WidgetSwitcher',
            'TextBlock', 'Image', 'Button', 'Border', 'ProgressBar',
            'Slider', 'CheckBox', 'EditableText', 'Spacer', 'Throbber'
        }
        if v not in valid_types:
            raise ValueError(f"Invalid widget type: {v}")
        return v


# 순환 참조 해결
WidgetSpec.model_rebuild()


class CreateWidgetRequest(BaseModel):
    """위젯 블루프린트 생성 요청"""
    asset_path: str = Field(..., description="에셋 경로 (예: /Game/UI/MyWidget)")
    parent_class: Optional[str] = Field(None, description="부모 UserWidget 클래스")
    root_widget: WidgetSpec = Field(..., description="루트 위젯 명세")
    overwrite: bool = Field(False, description="기존 에셋 덮어쓰기")


class AddWidgetRequest(BaseModel):
    """위젯 추가 요청"""
    asset_path: str
    parent_widget_name: str = Field(..., description="부모 위젯 이름")
    widget_spec: WidgetSpec


class ModifyWidgetRequest(BaseModel):
    """위젯 수정 요청"""
    asset_path: str
    widget_name: str
    properties: Optional[Dict[str, Any]] = None
    slot_properties: Optional[Dict[str, Any]] = None


class RemoveWidgetRequest(BaseModel):
    """위젯 삭제 요청"""
    asset_path: str
    widget_name: str


class SaveAssetRequest(BaseModel):
    """에셋 저장 요청"""
    asset_path: str
    compile: bool = True


class OperationResult(BaseModel):
    """작업 결과"""
    success: bool
    asset_path: Optional[str] = None
    widget_count: int = 0
    message: Optional[str] = None
    error_code: Optional[int] = None
    error_message: Optional[str] = None
    warnings: List[str] = Field(default_factory=list)
    compile_status: Optional[str] = None


class WidgetInfo(BaseModel):
    """위젯 정보"""
    name: str
    type: str
    parent_name: Optional[str] = None
    children_names: List[str] = Field(default_factory=list)
    slot_type: Optional[str] = None


class WidgetTreeInfo(BaseModel):
    """위젯 트리 정보"""
    asset_path: str
    root_widget_name: Optional[str] = None
    widgets: List[WidgetInfo] = Field(default_factory=list)
    total_widget_count: int = 0
    parent_class_name: Optional[str] = None
```

---

## 4. UE HTTP 클라이언트

### 4.1 ue_client.py

```python
"""Unreal Editor HTTP 클라이언트"""

import httpx
from typing import Optional, Dict, Any
from .models import (
    CreateWidgetRequest, AddWidgetRequest, ModifyWidgetRequest,
    RemoveWidgetRequest, SaveAssetRequest, OperationResult, WidgetTreeInfo
)
from .config import settings


class UEClient:
    """Unreal Editor HTTP API 클라이언트"""

    def __init__(self, base_url: Optional[str] = None):
        self.base_url = base_url or settings.ue_endpoint
        self.client = httpx.Client(timeout=30.0)

    def _request(self, method: str, path: str, data: Optional[Dict] = None) -> Dict[str, Any]:
        """HTTP 요청 실행"""
        url = f"{self.base_url}{path}"
        
        if method == "GET":
            response = self.client.get(url, params=data)
        elif method == "POST":
            response = self.client.post(url, json=data)
        elif method == "PUT":
            response = self.client.put(url, json=data)
        elif method == "DELETE":
            response = self.client.delete(url, json=data)
        else:
            raise ValueError(f"Unsupported method: {method}")

        response.raise_for_status()
        return response.json()

    def create_widget_blueprint(self, request: CreateWidgetRequest) -> OperationResult:
        """위젯 블루프린트 생성"""
        data = request.model_dump(exclude_none=True)
        # root_widget을 JSON 문자열로 변환
        import json
        data['root_widget_json'] = json.dumps(data.pop('root_widget'))
        
        result = self._request("POST", "/api/umg/create", data)
        return OperationResult(**result)

    def add_widget(self, request: AddWidgetRequest) -> OperationResult:
        """위젯 추가"""
        data = request.model_dump(exclude_none=True)
        import json
        data['widget_spec_json'] = json.dumps(data.pop('widget_spec'))
        
        result = self._request("POST", "/api/umg/add", data)
        return OperationResult(**result)

    def modify_widget(self, request: ModifyWidgetRequest) -> OperationResult:
        """위젯 수정"""
        data = request.model_dump(exclude_none=True)
        result = self._request("PUT", "/api/umg/modify", data)
        return OperationResult(**result)

    def remove_widget(self, request: RemoveWidgetRequest) -> OperationResult:
        """위젯 삭제"""
        data = request.model_dump()
        result = self._request("DELETE", "/api/umg/remove", data)
        return OperationResult(**result)

    def save_asset(self, request: SaveAssetRequest) -> OperationResult:
        """에셋 저장"""
        data = request.model_dump()
        result = self._request("POST", "/api/umg/save", data)
        return OperationResult(**result)

    def get_widget_tree(self, asset_path: str) -> WidgetTreeInfo:
        """위젯 트리 조회"""
        result = self._request("GET", "/api/umg/tree", {"asset_path": asset_path})
        return WidgetTreeInfo(**result)

    def health_check(self) -> bool:
        """서버 상태 확인"""
        try:
            self._request("GET", "/api/umg/health")
            return True
        except Exception:
            return False

    def close(self):
        """클라이언트 종료"""
        self.client.close()
```

---

## 5. MCP 도구 정의

### 5.1 tools.py

```python
"""MCP 도구 정의"""

from mcp.server import Server
from mcp.types import Tool, TextContent
from .models import (
    CreateWidgetRequest, AddWidgetRequest, ModifyWidgetRequest,
    RemoveWidgetRequest, SaveAssetRequest, WidgetSpec
)
from .ue_client import UEClient


def register_tools(server: Server, ue_client: UEClient):
    """MCP 도구 등록"""

    @server.tool()
    async def umg_create_asset(
        asset_path: str,
        root_widget: dict,
        parent_class: str = None,
        overwrite: bool = False
    ) -> list[TextContent]:
        """
        새로운 위젯 블루프린트를 생성합니다.

        Args:
            asset_path: 저장 경로 (예: /Game/UI/MainMenu)
            root_widget: 루트 위젯 JSON 명세
            parent_class: 부모 UserWidget 클래스 경로 (선택)
            overwrite: 기존 에셋 덮어쓰기 여부

        Returns:
            생성 결과 메시지
        """
        try:
            widget_spec = WidgetSpec(**root_widget)
            request = CreateWidgetRequest(
                asset_path=asset_path,
                parent_class=parent_class,
                root_widget=widget_spec,
                overwrite=overwrite
            )
            result = ue_client.create_widget_blueprint(request)

            if result.success:
                msg = f"✅ Widget blueprint created: {result.asset_path}\n"
                msg += f"   Widgets: {result.widget_count}\n"
                if result.warnings:
                    msg += f"   Warnings: {len(result.warnings)}\n"
                    for w in result.warnings[:3]:
                        msg += f"   - {w}\n"
                return [TextContent(type="text", text=msg)]
            else:
                return [TextContent(
                    type="text",
                    text=f"❌ Failed: [{result.error_code}] {result.error_message}"
                )]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]

    @server.tool()
    async def umg_add_widget(
        asset_path: str,
        parent_widget_name: str,
        widget_spec: dict
    ) -> list[TextContent]:
        """
        기존 위젯 블루프린트에 위젯을 추가합니다.

        Args:
            asset_path: 에셋 경로
            parent_widget_name: 부모 위젯 이름
            widget_spec: 추가할 위젯 명세
        """
        try:
            spec = WidgetSpec(**widget_spec)
            request = AddWidgetRequest(
                asset_path=asset_path,
                parent_widget_name=parent_widget_name,
                widget_spec=spec
            )
            result = ue_client.add_widget(request)

            if result.success:
                return [TextContent(
                    type="text",
                    text=f"✅ Widget added to {parent_widget_name}"
                )]
            else:
                return [TextContent(
                    type="text",
                    text=f"❌ Failed: {result.error_message}"
                )]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]

    @server.tool()
    async def umg_modify_widget(
        asset_path: str,
        widget_name: str,
        properties: dict = None,
        slot_properties: dict = None
    ) -> list[TextContent]:
        """
        위젯 속성을 수정합니다.

        Args:
            asset_path: 에셋 경로
            widget_name: 위젯 이름
            properties: 수정할 속성
            slot_properties: 수정할 슬롯 속성
        """
        try:
            request = ModifyWidgetRequest(
                asset_path=asset_path,
                widget_name=widget_name,
                properties=properties,
                slot_properties=slot_properties
            )
            result = ue_client.modify_widget(request)

            if result.success:
                return [TextContent(
                    type="text",
                    text=f"✅ Widget '{widget_name}' modified"
                )]
            else:
                return [TextContent(
                    type="text",
                    text=f"❌ Failed: {result.error_message}"
                )]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]

    @server.tool()
    async def umg_remove_widget(
        asset_path: str,
        widget_name: str
    ) -> list[TextContent]:
        """
        위젯을 삭제합니다.

        Args:
            asset_path: 에셋 경로
            widget_name: 삭제할 위젯 이름
        """
        try:
            request = RemoveWidgetRequest(
                asset_path=asset_path,
                widget_name=widget_name
            )
            result = ue_client.remove_widget(request)

            if result.success:
                return [TextContent(
                    type="text",
                    text=f"✅ Widget '{widget_name}' removed"
                )]
            else:
                return [TextContent(
                    type="text",
                    text=f"❌ Failed: {result.error_message}"
                )]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]

    @server.tool()
    async def umg_save_asset(
        asset_path: str,
        compile: bool = True
    ) -> list[TextContent]:
        """
        위젯 블루프린트를 컴파일하고 저장합니다.

        Args:
            asset_path: 에셋 경로
            compile: 저장 전 컴파일 여부
        """
        try:
            request = SaveAssetRequest(
                asset_path=asset_path,
                compile=compile
            )
            result = ue_client.save_asset(request)

            if result.success:
                return [TextContent(
                    type="text",
                    text=f"✅ Asset saved: {asset_path}"
                )]
            else:
                return [TextContent(
                    type="text",
                    text=f"❌ Failed: {result.error_message}"
                )]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]

    @server.tool()
    async def umg_get_widget_tree(asset_path: str) -> list[TextContent]:
        """
        위젯 블루프린트의 위젯 트리 구조를 조회합니다.

        Args:
            asset_path: 에셋 경로
        """
        try:
            tree = ue_client.get_widget_tree(asset_path)

            msg = f"📊 Widget Tree: {tree.asset_path}\n"
            msg += f"   Root: {tree.root_widget_name}\n"
            msg += f"   Total: {tree.total_widget_count} widgets\n\n"

            for widget in tree.widgets:
                indent = "   "
                msg += f"{indent}• {widget.name} ({widget.type})\n"

            return [TextContent(type="text", text=msg)]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Error: {str(e)}")]
```

---

## 6. MCP 서버

### 6.1 server.py

```python
"""UMG MCP 서버 메인"""

import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from .tools import register_tools
from .ue_client import UEClient
from .config import settings


def create_server() -> Server:
    """MCP 서버 생성"""
    server = Server("umg-mcp-server")
    ue_client = UEClient(settings.ue_endpoint)

    # 도구 등록
    register_tools(server, ue_client)

    return server


async def main():
    """서버 실행"""
    server = create_server()

    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )


if __name__ == "__main__":
    asyncio.run(main())
```

### 6.2 config.py

```python
"""설정"""

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """서버 설정"""
    ue_endpoint: str = "http://localhost:8080"
    log_level: str = "INFO"

    class Config:
        env_prefix = "UMG_MCP_"


settings = Settings()
```

---

## 7. 설치 및 실행

### 7.1 requirements.txt

```
mcp>=1.0.0
pydantic>=2.0.0
pydantic-settings>=2.0.0
httpx>=0.25.0
```

### 7.2 pyproject.toml

```toml
[project]
name = "umg-mcp-server"
version = "1.0.0"
description = "MCP server for UMG asset generation in Unreal Engine"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "httpx>=0.25.0",
]

[project.scripts]
umg-mcp = "umg_mcp_server.server:main"
```

### 7.3 Claude Desktop 설정

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
