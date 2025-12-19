# 아이템 아이콘 생성 및 분할 워크플로우

이 문서는 프로젝트용 아이템 아이콘을 생성하고 처리하는 절차를 정리한 문서입니다.

## 1. 이미지 생성 (프롬프트 가이드)

AI 이미지 생성기에게 새로운 아이템 아이콘을 요청할 때는 다음 사양을 사용하십시오:

*   **형식:** 1024x1024 픽셀, 4x4 격자 (총 16개 아이콘).
*   **배경:** **완전한 검은색 (#000000)**.
    *   *이유:* 투명 배경으로 생성하면 품질이 저하되거나 아티팩트가 남는 경우가 많습니다. 검은 배경은 AI 도구를 사용해 깔끔하게 제거하기 좋습니다.
*   **스타일:** 깔끔하고 다채로운 색감의 카툰/벡터풍 게임 에셋. `Images/` 폴더 내 기존 스타일과 일치시킬 것.
*   **중요 제약 사항:**
    *   **아이템 내부에 검은색 사용 금지** (깔끔한 배경 제거를 위해 필수).
    *   아이템은 격자 셀 중앙에 위치해야 하며 서로 겹치지 않아야 합니다.
*   **아이템 목록:** 격자에 들어갈 16개 아이템을 명확히 지정하십시오.
    *   1행: [아이템 1], [아이템 2], [아이템 3], [아이템 4]
    *   2행: [아이템 5], [아이템 6], [아이템 7], [아이템 8]
    *   3행: [아이템 9], [아이템 10], [아이템 11], [아이템 12]
    *   4행: [아이템 13], [아이템 14], [아이템 15], [아이템 16]

## 2. 후처리 및 분할

이미지 생성 후, Python 스크립트와 `rembg`를 사용하여 처리합니다.

### 사전 준비
*   Python 설치 필요.
*   라이브러리: `Pillow`, `rembg`.
    ```bash
    pip install pillow rembg[gpu]  # NVIDIA GPU가 없으면 rembg[cpu]
    ```

### 절차
1.  **백업:** 생성된 원본 이미지를 `d:\github\miyakov\Images\` 경로에 접미사(예: `_original.png`)를 붙여 저장합니다.
2.  **분할 및 배경 제거:**
    *   1024x1024 이미지를 16개의 256x256 타일로 자릅니다.
    *   `rembg`를 사용하여 각 타일에서 검은색 배경을 제거합니다.
3.  **저장:** 각 타일을 생성 단계에서 지정한 아이템 이름으로 PNG 파일로 저장합니다.
    *   **대상 폴더:** `d:\github\miyakov\Images\GeneratedItems\`

### Python 스크립트 템플릿

처리를 위해 다음 스크립트 구조를 사용하십시오:

```python
import os
os.environ["NUMBA_DISABLE_JIT"] = "1" # numba 캐시 문제 방지

from PIL import Image
from rembg import remove

def process_icons(image_path, output_dir, item_names):
    if not os.path.exists(output_dir):
        os.makedirs(output_dir)

    img = Image.open(image_path)
    width, height = img.size
    print(f"이미지 크기: {width}x{height} (예상: 1024x1024)")
    
    rows, cols = 4, 4
    item_w = width // cols
    item_h = height // rows

    for i, name in enumerate(item_names):
        r = i // cols
        c = i % cols
        
        left = c * item_w
        upper = r * item_h
        right = left + item_w
        lower = upper + item_h
        
        crop = img.crop((left, upper, right, lower))
        
        print(f"처리 중: {name}...")
        # AI 모델(U2-Net)을 사용하여 배경 제거
        processed = remove(crop)
        
        # 저장
        save_path = os.path.join(output_dir, f"{name}.png")
        processed.save(save_path, "PNG")
        print(f"저장됨: {save_path}")

if __name__ == "__main__":
    # --- 설정 ---
    # 생성된 4x4 격자 이미지 경로
    INPUT_IMAGE = r"생성된_이미지_경로.png"
    
    # 출력 폴더
    OUTPUT_DIR = r"d:\github\miyakov\Images\GeneratedItems"
    
    # 격자에 대응하는 16개 아이템 이름 목록 (행 우선 순서)
    ITEMS = [
        "아이템01", "아이템02", "아이템03", "아이템04",
        "아이템05", "아이템06", "아이템07", "아이템08",
        "아이템09", "아이템10", "아이템11", "아이템12",
        "아이템13", "아이템14", "아이템15", "아이템16"
    ]
    # ---------------------
    
    process_icons(INPUT_IMAGE, OUTPUT_DIR, ITEMS)
```
