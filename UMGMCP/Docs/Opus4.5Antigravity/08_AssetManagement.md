# 08. 에셋 생성/저장 관리
## UMG MCP Asset Generator System

---

## 1. 개요

위젯 블루프린트의 생성, 컴파일, 저장을 담당하는 에셋 관리 시스템입니다.

### 1.1 에셋 처리 흐름

```mermaid
flowchart LR
    A[경로 검증] --> B[패키지 생성]
    B --> C[블루프린트 생성]
    C --> D[위젯 트리 구성]
    D --> E[컴파일]
    E --> F[저장]
    F --> G[Content Browser 갱신]
```

---

## 2. 위젯 블루프린트 생성

### 2.1 팩토리 유틸리티

```cpp
// UMGBlueprintFactory.h
#pragma once

#include "CoreMinimal.h"

class UWidgetBlueprint;
class UUserWidget;

class UMGMCPEDITOR_API FUMGBlueprintFactory
{
public:
    /**
     * 새 위젯 블루프린트 생성
     * @param AssetPath 에셋 경로 (예: /Game/UI/MyWidget)
     * @param ParentClass 부모 클래스 (nullptr = UUserWidget)
     * @param bOverwrite 기존 에셋 덮어쓰기
     * @param OutError 에러 메시지
     * @return 생성된 위젯 블루프린트
     */
    static UWidgetBlueprint* CreateWidgetBlueprint(
        const FString& AssetPath,
        UClass* ParentClass,
        bool bOverwrite,
        FString& OutError);

    /**
     * 블루프린트 컴파일
     * @param Blueprint 대상 블루프린트
     * @param OutError 에러 메시지
     * @return 성공 여부
     */
    static bool CompileBlueprint(UWidgetBlueprint* Blueprint, FString& OutError);

    /**
     * 블루프린트 저장
     * @param Blueprint 대상 블루프린트
     * @param OutError 에러 메시지
     * @return 성공 여부
     */
    static bool SaveBlueprint(UWidgetBlueprint* Blueprint, FString& OutError);

    /**
     * 기존 블루프린트 로드
     * @param AssetPath 에셋 경로
     * @return 로드된 블루프린트 (없으면 nullptr)
     */
    static UWidgetBlueprint* LoadBlueprint(const FString& AssetPath);
};
```

### 2.2 블루프린트 생성 구현

```cpp
UWidgetBlueprint* FUMGBlueprintFactory::CreateWidgetBlueprint(
    const FString& AssetPath,
    UClass* ParentClass,
    bool bOverwrite,
    FString& OutError)
{
    // 1. 경로 파싱
    FString PackagePath, AssetName;
    AssetPath.Split(TEXT("/"), &PackagePath, &AssetName, ESearchCase::IgnoreCase, ESearchDir::FromEnd);

    // 2. 기존 에셋 확인
    FString FullPackagePath = AssetPath;
    UPackage* ExistingPackage = FindPackage(nullptr, *FullPackagePath);
    
    if (ExistingPackage)
    {
        if (!bOverwrite)
        {
            OutError = TEXT("Asset already exists. Set overwrite=true to replace.");
            return nullptr;
        }
        
        // 기존 에셋 언로드
        UWidgetBlueprint* ExistingBP = FindObject<UWidgetBlueprint>(ExistingPackage, *AssetName);
        if (ExistingBP)
        {
            ExistingBP->ClearFlags(RF_Standalone | RF_Public);
            ExistingBP->RemoveFromRoot();
            ExistingBP->MarkAsGarbage();
        }
        ExistingPackage->FullyLoad();
    }

    // 3. 패키지 생성
    UPackage* Package = CreatePackage(*FullPackagePath);
    Package->FullyLoad();

    // 4. 부모 클래스 결정
    if (!ParentClass)
    {
        ParentClass = UUserWidget::StaticClass();
    }

    // 5. 위젯 블루프린트 팩토리 사용
    UWidgetBlueprintFactory* Factory = NewObject<UWidgetBlueprintFactory>();
    Factory->ParentClass = ParentClass;

    UWidgetBlueprint* NewBlueprint = Cast<UWidgetBlueprint>(
        Factory->FactoryCreateNew(
            UWidgetBlueprint::StaticClass(),
            Package,
            FName(*AssetName),
            RF_Public | RF_Standalone,
            nullptr,
            GWarn
        )
    );

    if (!NewBlueprint)
    {
        OutError = TEXT("Failed to create widget blueprint");
        return nullptr;
    }

    // 6. 초기 설정
    NewBlueprint->WidgetTree = NewObject<UWidgetTree>(NewBlueprint, TEXT("WidgetTree"));

    return NewBlueprint;
}
```

---

## 3. 블루프린트 컴파일

```cpp
bool FUMGBlueprintFactory::CompileBlueprint(UWidgetBlueprint* Blueprint, FString& OutError)
{
    if (!Blueprint)
    {
        OutError = TEXT("Blueprint is null");
        return false;
    }

    // 컴파일 옵션
    FCompilerResultsLog Results;
    FKismetEditorUtilities::CompileBlueprint(
        Blueprint,
        EBlueprintCompileOptions::SkipGarbageCollection,
        &Results
    );

    // 결과 확인
    if (Results.NumErrors > 0)
    {
        OutError = FString::Printf(TEXT("Compile failed with %d errors"), Results.NumErrors);
        
        // 첫 번째 에러 메시지 추가
        for (const TSharedRef<FTokenizedMessage>& Message : Results.Messages)
        {
            if (Message->GetSeverity() == EMessageSeverity::Error)
            {
                OutError += TEXT(": ") + Message->ToText().ToString();
                break;
            }
        }
        return false;
    }

    return true;
}
```

---

## 4. 에셋 저장

### 4.1 저장 구현

```cpp
bool FUMGBlueprintFactory::SaveBlueprint(UWidgetBlueprint* Blueprint, FString& OutError)
{
    if (!Blueprint)
    {
        OutError = TEXT("Blueprint is null");
        return false;
    }

    UPackage* Package = Blueprint->GetOutermost();
    if (!Package)
    {
        OutError = TEXT("Package not found");
        return false;
    }

    // 패키지 더티 마킹
    Package->MarkPackageDirty();

    // 저장 경로 계산
    FString PackageFilename = FPackageName::LongPackageNameToFilename(
        Package->GetName(),
        FPackageName::GetAssetPackageExtension()
    );

    // 저장 옵션
    FSavePackageArgs SaveArgs;
    SaveArgs.TopLevelFlags = RF_Public | RF_Standalone;
    SaveArgs.SaveFlags = SAVE_NoError;

    // 저장 실행
    FSavePackageResultStruct Result = UPackage::Save(Package, Blueprint, *PackageFilename, SaveArgs);

    if (Result.Result != ESavePackageResult::Success)
    {
        OutError = FString::Printf(TEXT("Failed to save package: %s"), *PackageFilename);
        return false;
    }

    // Asset Registry 갱신
    FAssetRegistryModule::AssetCreated(Blueprint);

    return true;
}
```

### 4.2 Content Browser 갱신

```cpp
void RefreshContentBrowser(const FString& AssetPath)
{
    // Content Browser 모듈 획득
    FContentBrowserModule& ContentBrowserModule = 
        FModuleManager::LoadModuleChecked<FContentBrowserModule>("ContentBrowser");

    // 경로 동기화
    TArray<FString> FoldersToSync;
    FString FolderPath;
    AssetPath.Split(TEXT("/"), &FolderPath, nullptr, ESearchCase::IgnoreCase, ESearchDir::FromEnd);
    FoldersToSync.Add(FolderPath);

    ContentBrowserModule.Get().SyncBrowserToFolders(FoldersToSync);
}
```

---

## 5. 기존 블루프린트 로드

```cpp
UWidgetBlueprint* FUMGBlueprintFactory::LoadBlueprint(const FString& AssetPath)
{
    // 에셋 로드 시도
    UWidgetBlueprint* Blueprint = LoadObject<UWidgetBlueprint>(nullptr, *AssetPath);
    
    if (!Blueprint)
    {
        // FSoftObjectPath 시도
        FSoftObjectPath SoftPath(AssetPath);
        Blueprint = Cast<UWidgetBlueprint>(SoftPath.TryLoad());
    }

    return Blueprint;
}
```

---

## 6. 서브시스템 통합

### 6.1 CreateWidgetBlueprint

```cpp
FUMGOperationResult UUMGAssetGeneratorSubsystem::CreateWidgetBlueprint(
    const FString& AssetPath,
    const FString& ParentClassName,
    const FString& RootWidgetJson,
    bool bOverwrite)
{
    // 1. 경로 검증
    FString PathError;
    if (!ValidateAssetPath(AssetPath, PathError))
    {
        return FUMGOperationResult::Failure(UMGMCPError::InvalidAssetPath, PathError);
    }

    // 2. 부모 클래스 로드
    UClass* ParentClass = nullptr;
    if (!ParentClassName.IsEmpty())
    {
        ParentClass = LoadClass<UUserWidget>(nullptr, *ParentClassName);
        if (!ParentClass)
        {
            return FUMGOperationResult::Failure(
                UMGMCPError::InvalidParentClass,
                FString::Printf(TEXT("Parent class not found: %s"), *ParentClassName)
            );
        }
    }

    // 3. 블루프린트 생성
    FString CreateError;
    UWidgetBlueprint* Blueprint = FUMGBlueprintFactory::CreateWidgetBlueprint(
        AssetPath, ParentClass, bOverwrite, CreateError);

    if (!Blueprint)
    {
        return FUMGOperationResult::Failure(UMGMCPError::WidgetCreationFailed, CreateError);
    }

    // 4. 위젯 트리 생성
    TreeBuilder->ClearWarnings();
    FString TreeError;
    UWidget* RootWidget = TreeBuilder->ParseJsonToWidgetTree(
        RootWidgetJson, Blueprint->WidgetTree, TreeError);

    if (!RootWidget)
    {
        return FUMGOperationResult::Failure(UMGMCPError::WidgetCreationFailed, TreeError);
    }

    Blueprint->WidgetTree->RootWidget = RootWidget;

    // 5. 컴파일
    FString CompileError;
    if (!FUMGBlueprintFactory::CompileBlueprint(Blueprint, CompileError))
    {
        // 컴파일 실패해도 저장은 시도
        UE_LOG(LogUMGMCP, Warning, TEXT("Compile warning: %s"), *CompileError);
    }

    // 6. 저장
    FString SaveError;
    if (!FUMGBlueprintFactory::SaveBlueprint(Blueprint, SaveError))
    {
        return FUMGOperationResult::Failure(UMGMCPError::SaveFailed, SaveError);
    }

    // 7. 결과 반환
    FUMGOperationResult Result = FUMGOperationResult::Success(
        AssetPath, CountWidgets(Blueprint->WidgetTree));
    Result.Warnings = TreeBuilder->GetWarnings();
    return Result;
}
```

### 6.2 SaveAsset

```cpp
FUMGOperationResult UUMGAssetGeneratorSubsystem::SaveAsset(
    const FString& AssetPath,
    bool bCompile)
{
    // 1. 블루프린트 로드
    UWidgetBlueprint* Blueprint = FUMGBlueprintFactory::LoadBlueprint(AssetPath);
    if (!Blueprint)
    {
        return FUMGOperationResult::Failure(
            UMGMCPError::AssetLoadFailed,
            FString::Printf(TEXT("Failed to load: %s"), *AssetPath)
        );
    }

    // 2. 컴파일 (옵션)
    FString CompileError;
    if (bCompile && !FUMGBlueprintFactory::CompileBlueprint(Blueprint, CompileError))
    {
        return FUMGOperationResult::Failure(UMGMCPError::CompileFailed, CompileError);
    }

    // 3. 저장
    FString SaveError;
    if (!FUMGBlueprintFactory::SaveBlueprint(Blueprint, SaveError))
    {
        return FUMGOperationResult::Failure(UMGMCPError::SaveFailed, SaveError);
    }

    return FUMGOperationResult::Success(AssetPath, CountWidgets(Blueprint->WidgetTree));
}
```

---

## 7. 경로 검증

```cpp
bool UUMGAssetGeneratorSubsystem::ValidateAssetPath(const FString& Path, FString& OutError) const
{
    // /Game/ 또는 /Project/ 시작 필수
    if (!Path.StartsWith(TEXT("/Game/")) && !Path.StartsWith(TEXT("/Project/")))
    {
        OutError = TEXT("Path must start with /Game/ or /Project/");
        return false;
    }

    // 상대 경로 차단
    if (Path.Contains(TEXT("..")))
    {
        OutError = TEXT("Relative path (..) not allowed");
        return false;
    }

    // 유효하지 않은 문자
    static const FString InvalidChars = TEXT("\\:*?\"<>|");
    for (TCHAR Char : InvalidChars)
    {
        if (Path.Contains(FString(1, &Char)))
        {
            OutError = FString::Printf(TEXT("Invalid character: %c"), Char);
            return false;
        }
    }

    // 빈 이름 검사
    FString FileName;
    Path.Split(TEXT("/"), nullptr, &FileName, ESearchCase::IgnoreCase, ESearchDir::FromEnd);
    if (FileName.IsEmpty())
    {
        OutError = TEXT("Asset name cannot be empty");
        return false;
    }

    return true;
}
```

---

## 8. 위젯 카운트 헬퍼

```cpp
int32 CountWidgets(UWidgetTree* WidgetTree)
{
    if (!WidgetTree || !WidgetTree->RootWidget)
    {
        return 0;
    }

    int32 Count = 0;
    TArray<UWidget*> Stack;
    Stack.Push(WidgetTree->RootWidget);

    while (Stack.Num() > 0)
    {
        UWidget* Current = Stack.Pop();
        Count++;

        if (UPanelWidget* Panel = Cast<UPanelWidget>(Current))
        {
            for (int32 i = 0; i < Panel->GetChildrenCount(); i++)
            {
                Stack.Push(Panel->GetChildAt(i));
            }
        }
    }

    return Count;
}
```
