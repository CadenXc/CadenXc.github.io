---
title: ProceduralMesh转StaticMesh
date: 2025-09-06 00:01:05
tags:
 - UE
 - C++
 - 程序化网格体
 - ProceduralMeshComponent
description:
  - 在运行时ProcduralMesh转换为StaticMesh。我使用的Ue引擎版本是4.26.2。
---

在ProceduralMeshComponent的细节面板上，有一个按键convert to static mesh，可以把你当前的PMC转换为静态模型。这个function的位置在`ProceduralMeshComponentDetails.cpp`，使用的是`FStaticMeshSourceModel`来构建。但是这个函数无法在运行时使用，因为有`FStaticMeshSourceModel& AddSourceModel();`ve这个函数有`WITH_EDITORONLY_DATA`的宏限制。

如果想在运行时把程序化模型转换为静态模型该怎么办呢？这就要使用FMeshDescription了。下面贴一段代码。

```c++
UStaticMesh* AProceduralMeshActor::ConvertProceduralMeshToStaticMesh()

{

// 1. 验证输入参数

**if** (!ProceduralMeshComponent)

{

UE_LOG(LogTemp, Warning,

TEXT("ConvertProceduralMeshToStaticMesh: PMC为空。"));

**return** nullptr;

}

**const** int32 NumSections = ProceduralMeshComponent->GetNumSections();

**if** (NumSections == 0)

{

UE_LOG(LogTemp, Warning,

TEXT("ConvertProceduralMeshToStaticMesh: PMC没有网格段。"));

**return** nullptr;

}

// 2. 创建一个临时的、只存在于内存中的UStaticMesh对象

UStaticMesh* StaticMesh = NewObject<UStaticMesh>(

GetTransientPackage(), NAME_None, RF_Public | RF_Standalone);

**if** (!StaticMesh)

{

UE_LOG(LogTemp, Error, TEXT("无法创建临时的静态网格对象。"));

**return** nullptr;

}

// 3. 创建并设置 FMeshDescription

FMeshDescription MeshDescription;

FStaticMeshAttributes Attributes(MeshDescription);

Attributes.Register(); // 这会注册包括材质槽名称在内的所有必要属性

// 获取对多边形组材质槽名称属性的引用，以便后续设置

// 这是将几何体与材质关联的关键

TPolygonGroupAttributesRef<FName> PolygonGroupMaterialSlotNames =

Attributes.GetPolygonGroupMaterialSlotNames();

FMeshDescriptionBuilder MeshDescBuilder;

MeshDescBuilder.SetMeshDescription(&MeshDescription);

MeshDescBuilder.EnablePolyGroups();

MeshDescBuilder.SetNumUVLayers(1);

// 4. 从PMC数据填充 FMeshDescription

TMap<FVector, FVertexID> VertexMap;

**for** (int32 SectionIdx = 0; SectionIdx < NumSections; ++SectionIdx)

{

FProcMeshSection* SectionData =

ProceduralMeshComponent->GetProcMeshSection(SectionIdx);

**if** (!SectionData || SectionData->ProcVertexBuffer.Num() < 3 ||

SectionData->ProcIndexBuffer.Num() < 3)

{

UE_LOG(LogTemp, Warning,

TEXT("ConvertProceduralMeshToStaticMesh - 跳过段 %d: 数据不完整"),

SectionIdx);

**continue**;

}

// --- 开始材质设置修正 ---

// a. 获取本段的材质

UMaterialInterface* SectionMaterial = nullptr;

**if** (SectionIdx < StaticSectionMaterials.Num() &&

StaticSectionMaterials[SectionIdx] &&

::IsValid(StaticSectionMaterials[SectionIdx]))

{

SectionMaterial = StaticSectionMaterials[SectionIdx];

}

**else** **if** (StaticMeshMaterial && ::IsValid(StaticMeshMaterial))

{

SectionMaterial = StaticMeshMaterial;

}

**else**

{

SectionMaterial = ProceduralMeshComponent->GetMaterial(SectionIdx);

**if** (!SectionMaterial || !::IsValid(SectionMaterial))

{

SectionMaterial = UMaterial::GetDefaultMaterial(MD_Surface);

}

}

// b. 为每个材质段创建一个唯一的槽名称

**const** FName MaterialSlotName =

FName(*FString::Printf(TEXT("MaterialSlot_%d"), SectionIdx));

// c. 将材质和槽名称添加到StaticMesh的材质列表中

// 这会在最终的UStaticMesh上创建对应的材质槽

StaticMesh->StaticMaterials.Add(

FStaticMaterial(SectionMaterial, MaterialSlotName));

// d. 在MeshDescription中为这个材质段创建一个新的多边形组

**const** FPolygonGroupID PolygonGroup = MeshDescBuilder.AppendPolygonGroup();

// e. 将多边形组与我们刚刚创建的材质槽名称关联起来

// 这是最关键的一步！

PolygonGroupMaterialSlotNames.Set(PolygonGroup, MaterialSlotName);

// --- 结束材质设置修正 ---

TArray<FVertexInstanceID> VertexInstanceIDs;

**for** (**const** FProcMeshVertex& ProcVertex : SectionData->ProcVertexBuffer)

{

FVertexID VertexID;

**if** (FVertexID* FoundID = VertexMap.Find(ProcVertex.Position))

{

VertexID = *FoundID;

}

**else**

{

VertexID = MeshDescBuilder.AppendVertex(ProcVertex.Position);

VertexMap.Add(ProcVertex.Position, VertexID);

}

FVertexInstanceID InstanceID = MeshDescBuilder.AppendInstance(VertexID);

MeshDescBuilder.SetInstanceNormal(InstanceID, ProcVertex.Normal);

MeshDescBuilder.SetInstanceUV(InstanceID, ProcVertex.UV0, 0);

MeshDescBuilder.SetInstanceColor(InstanceID, FVector4(ProcVertex.Color));

VertexInstanceIDs.Add(InstanceID);

}

**for** (int32 i = 0; i < SectionData->ProcIndexBuffer.Num(); i += 3)

{

**if** (i + 2 >= SectionData->ProcIndexBuffer.Num())

{

**break**;

}

int32 Index1 = SectionData->ProcIndexBuffer[i + 0];

int32 Index2 = SectionData->ProcIndexBuffer[i + 1];

int32 Index3 = SectionData->ProcIndexBuffer[i + 2];

**if** (Index1 >= VertexInstanceIDs.Num() ||

Index2 >= VertexInstanceIDs.Num() ||

Index3 >= VertexInstanceIDs.Num())

{

UE_LOG(

LogTemp, Error,

TEXT("ConvertProceduralMeshToStaticMesh - 段 %d: 三角形索引越界"),

SectionIdx);

**continue**;

}

FVertexInstanceID V1 = VertexInstanceIDs[Index1];

FVertexInstanceID V2 = VertexInstanceIDs[Index2];

FVertexInstanceID V3 = VertexInstanceIDs[Index3];

// 将三角形添加到指定了材质信息的PolygonGroup中

MeshDescBuilder.AppendTriangle(V1, V2, V3, PolygonGroup);

}

}

**if** (MeshDescription.Vertices().Num() == 0)

{

UE_LOG(LogTemp, Warning,

TEXT("无法从PMC创建静态网格，因为没有有效的顶点数据。"));

**return** nullptr;

}

// 7. 使用 FMeshDescription 构建 Static Mesh

TArray<**const** FMeshDescription*> MeshDescPtrs;

MeshDescPtrs.Emplace(&MeshDescription);

UStaticMesh::FBuildMeshDescriptionsParams BuildParams;

BuildParams.bBuildSimpleCollision = bGenerateCollision;

StaticMesh->BuildFromMeshDescriptions(MeshDescPtrs, BuildParams);

// ... (后续的碰撞体设置和PostEditChange保持不变)

**if** (bGenerateCollision && ProceduralMeshComponent->ProcMeshBodySetup)

{

StaticMesh->CreateBodySetup();

UBodySetup* NewBodySetup = StaticMesh->BodySetup;

**if** (NewBodySetup)

{

NewBodySetup->AggGeom.ConvexElems =

ProceduralMeshComponent->ProcMeshBodySetup->AggGeom.ConvexElems;

NewBodySetup->bGenerateMirroredCollision = false;

NewBodySetup->bDoubleSidedGeometry = true;

NewBodySetup->CollisionTraceFlag = CTF_UseDefault;

NewBodySetup->CreatePhysicsMeshes();

}

}

// StaticMesh->PostEditChange();

**return** StaticMesh;

}
```
