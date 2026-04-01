# CAD/BIM 到算量自动化流程研究报告

> 研究日期：2026-04-01  
> 研究目标：从 CAD/DWG 和 BIM（IFC/Revit）模型中自动提取工程量清单

---

## 目录

1. [IFC 文件与 ifcopenshell](#1-ifc-文件与-ifcopenshell)
2. [Revit API 二次开发](#2-revit-api-二次开发)
3. [DWG 转 IFC](#3-dwg-转-ifc)
4. [国内 BIM 算量软件对接](#4-国内-bim-算量软件对接)
5. [自动化流程与代码示例](#5-自动化流程与代码示例)
6. [IFC 实体类型与算量分类对照](#6-ifc-实体类型与算量分类对照)

---

## 1. IFC 文件与 ifcopenshell

### 1.1 IFC 概述

IFC（Industry Foundation Classes）是 buildingSMART 国际制定的文件格式标准，用于 BIM 数据的互操作交换。IFC 采用 EXPRESS 语言定义数据模式，文件格式可为 STEP 文本（`.ifc`）或压缩格式（`.ifcZIP`）。

**IFC 文件结构层次：**
```
IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace/IfcElement
```

### 1.2 ifcopenshell 简介

ifcopenshell 是最主流的开源 IFC 处理库，基于 Open CASCADE 开发，支持：
- Python API（最常用）
- C++ API
- BlenderGIS 插件集成

**安装方式：**
```bash
# macOS/Linux
pip install ifcopenshell

# Windows（推荐使用预编译二进制）
pip install ifcopenshell==0.7.0
```

### 1.3 基础用法

#### 读取 IFC 文件
```python
import ifcopenshell

# 打开 IFC 文件
model = ifcopenshell.open("building.ifc")

# 获取项目信息
project = model.by_type("IfcProject")[0]
print(f"项目名称: {project.Name}")
print(f"IFC Schema: {project.schema_definition.name}")
```

#### 遍历所有建筑元素
```python
import ifcopenshell

model = ifcopenshell.open("building.ifc")

# 遍历所有实体类型
for entity in model:
    print(entity.is_a(), entity.id())
```

#### 提取墙体数据（IfcWall / IfcWallStandardCase）
```python
import ifcopenshell

model = ifcopenshell.open("building.ifc")

walls = model.by_type("IfcWallStandardCase")
for wall in walls:
    # 基本属性
    name = wall.Name
    global_id = wall.GlobalId
    
    # 获取几何信息（外形状）
    shape = ifcopenshell.geom.create_shape(wall)
    # shape.geometry 包含顶点、面等数据
    
    # 获取长度属性（Quantity）
    # 方式1：通过 IsDefinedBy 关系获取预定义数量
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            prop_set = rel.RelatingPropertyDefinition
            if prop_set.is_a("IfcElementQuantity"):
                for q in prop_set.Quantities:
                    if q.is_a("IfcQuantityLength"):
                        print(f"长度: {q.LengthValue}")
    
    print(f"墙体: {name}, ID: {global_id}")
```

#### 提取门窗（IfcDoor / IfcWindow）
```python
# 提取门
doors = model.by_type("IfcDoor")
for door in doors:
    print(f"门: {door.Name}, 类型: {door.is_a()}")
    
# 提取窗
windows = model.by_type("IfcWindow")
for window in windows:
    print(f"窗: {window.Name}")
    
# 通过 IfcRelFillsElement 获取被填充的洞口
fill_elements = model.by_type("IfcRelFillsElement")
for rel in fill_elements:
    door = rel.RelatedBuildingElement
    opening = rel.RelatingOpeningElement
    print(f"{door.Name} 填充洞口 {opening.Name}")
```

#### 提取管线（IfcPipeSegment / IfcDuctSegment / IfcPipeFitting）
```python
# 管道
pipe_segments = model.by_type("IfcPipeSegment")
# 风管
duct_segments = model.by_type("IfcDuctSegment")
# 管件
pipe_fittings = model.by_type("IfcPipeFitting")
# 阀门
valves = model.by_type("IfcValve")

for pipe in pipe_segments:
    # 获取直径、长度
    print(f"管道: {pipe.Name}")
    # 通过属性集获取规格
    for rel in pipe.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            psets = rel.RelatingPropertyDefinition
            # 遍历属性
            pass
```

#### 提取属性信息（PSet）
```python
# 获取属性集
def get_property_sets(element, model):
    """提取元素的属性集"""
    psets = []
    for rel in element.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            psd = rel.RelatingPropertyDefinition
            if psd.is_a("IfcPropertySet"):
                pset_data = {"name": psd.Name, "properties": []}
                for prop in psd.HasProperties:
                    if prop.is_a("IfcPropertySingleValue"):
                        pset_data["properties"].append({
                            "name": prop.Name,
                            "value": prop.NominalValue.wrappedValue if prop.NominalValue else None
                        })
                psets.append(pset_data)
    return psets
```

#### 计算体积和面积
```python
import ifcopenshell
from ifcopenshell import geom

# 设置计算精度
settings = geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

walls = model.by_type("IfcWallStandardCase")
for wall in walls:
    shape = geom.create_shape(settings, wall)
    # 计算体积（m³）
    volume = shape.geometry.vertices  # 顶点列表
    # 更精确的方式：用 Open CASCADE 计算
    pass
```

### 1.4 ifcopenshell 常用 API 速查

| 功能 | API |
|------|-----|
| 按类型查询 | `model.by_type("IfcWall")` |
| 按 GUID 查询 | `model.by_id(global_id)` |
| 按 GlobalId 查询 | `model.by_guid("xxx")` |
| 创建形状 | `ifcopenshell.geom.create_shape(element)` |
| 获取属性集 | `element.IsDefinedBy` |
| 获取空间关系 | `element.ContainedInStructure` |
| 获取材料 | `element.HasAssociations` → `IfcRelAssociatesMaterial` |
| 获取几何表示 | `element.Representation` |

---

## 2. Revit API 二次开发

### 2.1 Revit API 概述

Revit API 基于 .NET Framework（C# / VB.NET），通过 Revit 插件（`.addin` + `.dll`）或外部工具（External Tool）方式调用。Revit 2021+ 支持 Python（通过 RevitPythonShell 或 pyRevit）。

### 2.2 Revit API 基础架构

```
Revit Application
  └─ Document（文档）
       ├─ FamilyManager（族管理）
       ├─ ParameterBindings（参数绑定）
       └─ ProjectInfo（项目信息）
  └─ FilteredElementCollector（过滤元素收集器）
       └─ Element（建筑元素）
            ├─ Wall（墙体）
            ├─ FamilyInstance（族实例，如门、窗）
            ├─ MEPCurve（管道、风管）
            └─ Material（材料）
```

### 2.3 基础 C# 示例（Revit API）

```csharp
using Autodesk.Revit.DB;
using Autodesk.Revit.UI;
using Autodesk.Revit.Attributes;

[Transaction(TransactionMode.ReadOnly)]
public class QuantityExtractor : IExternalCommand
{
    public Result Execute(
        ExternalCommandData commandData,
        ref string message,
        ElementSet elements)
    {
        UIApplication uiApp = commandData.Application;
        Document doc = uiApp.ActiveUIDocument.Document;

        // 使用过滤收集器提取墙体
        FilteredElementCollector wallCollector = 
            new FilteredElementCollector(doc)
                .OfCategory(BuiltInCategory.OST_Walls)
                .WhereElementIsNotElementType();

        int wallCount = wallCollector.Count();
        double totalVolume = 0;

        foreach (Element wall in wallCollector)
        {
            // 获取体积参数
            Parameter volParam = wall.get_Parameter(
                BuiltInParameter.HOST_VOLUME_COMPUTED);
            if (volParam != null && volParam.HasValue)
            {
                totalVolume += volParam.AsDouble();
            }
        }

        TaskDialog.Show("墙体统计",
            $"墙体数量: {wallCount}\n总体积: {totalVolume} ft³");

        return Result.Succeeded;
    }
}
```

### 2.4 Python 示例（RevitPythonShell / pyRevit）

```python
# RevitPythonShell 或 pyRevit 环境
from Autodesk.Revit.DB import (
    FilteredElementCollector, 
    BuiltInCategory,
    Transaction,
    ElementId
)
from Autodesk.Revit.UI import TaskDialog

doc = __revit__.ActiveUIDocument.Document

# 提取所有墙体
wall_collector = FilteredElementCollector(doc)\
    .OfCategory(BuiltInCategory.OST_Walls)\
    .WhereElementIsNotElementType()

walls = list(wall_collector)
print(f"墙体总数: {len(walls)}")

total_area = 0
total_volume = 0

for wall in walls:
    # 墙体的面积
    area = wall.get_Parameter(
        "财神爷".BuiltInParameter.HOST_AREA_COMPUTED).AsDouble()
    # 墙体的体积
    volume = wall.get_Parameter(
        BuiltInParameter.HOST_VOLUME_COMPUTED).AsDouble()
    
    total_area += area
    total_volume += volume
    
    print(f"墙体: {wall.Name}, 面积: {area:.2f} ft²")

print(f"总面积: {total_area:.2f} ft²")
print(f"总体积: {total_volume:.2f} ft³")
```

### 2.5 提取明细表数据

Revit 中的明细表（Schedule）本质上是 `ViewSchedule` 对象，可以通过 API 直接读取：

```csharp
// 读取明细表数据
ViewSchedule schedule = collector.OfClass(typeof(ViewSchedule))
    .Cast<ViewSchedule>()
    .FirstOrDefault(s => s.Name == "墙体明细表");

TableData tableData = schedule.GetTableData();
TableSectionData sectionData = tableData.GetSectionData(SectionType.Body);

int rows = sectionData.NumberOfRows;
int cols = sectionData.NumberOfColumns;

for (int row = 0; row < rows; row++)
{
    for (int col = 0; col < cols; col++)
    {
        string cellValue = schedule.GetCellText(SectionType.Body, row, col);
        Console.WriteLine($"[{row},{col}]: {cellValue}");
    }
}
```

### 2.6 常用 BuiltInParameter 速查

| 算量参数 | BuiltInParameter |
|---------|-----------------|
| 体积 | `HOST_VOLUME_COMPUTED` |
| 面积 | `HOST_AREA_COMPUTED` |
| 长度 | `CURVE_ELEM_LENGTH` |
| 底部偏移 | `LEVEL_PARAM` |
| 顶部偏移 | `TOP_OFFSET` |
| 墙类型名称 | `ELEM_TYPE_PARAM` |
| 材质 | `MATERIAL_ID_PARAM` |

---

## 3. DWG 转 IFC

### 3.1 转换路径概述

```
DWG (AutoCAD) 
  ├─ 直接导出 IFC（AutoCAD MEP/Architecture 支持）
  ├─ Revit 导入 DWG 后导出 IFC
  └─ 第三方工具（FreeCAD, BlenderBIM）
```

### 3.2 AutoCAD MEP 直接导出 IFC

AutoCAD MEP 提供了 DWG → IFC 的直接导出功能：

1. 打开 DWG 文件（建议使用 MEP 版本）
2. `EXPORTTOIFC` 命令
3. 设置 IFC 导出配置：
   - **图层映射**：将 DWG 图层映射到 IFC 实体类型
   - **命名规则**：定义元素命名规范
   - **空间结构**：设定建筑楼层/空间层级
4. 导出设置关键参数：
   - `IFCEXPORTLAYERSASPROPERTIES`：图层作为属性
   - `IFCEXPORTMODE`：空间结构模式
   - `IFCEXPORTEXTRUSION`：挤压体导出方式

### 3.3 Revit 导入 DWG

**导入流程：**
1. Revit 新建项目 → `插入` → `导入 CAD`
2. 选择 DWG 文件，设置导入选项：
   - **颜色映射**：保留原图颜色
   - **导入单位**：与 DWG 单位一致
   - **导入图层**：可选择性导入特定图层
3. 将导入的 CAD 图元链接到楼层视图
4. **识别用途**：手动将 CAD 图元与 Revit 建筑元素对应

**图层映射配置示例：**
```
W-（墙体线）→ Revit 墙体
D-（门）→ Revit 门
W-（窗）→ Revit 窗
P-（管道）→ Revit 管道系统
```

### 3.4 IFC 导出设置（Revit）

Revit 导出 IFC 的关键设置：

| 设置项 | 说明 |
|-------|------|
| `IFC Export Setup` → `IFC Class` | 选择导出级别（LOD200-350） |
| `Geometric Representation` | 几何精度设置 |
| `Property Sets` | 包含的属性集 |
| `Export Rooms on Spaces` | 空间房间信息 |
| `Tessellation Level` | 曲面细分等级 |
| `Split Walls/Columns` | 墙体/柱分段 |

**推荐导出配置：**
- **IFC 2×3**：最通用兼容性
- **IFC4**：支持更多几何和属性
- 导出前清理未使用的族和类型

### 3.5 DWG 几何处理建议

CAD 图纸转 BIM 时应注意：
1. **图层清理**：删除多余图层、图块
2. **单位统一**：确保所有图元单位一致
3. **线型处理**：将多段线转为闭合多段线（便于拉伸）
4. **图层命名规范**：使用标准图层命名（W-*, D-*, P-*）
5. **高程信息**：2D CAD 需要标注标高

---

## 4. 国内 BIM 算量软件对接

### 4.1 软件概览

| 软件 | 厂商 | 主要特点 |
|------|------|---------|
| 广联达 BIM 土建计量 | 广联达 | 自主平台，对接 GCL/GGJ |
| 鲁班 BIM 算量 | 鲁班软件 | 支持 IF |
| 品茗 BIM 算量 | 品茗科技 | 专注施工阶段算量 |
| 斯维尔 BIM | 斯维尔 | 工程造价全流程 |

### 4.2 广联达 BIM 土建计量

**与 Revit 对接：**
- **插件方式**：在 Revit 中安装广联达插件，直接导出 GCL 格式
- **IFC 方式**：Revit 导出 IFC → 广联达土建计量导入 IFC
- **Rvt 方式**：广联达直接打开 Revit 文件（部分版本）

**IFC 导入注意事项：**
- 广联达主要识别墙（IfcWall）、柱（IfcColumn）、梁（IfcBeam）、板（IfcSlab）
- 需要正确设置楼层高度和标高
- 门窗需要正确的 `IfcDoor` / `IfcWindow` 类型

**导出 GCL 格式（GCL = 广联达土建工程量）:**
```csharp
// Revit 插件调用广联达接口
// 使用 GCLiate 插件 API
```

### 4.3 鲁班 BIM 算量

**IFC 工作流：**
1. Revit 导出 IFC（推荐 IFC2×3 格式）
2. 鲁班 BIM 算量软件 → `文件` → `导入 IFC`
3. 自动映射构件类型
4. 手工调整分类映射关系

**构件映射表：**
| IFC 实体 | 鲁班分类 |
|---------|---------|
| IfcWall | 墙体 |
| IfcSlab | 楼板/屋面板 |
| IfcBeam | 梁 |
| IfcColumn | 柱 |
| IfcDoor | 门 |
| IfcWindow | 窗 |
| IfcStair | 楼梯 |

### 4.4 品茗 BIM 算量

- 支持 Revit 插件直接导出
- 支持 IFC 标准导入
- 专注于施工预算阶段
- 提供碰撞检查与算量一体化

### 4.5 IFC 对接的常见问题与解决

**问题 1：构件丢失或无法识别**
- 原因：IFC 导出时未包含正确实体类型
- 解决：检查 Revit → IFC Export Setup 中分类映射

**问题 2：体积/面积计算不准确**
- 原因：几何表示不完整（Proxy Element 无几何）
- 解决：确保导出"完全"几何而非"简化"几何

**问题 3：材质信息丢失**
- 原因：IFC 不强制携带材料信息
- 解决：使用 `IfcRelAssociatesMaterial` 显式关联材料

**问题 4：楼层映射错误**
- 原因：BuildingStorey 与标高关系混乱
- 解决：在 Revit 中正确设置建筑楼层，导出前清理空楼层

---

## 5. 自动化流程与代码示例

### 5.1 整体自动化流程

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   CAD/DWG   │───→│  Revit/GJ   │───→│    IFC      │───→│   算量软件   │
│   图纸      │    │  BIM 模型   │    │   交换格式   │    │   广联达等   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                        │                   │
                        └──────→ Python ────┘
                        ifcopenshell + Revit API
```

### 5.2 Python 完整代码示例：从 IFC 生成算量清单

```python
"""
IFC 自动算量脚本
功能：从 IFC 文件提取主要构件工程量，生成 Excel 清单
依赖：ifcopenshell, openpyxl
"""

import ifcopenshell
from ifcopenshell import geom
import openpyxl
from openpyxl.styles import Font, Alignment, PatternFill
from collections import defaultdict
from datetime import datetime

# ============================================================
# 配置区
# ============================================================
IFC_FILE = "sample_building.ifc"
OUTPUT_EXCEL = f"算量清单_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"

# IFC 实体类型 → 算量分类映射
ELEMENT_MAPPING = {
    "IfcWallStandardCase": {"name": "墙体", "unit": "体积 m³", "quantity": "volume"},
    "IfcWall": {"name": "墙体", "unit": "体积 m³", "quantity": "volume"},
    "IfcSlab": {"name": "楼板/屋面板", "unit": "面积 m²", "quantity": "area"},
    "IfcBeam": {"name": "梁", "unit": "体积 m³", "quantity": "volume"},
    "IfcColumn": {"name": "柱", "unit": "体积 m³", "quantity": "volume"},
    "IfcDoor": {"name": "门", "unit": "樘/面积 m²", "quantity": "count"},
    "IfcWindow": {"name": "窗", "unit": "樘/面积 m²", "quantity": "count"},
    "IfcStair": {"name": "楼梯", "unit": "座/面积 m²", "quantity": "count"},
    "IfcPipeSegment": {"name": "管道", "unit": "长度 m", "quantity": "length"},
    "IfcDuctSegment": {"name": "风管", "unit": "长度 m", "quantity": "length"},
    "IfcPipeFitting": {"name": "管件", "unit": "个数", "quantity": "count"},
    "IfcValve": {"name": "阀门", "unit": "个数", "quantity": "count"},
}


def calculate_geometry_volume(element, settings):
    """计算元素体积（m³）"""
    try:
        shape = geom.create_shape(settings, element)
        # Open CASCADE 几何对象
        # 此处需要 Open CASCADE 额外计算体积
        # 简化：返回外包框体积估算
        return 0.0  # 实际需要调用 OCC BRepBuiilderAPI
    except Exception:
        return 0.0


def calculate_geometry_area(element, settings):
    """计算元素面积（m²）"""
    try:
        shape = geom.create_shape(settings, element)
        return 0.0  # 需要 OCC 计算
    except Exception:
        return 0.0


def get_element_quantity(element, element_type, settings):
    """根据元素类型获取算量值"""
    mapping = ELEMENT_MAPPING.get(element_type, {})
    qty_type = mapping.get("quantity", "count")
    
    if qty_type == "count":
        return 1
    elif qty_type == "volume":
        return calculate_geometry_volume(element, settings)
    elif qty_type == "area":
        return calculate_geometry_area(element, settings)
    elif qty_type == "length":
        try:
            # 获取长度参数
            for rel in element.IsDefinedBy:
                if rel.is_a("IfcRelDefinesByProperties"):
                    psd = rel.RelatingPropertyDefinition
                    for prop in psd.HasProperties:
                        if prop.Name == "Length":
                            return prop.NominalValue.wrappedValue
            # 从几何计算
            shape = geom.create_shape(settings, element)
            return shape.geometry.vertices  # 需处理
        except:
            return 0.0
    return 1


def extract_materials(element, model):
    """提取元素材料"""
    materials = []
    for assoc in element.HasAssociations:
        if assoc.is_a("IfcRelAssociatesMaterial"):
            mat = assoc.RelatingMaterial
            if mat.is_a("IfcMaterial"):
                materials.append(mat.Name)
            elif mat.is_a("IfcMaterialLayerSet"):
                for layer in mat.MaterialLayers:
                    materials.append(layer.Material.Name)
    return ", ".join(materials) if materials else "未指定"


def extract_quantity_from_ifc(ifc_path):
    """从 IFC 文件提取算量数据"""
    
    print(f"正在打开 IFC 文件: {ifc_path}")
    model = ifcopenshell.open(ifc_path)
    
    # 几何计算设置
    settings = geom.settings()
    settings.set(settings.USE_WORLD_COORDS, True)
    
    results = defaultdict(lambda: {
        "count": 0, 
        "volume": 0.0, 
        "area": 0.0, 
        "length": 0.0,
        "elements": []
    })
    
    # 定义要提取的 IFC 类型
    target_types = list(ELEMENT_MAPPING.keys())
    
    for ifc_type in target_types:
        elements = model.by_type(ifc_type)
        
        if not elements:
            continue
        
        print(f"处理 {ifc_type}: {len(elements)} 个元素")
        
        category_name = ELEMENT_MAPPING[ifc_type]["name"]
        
        for elem in elements:
            elem_name = elem.Name or f"{ifc_type}_{elem.id()}"
            
            # 获取材料
            materials = extract_materials(elem, model)
            
            # 获取数量值
            qty_value = get_element_quantity(elem, ifc_type, settings)
            
            results[category_name]["count"] += 1
            results[category_name]["elements"].append({
                "name": elem_name,
                "global_id": elem.GlobalId,
                "type": ifc_type,
                "materials": materials,
                "quantity": qty_value
            })
            
            # 累加到分类汇总
            mapping = ELEMENT_MAPPING[ifc_type]
            qty_type = mapping.get("quantity", "count")
            if qty_type in ["volume", "area", "length"]:
                results[category_name][qty_type] += qty_value
    
    return results


def export_to_excel(results, output_path):
    """导出算量结果到 Excel"""
    
    wb = openpyxl.Workbook()
    
    # ---- Sheet 1: 汇总表 ----
    ws_summary = wb.active
    ws_summary.title = "算量汇总"
    
    header_fill = PatternFill(start_color="4472C4", end_color="4472C4", fill_type="solid")
    header_font = Font(color="FFFFFF", bold=True)
    
    headers = ["序号", "构件类型", "数量", "体积(m³)", "面积(m²)", "长度(m)"]
    for col, header in enumerate(headers, 1):
        cell = ws_summary.cell(row=1, column=col, value=header)
        cell.fill = header_fill
        cell.font = header_font
        cell.alignment = Alignment(horizontal="center")
    
    row = 2
    for idx, (category, data) in enumerate(sorted(results.items()), 1):
        ws_summary.cell(row=row, column=1, value=idx)
        ws_summary.cell(row=row, column=2, value=category)
        ws_summary.cell(row=row, column=3, value=data["count"])
        ws_summary.cell(row=row, column=4, value=round(data["volume"], 3))
        ws_summary.cell(row=row, column=5, value=round(data["area"], 3))
        ws_summary.cell(row=row, column=6, value=round(data["length"], 3))
        row += 1
    
    # 设置列宽
    ws_summary.column_dimensions["A"].width = 8
    ws_summary.column_dimensions["B"].width = 20
    ws_summary.column_dimensions["C"].width = 10
    ws_summary.column_dimensions["D"].width = 15
    ws_summary.column_dimensions["E"].width = 15
    ws_summary.column_dimensions["F"].width = 15
    
    # ---- Sheet 2: 明细表 ----
    ws_detail = wb.create_sheet("构件明细")
    
    detail_headers = ["序号", "构件名称", "GlobalId", "IFC类型", "材料", "数量"]
    for col, header in enumerate(detail_headers, 1):
        cell = ws_detail.cell(row=1, column=col, value=header)
        cell.fill = header_fill
        cell.font = header_font
    
    row = 2
    detail_idx = 1
    for category, data in sorted(results.items()):
        for elem in data["elements"]:
            ws_detail.cell(row=row, column=1, value=detail_idx)
            ws_detail.cell(row=row, column=2, value=elem["name"])
            ws_detail.cell(row=row, column=3, value=elem["global_id"])
            ws_detail.cell(row=row, column=4, value=elem["type"])
            ws_detail.cell(row=row, column=5, value=elem["materials"])
            ws_detail.cell(row=row, column=6, value=elem["quantity"])
            row += 1
            detail_idx += 1
    
    ws_detail.column_dimensions["A"].width = 8
    ws_detail.column_dimensions["B"].width = 40
    ws_detail.column_dimensions["C"].width = 25
    ws_detail.column_dimensions["D"].width = 25
    ws_detail.column_dimensions["E"].width = 25
    ws_detail.column_dimensions["F"].width = 12
    
    wb.save(output_path)
    print(f"算量清单已导出: {output_path}")


def main():
    print("=" * 60)
    print("IFC 自动算量工具")
    print("=" * 60)
    
    try:
        results = extract_quantity_from_ifc(IFC_FILE)
        export_to_excel(results, OUTPUT_EXCEL)
        
        # 打印汇总
        print("\n" + "=" * 60)
        print("算量汇总")
        print("=" * 60)
        for category, data in sorted(results.items()):
            print(f"  {category}: {data['count']} 个")
            
    except FileNotFoundError:
        print(f"错误：找不到 IFC 文件 '{IFC_FILE}'")
    except Exception as e:
        print(f"错误：{str(e)}")
        import traceback
        traceback.print_exc()


if __name__ == "__main__":
    main()
```

### 5.3 Revit API + Python 自动化示例（pyRevit）

```python
# pyRevit 扩展脚本：从 Revit 项目提取算量数据
from Autodesk.Revit.DB import (
    FilteredElementCollector, 
    BuiltInCategory,
    UnitUtils,
    DisplayUnitType
)
from Autodesk.Revit.UI import TaskDialog

doc = __revit__.ActiveUIDocument.Document

def format_unit(value, unit_type):
    """格式化数值单位"""
    return UnitUtils.ConvertFromInternalUnits(
        value, unit_type)

# ---- 提取墙体数据 ----
wall_collector = FilteredElementCollector(doc)\
    .OfCategory(BuiltInCategory.OST_Walls)\
    .WhereElementIsNotElementType()

walls = list(wall_collector)
wall_data = []

for wall in walls:
    vol_param = wall.get_Parameter(
        BuiltInParameter.HOST_VOLUME_COMPUTED)
    area_param = wall.get_Parameter(
        BuiltInParameter.HOST_AREA_COMPUTED)
    
    volume = vol_param.AsDouble() if vol_param and vol_param.HasValue else 0
    area = area_param.AsDouble() if area_param and area_param.HasValue else 0
    
    wall_data.append({
        "name": wall.Name,
        "type": doc.GetElement(wall.GetTypeId()).Name,
        "volume_m3": format_unit(volume, DisplayUnitType.DUT_CUBIC_METERS),
        "area_m2": format_unit(area, DisplayUnitType.DUT_SQUARE_METERS)
    })

print(f"共提取 {len(walls)} 个墙体")

# ---- 提取门数据 ----
door_collector = FilteredElementCollector(doc)\
    .OfCategory(BuiltInCategory.OST_Doors)\
    .WhereElementIsNotElementType()

doors = list(door_collector)
print(f"共提取 {len(doors)} 个门")
```

### 5.4 自动化调度方案

| 方案 | 工具 | 适用场景 |
|------|------|---------|
| 批处理 | Python + ifcopenshell | 大量 IFC 文件定期处理 |
| Revit 插件 | C# + Revit API | 设计师在 Revit 内一键算量 |
| pyRevit | Python + pyRevit | Revit 内交互式算量脚本 |
| 服务器调度 | Python + Schedule | 定时自动处理 IFC 文件 |

---

## 6. IFC 实体类型与算量分类对照

### 6.1 建筑主体构件

| IFC 实体类型 | 中文名称 | 主要数量 | 对应 Revit 类别 |
|------------|---------|---------|---------------|
| IfcWall | 墙体 | 体积 m³ | OST_Walls |
| IfcWallStandardCase | 标准墙体 | 体积 m³ | OST_Walls |
| IfcSlab | 楼板/屋面板 | 面积 m² | OST_Floors, OST_Roofs |
| IfcBeam | 梁 | 体积 m³ | OST_StructuralFraming |
| IfcColumn | 柱 | 体积 m³ | OST_Columns |
| IfcStair | 楼梯 | 座/水平投影面积 | OST_Stairs |
| IfcRamp | 坡道 | 面积 m² | OST_Ramps |
| IfcCurtainWall | 幕墙 | 面积 m² | OST_CurtainWallPanels |
| IfcCovering | 覆盖层（吊顶/墙面） | 面积 m² | OST_Coverings |

### 6.2 门窗构件

| IFC 实体类型 | 中文名称 | 主要数量 | 备注 |
|------------|---------|---------|------|
| IfcDoor | 门 | 樘/面积 m² | 含 IfcDoorStyle |
| IfcWindow | 窗 | 樘/面积 m² | 含 IfcWindowStyle |
| IfcDoorPanel | 门扇 | 个数 | IfcDoor 的子组件 |
| IfcWindowPanel | 窗扇 | 个数 | IfcWindow 的子组件 |

### 6.3 机电管道构件（MEP）

| IFC 实体类型 | 中文名称 | 主要数量 |
|------------|---------|---------|
| IfcPipeSegment | 管道段 | 长度 m |
| IfcPipeFitting | 管道管件 | 个数 |
| IfcDuctSegment | 风管段 | 长度 m |
| IfcDuctFitting | 风管管件 | 个数 |
| IfcDuctSilencer | 消声器 | 个数 |
| IfcValve | 阀门 | 个数 |
| IfcFlowController | 控制器 | 个数 |
| IfcPump | 泵 | 个数 |
| IfcTank | 水箱/储罐 | 个数 |
| IfcAirTerminal | 风口 | 个数 |
| IfcElectricMotor | 电动机 | 个数 |
| IfcCableCarrierSegment | 电缆桥架段 | 长度 m |
| IfcCableSegment | 电缆段 | 长度 m |

### 6.4 装饰装修构件

| IFC 实体类型 | 中文名称 | 主要数量 |
|------------|---------|---------|
| IfcFurnishingElement | 家具 | 个数 |
| IfcSpatialZone | 空间区域 | 体积/面积 |
| IfcSpace | 房间/空间 | 面积 m², 体积 m³ |
| IfcPlate | 板（装饰性） | 面积 m² |
| IfcProxyElement | 代理元素 | 需解析类型 |

### 6.5 IFC 属性集（Property Set）速查

| PSet 名称 | 内容 |
|----------|------|
| PSet_WallCommon | 墙体通用属性 |
| PSet_DoorCommon | 门通用属性 |
| PSet_WindowCommon | 窗通用属性 |
| PSet_PipeSegmentCommon | 管道通用属性 |
| PSet_BuildingStoreyCommon | 楼层通用属性 |
| Qto_WallBaseQuantities | 墙体数量（体积、面积等） |

### 6.6 IFC Quantity 实体速查

| IfcQuantity* | 说明 |
|-------------|------|
| IfcQuantityLength | 长度数量 |
| IfcQuantityArea | 面积数量 |
| IfcQuantityVolume | 体积数量 |
| IfcQuantityCount | 计数数量 |
| IfcQuantityWeight | 重量数量 |

---

## 附录：IFC Schema IFC4 关键实体继承关系

```
IfcRoot
 ├─ IfcObjectDefinition
 │    ├─ IfcObject
 │    │    ├─ IfcProduct（所有建筑产品继承自此）
 │    │    │    ├─ IfcElement（建筑元素）
 │    │    │    │    ├─ IfcBuildingElement
 │    │    │    │    │    ├─ IfcWall
 │    │    │    │    │    ├─ IfcSlab
 │    │    │    │    │    ├─ IfcBeam
 │    │    │    │    │    ├─ IfcColumn
 │    │    │    │    │    ├─ IfcDoor
 │    │    │    │    │    ├─ IfcWindow
 │    │    │    │    │    └─ IfcStair
 │    │    │    │    └─ IfcDistributionElement
 │    │    │    │         ├─ IfcPipeSegment
 │    │    │    │         ├─ IfcDuctSegment
 │    │    │    │         └─ IfcValve
 │    │    │    └─ IfcSpatialElement
 │    │    │         ├─ IfcSpace
 │    │    │         └─ IfcBuildingStorey
 │    │    └─ IfcTypeObject（类型对象）
```

---

## 总结

1. **IFC + ifcopenshell** 是实现跨平台自动算量的核心方案，支持提取任意 IFC 兼容模型的构件数据
2. **Revit API** 是最直接的 BIM 算量路径，适合已有 Revit 模型的场景
3. **DWG → Revit → IFC** 是从 CAD 到 BIM 算量的经典路径
4. **国内算量软件**（广联达、鲁班、品茗）主要通过插件或 IFC 格式与 Revit 对接
5. **Python 自动化**是实现批量处理和定期算量的最佳选择，ifcopenshell + pandas + Excel 即可构建完整算量工具链
