# DWG 图纸自动化算量 — 实操研究

> 基于 ezdxf 1.4.2 + Python 3.9 实测
> 研究目标：导线穿管、桥架、设备数量、标注文字自动提取

---

## 一、ezdxf 读取 DWG 图纸核心方法

### 1.1 基本读取流程

```python
import ezdxf

# 打开 DWG 文件（ezdxf 内部将 DWG 转为 DXF 处理）
doc = ezdxf.readfile("your_drawing.dwg")

# 获取模型空间（主图纸区域）
msp = doc.modelspace()

# 获取所有图层
for layer in doc.layers:
    print(f"图层名: {layer.dxf.name}, 颜色: {layer.dxf.color}")
```

### 1.2 四种核心数据读取方法

#### (1) Layers（图层）
```python
# 遍历所有图层
for layer in doc.layers:
    print(f"  {layer.dxf.name} | 颜色={layer.dxf.color} | 冻结={layer.is_frozen()}")

# 按名称查图层
layer = doc.layers.get('E-POWER')

# 添加图层
doc.layers.add('MY_LAYER', color=7)

# 筛选含特定前缀的图层
electrical_layers = [l for l in doc.layers if l.dxf.name.startswith('E-')]
```

#### (2) Entities（实体）- 三种方式

**方式 A：query() 查询语言（最常用）**
```python
# 查询所有实体
all_entities = msp.query('*')

# 按类型查
lines = msp.query('LINE')
circles = msp.query('CIRCLE')
polylines = msp.query('LWPOLYLINE')
texts = msp.query('TEXT') + msp.query('MTEXT')

# 按图层查
e_lines = msp.query('[layer=="E-POWER"]')

# 组合：某图层 + 某类型
tray_lines = msp.query('LWPOLYLINE[layer=="E-TRAY"]')

# 正则匹配图层名
import re
matched = [e for e in msp.query('*') 
           if re.match(r'E-(POWER|LIGHT|TRAY)', e.dxf.layer)]
```

**方式 B：groupby() 分组**
```python
by_layer = msp.groupby(key='layer')
for layer_name, entities in sorted(by_layer.items()):
    print(f"图层 '{layer_name}': {len(entities)} 个实体")

by_type = msp.groupby(key='dxftype')
for dtype, entities in by_type.items():
    print(f"{dtype}: {len(entities)}")
```

**方式 C：直接遍历（最原始）**
```python
for entity in msp:
    print(f"  {entity.dxftype} | layer={entity.dxf.layer}")
```

#### (3) Blocks（块/图块）
```python
# 所有块定义
for block_name in doc.block_records:
    print(f"  Block: {block_name}")

# 查特定块
block = doc.blocks.get('LIGHT Fixture')

# 所有块引用（模型空间中使用了哪些块）
block_refs = msp.query('INSERT')
for ref in block_refs:
    print(f"  块名={ref.dxf.name}, 位置={ref.dxf.insert}, layer={ref.dxf.layer}")

# 按块名统计数量
from collections import Counter
block_counts = Counter(ref.dxf.name for ref in block_refs)
for name, count in sorted(block_counts.items()):
    print(f"  {name}: {count}个")
```

#### (4) Layouts（布局）
```python
# 所有布局名
print(doc.layout_names())
# ['Model', 'Layout1', 'Layout2', ...]

# 获取模型空间
msp = doc.modelspace()

# 获取图纸空间（布局）
for layout_name in doc.layout_names_in_taborder():
    if layout_name != 'Model':
        layout = doc.layout(layout_name)
        print(f"布局 '{layout_name}': {len(list(layout))} 个实体")
```

### 1.3 常用实体属性速查表

| 实体类型 | 关键属性 | 用途 |
|---------|---------|------|
| `LINE` | `.dxf.start`, `.dxf.end` | 计算导线长度 |
| `LWPOLYLINE` | `.vertices()`, `.is_closed` | 计算导线/桥架长度 |
| `CIRCLE` | `.dxf.center`, `.dxf.radius` | 计算圆形设备、周长 |
| `ARC` | `.dxf.start_angle`, `.dxf.end_angle`, `.dxf.radius` | 计算弧长 |
| `INSERT` (BlockRef) | `.dxf.name`, `.dxf.insert`, `.attribs` | 统计设备数量 |
| `TEXT` | `.dxf.text`, `.dxf.height` | 提取标注文字 |
| `MTEXT` | `.dxf.text` | 提取多行标注 |
| `DIMENSION` | `.dxf.actual_measurement` | 提取尺寸标注 |
| `HATCH` | `.dxf.pattern_name` | 区域填充（面积） |

---

## 二、图层命名规范参考表

### 2.1 中国标准 (GB/T 50114-2010 等)

```
前缀      系统              示例图层名
──────────────────────────────────────────────
A-        建筑              A-WALL, A-DOOR, A-WINDOW
S-        结构              S-WALL, S-BEAM, S-COLUMN
E-        电气（强电）      E-POWER, E-LIGHT, E-DIST
F-        消防              F-SPRINKLER, F-HYDRANT, F-ALARM
W-        给水排水          W-SUPPLY, W-DRAIN, W-HVAC
H-        暖通              H-SUPPLY, H-RETURN, H-DUCT
T-        弱电/电信          T-DATA, T-PHONE, T-TV
C-        综合布线          C-WORK, C-HORIZ, C-backbone
M-        机械              M-EQUIP, M-PIPE
R-        精装/装饰          R-FIXTURE, R-CABINET
D-        市政/总图          D-ROAD, D-SITE, D-GRID
ELV-      弱电（常用写法）    ELV-FIRE, ELV-SECURITY, ELV-NETWORK
FA-       火灾报警           FA-ZONEA, FA-PULL, FA-DETECTOR
```

### 2.2 国际标准 (AutoCAD ISO 标准)

```
前缀/名称            系统
─────────────────────────────────
ARCH / A-*           建筑
STR / S-*            结构
ELEC / E-*           电气
PLMB / P-*           给排水
HVAC / M-*           暖通/机械
FIRE / F-*           消防
TELE / T-*           电信/网络
DATA / D-*           数据/综合布线
ANNO-*               标注/注释
DIM-*                尺寸标注
TEXT-*               文字
```

### 2.3 常见 CAD 软件图层命名约定

**天正（TArch）**：
```
TL-         墙线
TZ-         柱子
TW-         窗户
TD-         门
TH-         楼梯
TL-         栏杆
E-          照明
TV-         有线电视
```

**Revit 导出 DWG 常见命名**：
```
A-         建筑
S-         结构
M-         机电
P-         给排水
E-         电气
F-         消防
```

### 2.4 MEP 各子系统详细图层规范

**电气系统 (E-/ELEC-)：**
```
E-POWER-**      配电系统
E-LIGHT-**      照明系统
E-Socket-**     插座系统
E-Ground-**     接地系统
E-Trunk-**      干线/桥架
E-TRAY-**       电缆桥架
E-Conduit-**    导管/穿管
E-FIRE-**       消防电源
```

**弱电系统 (T-/ELV-/FA-)：**
```
ELV-DATA-**     数据网络
ELV-PHONE-**    电话
ELV-FA-**       火灾报警
ELV-CCTV-**     视频监控
ELV-ACS-**      门禁
ELV-BA-**       楼宇自控
ELV-EXTV-**     广播/背景音乐
ELV-TV-**       有线电视
```

**消防系统 (F-/FA-)：**
```
F-SPRINK-**     喷淋系统
F-HYDRANT-**    消火栓
F-ALARM-**      报警系统
F-MAIN-**       消防给水
F-DRAIN-**      消防排水
F-SMOKE-**      防排烟
```

**给排水 (W-/P-)：**
```
W-SUPPLY-**     给水管
W-DRAIN-**      排水管
W-HOT-**        热水管
W-GAS-**        燃气管
W-IRRIG-**      绿化灌溉
```

---

## 三、自动提取工程量实战代码

### 3.1 完整算量框架

```python
"""
DWG 图纸自动化算量框架
支持：导线长度、桥架长度、设备数量、标注文字提取
"""

import ezdxf
import re
import math
from math import pi, radians
from collections import defaultdict, Counter
from dataclasses import dataclass, field
from typing import List, Dict, Tuple, Optional


@dataclass
class QuantityItem:
    """算量项"""
    name: str           # 名称，如 "WD100 配电导线"
    layer: str          # 来源图层
    qty: float          # 数量
    unit: str           # 单位
    detail: str = ""    # 明细说明


class DWGMEPQuantityExtractor:
    """DWG MEP 工程量提取器"""
    
    # ========== 电气系统图层模式 ==========
    ELECTRICAL_LINE_PATTERNS = [
        r'E-POWER', r'E-LIGHT', r'E-SOCK', r'E-CON',
        r'E-TRUNK', r'E-TRAY', r'E-GROUND'
    ]
    FIRE_ALARM_PATTERNS = [
        r'FA-', r'F-ALARM', r'F-SPRINK', r'F-HYDR',
        r'F-FIRE', r'F-MAIN', r'ELV-FA', r'ELV-FIRE'
    ]
    ELV_PATTERNS = [
        r'ELV-', r'T-', r'C-', r'DATA', r'TV-',
        r'CCTV', r'ACS-', r'BA-', r'EXTV'
    ]
    WATER_PATTERNS = [
        r'W-', r'WATER', r'P-', r'PLMB'
    ]
    
    def __init__(self, filepath: str):
        self.doc = ezdxf.readfile(filepath)
        self.msp = self.doc.modelspace()
        self.results: List[QuantityItem] = []
        self._entity_cache: Dict[str, list] = {}
    
    # ---- 工具方法 ----
    
    def _distance(self, p1: Tuple, p2: Tuple) -> float:
        """计算两点距离（支持2D/3D元组）"""
        from ezdxf.math import Vec3
        v1 = Vec3(p1[0], p1[1], p1[2] if len(p1) > 2 else 0)
        v2 = Vec3(p2[0], p2[1], p2[2] if len(p2) > 2 else 0)
        return (v2 - v1).magnitude
    
    def _pattern_matches(self, layer_name: str, patterns: List[str]) -> bool:
        """判断图层是否匹配任意模式"""
        for pat in patterns:
            if re.match(pat.replace('*', '.*'), layer_name, re.IGNORECASE):
                return True
        return False
    
    def _get_all_layers(self) -> List[str]:
        """获取所有图层名"""
        return [layer.dxf.name for layer in self.doc.layers]
    
    def _get_entities_by_layer_pattern(self, patterns: List[str], 
                                         entity_types: List[str] = None) -> List:
        """按图层模式获取实体"""
        cache_key = f"{','.join(patterns)}_{','.join(entity_types or ['*'])}"
        if cache_key in self._entity_cache:
            return self._entity_cache[cache_key]
        
        results = []
        for entity in self.msp:
            if not self._pattern_matches(entity.dxf.layer, patterns):
                continue
            if entity_types and entity.dxftype not in entity_types:
                continue
            results.append(entity)
        
        self._entity_cache[cache_key] = results
        return results
    
    # ---- 长度计算 ----
    
    def calc_line_length(self, entity) -> float:
        """计算 LINE 长度"""
        return self._distance(entity.dxf.start, entity.dxf.end)
    
    def calc_polyline_length(self, entity) -> float:
        """计算 LWPOLYLINE 总长度"""
        verts = list(entity.vertices())
        if not verts or len(verts) < 2:
            return 0.0
        total = 0.0
        for i in range(len(verts) - 1):
            total += self._distance(verts[i], verts[i+1])
        # 如果闭合，加上最后一段
        if entity.is_closed:
            total += self._distance(verts[-1], verts[0])
        return total
    
    def calc_arc_length(self, entity) -> float:
        """计算 ARC 弧长"""
        r = entity.dxf.radius
        start = entity.dxf.start_angle
        end = entity.dxf.end_angle
        # 处理角度跨越0°的情况
        if end < start:
            end += 360
        angle_deg = end - start
        return r * radians(angle_deg)
    
    def calc_circle_circumference(self, entity) -> float:
        """计算 CIRCLE 周长"""
        return 2 * pi * entity.dxf.radius
    
    def calc_polyline_area(self, entity) -> float:
        """用鞋带算法计算闭合多边形面积"""
        verts = list(entity.vertices())
        if not entity.is_closed or len(verts) < 3:
            return 0.0
        n = len(verts)
        area = 0.0
        for i in range(n):
            j = (i + 1) % n
            area += verts[i][0] * verts[j][1]
            area -= verts[j][0] * verts[i][1]
        return abs(area) / 2.0
    
    # ---- 提取方法 ----
    
    def extract_electrical_conduit(self) -> List[QuantityItem]:
        """提取电气导线穿管（按图层分组统计长度）"""
        print("正在提取导线穿管工程量...")
        
        # 按图层分别统计
        layer_groups = defaultdict(list)
        lines = self._get_entities_by_layer_pattern(
            self.ELECTRICAL_LINE_PATTERNS, ['LINE', 'LWPOLYLINE', 'ARC', 'CIRCLE']
        )
        
        for entity in lines:
            layer_groups[entity.dxf.layer].append(entity)
        
        for layer, entities in sorted(layer_groups.items()):
            total_length = 0.0
            for e in entities:
                etype = e.dxftype
                if etype == 'LINE':
                    total_length += self.calc_line_length(e)
                elif etype == 'LWPOLYLINE':
                    total_length += self.calc_polyline_length(e)
                elif etype == 'ARC':
                    total_length += self.calc_arc_length(e)
                elif etype == 'CIRCLE':
                    # 圆形通常代表线管端点等，不计入长度
                    pass
            
            if total_length > 0:
                self.results.append(QuantityItem(
                    name=f"导线/穿管 [{layer}]",
                    layer=layer,
                    qty=total_length,
                    unit="m",
                    detail=f"LINE/PLINE 共 {len(entities)} 段"
                ))
        
        return self.results
    
    def extract_cable_tray(self) -> List[QuantityItem]:
        """提取电缆桥架（按图层统计）"""
        print("正在提取电缆桥架工程量...")
        
        tray_entities = self._get_entities_by_layer_pattern(
            [r'E-TRAY', r'E-TRUNK', r'TRAY', r'TRUNK'], 
            ['LINE', 'LWPOLYLINE', 'ARC']
        )
        
        layer_groups = defaultdict(list)
        for e in tray_entities:
            layer_groups[e.dxf.layer].append(e)
        
        for layer, entities in sorted(layer_groups.items()):
            total_length = 0.0
            for e in entities:
                if e.dxftype == 'LINE':
                    total_length += self.calc_line_length(e)
                elif e.dxftype == 'LWPOLYLINE':
                    total_length += self.calc_polyline_length(e)
                elif e.dxftype == 'ARC':
                    total_length += self.calc_arc_length(e)
            
            if total_length > 0:
                self.results.append(QuantityItem(
                    name=f"电缆桥架 [{layer}]",
                    layer=layer,
                    qty=total_length,
                    unit="m",
                    detail=f"共 {len(entities)} 段"
                ))
        
        return self.results
    
    def extract_equipment_count(self) -> List[QuantityItem]:
        """提取设备数量（统计 BlockRef/INSERT）"""
        print("正在提取设备数量...")
        
        block_refs = list(self.msp.query('INSERT'))
        
        # 按块名 + 图层分组
        counter = Counter()
        for ref in block_refs:
            key = (ref.dxf.name, ref.dxf.layer)
            counter[key] += 1
        
        for (block_name, layer), count in sorted(counter.items()):
            if count == 0:
                continue
            # 尝试从块属性获取规格信息
            detail = f"layer={layer}"
            attrs = list(ref.attribs)
            if attrs:
                attr_texts = [f"{a.dxf.tag}={a.dxf.text}" for a in attrs]
                detail += f" | attrs: {', '.join(attr_texts[:3])}"  # 最多3个属性
            
            self.results.append(QuantityItem(
                name=f"设备 [{block_name}]",
                layer=layer,
                qty=count,
                unit="个",
                detail=detail
            ))
        
        return self.results
    
    def extract_dimensions(self) -> List[QuantityItem]:
        """提取尺寸标注（配电箱、柜体尺寸等）"""
        print("正在提取尺寸标注...")
        
        dims = self.msp.query('DIMENSION') + self.msp.query('LINEARMEDIMENSION') \
               + self.msp.query('ALIGNEDDIMENSION')
        
        layer_groups = defaultdict(list)
        for dim in dims:
            layer_groups[dim.dxf.layer].append(dim)
        
        for layer, dims_in_layer in sorted(layer_groups.items()):
            for dim in dims_in_layer:
                try:
                    measurement = float(dim.dxf.actual_measurement)
                    self.results.append(QuantityItem(
                        name=f"标注 [{layer}]",
                        layer=layer,
                        qty=measurement,
                        unit="mm或根据图纸单位",
                        detail=f"类型={dim.dxftype}"
                    ))
                except (ValueError, AttributeError):
                    pass
        
        return self.results
    
    def extract_annotations(self) -> Dict[str, List[str]]:
        """提取所有文字标注（配电编号、线缆规格等）"""
        print("正在提取标注文字...")
        
        texts = {
            'TEXT': list(self.msp.query('TEXT')),
            'MTEXT': list(self.msp.query('MTEXT'))
        }
        
        annotations = defaultdict(list)
        
        for text_type, text_entities in texts.items():
            for t in text_entities:
                layer = t.dxf.layer
                content = t.dxf.text.strip() if hasattr(t.dxf, 'text') else ''
                if content:
                    annotations[layer].append(content)
        
        return dict(annotations)
    
    def extract_fire_alarm(self) -> List[QuantityItem]:
        """提取消防报警设备（探测器、模块、手报等）"""
        print("正在提取消防报警设备...")
        
        # 消防设备通常以 Block 形式存在
        fire_refs = self._get_entities_by_layer_pattern(
            self.FIRE_ALARM_PATTERNS, ['INSERT', 'CIRCLE', 'LINE']
        )
        
        # 按块名统计 INSERT
        block_refs = [e for e in fire_refs if e.dxftype == 'INSERT']
        counter = Counter()
        for ref in block_refs:
            counter[(ref.dxf.name, ref.dxf.layer)] += 1
        
        for (block_name, layer), count in sorted(counter.items()):
            self.results.append(QuantityItem(
                name=f"消防设备 [{block_name}]",
                layer=layer,
                qty=count,
                unit="个",
                detail=""
            ))
        
        # 按图层统计非块引用（布线）
        wire_refs = [e for e in fire_refs if e.dxftype != 'INSERT']
        layer_groups = defaultdict(list)
        for e in wire_refs:
            layer_groups[e.dxf.layer].append(e)
        
        for layer, entities in sorted(layer_groups.items()):
            total_length = sum(
                self.calc_line_length(e) if e.dxftype == 'LINE'
                else self.calc_polyline_length(e) if e.dxftype == 'LWPOLYLINE'
                else 0 for e in entities
            )
            if total_length > 0:
                self.results.append(QuantityItem(
                    name=f"消防布线 [{layer}]",
                    layer=layer,
                    qty=total_length,
                    unit="m",
                    detail=f"共 {len(entities)} 段"
                ))
        
        return self.results
    
    def extract_water_pipes(self) -> List[QuantityItem]:
        """提取给排水管道"""
        print("正在提取给排水管道...")
        
        pipe_entities = self._get_entities_by_layer_pattern(
            self.WATER_PATTERNS, ['LINE', 'LWPOLYLINE', 'ARC', 'CIRCLE']
        )
        
        layer_groups = defaultdict(list)
        for e in pipe_entities:
            layer_groups[e.dxf.layer].append(e)
        
        for layer, entities in sorted(layer_groups.items()):
            total_length = 0.0
            for e in entities:
                if e.dxftype == 'LINE':
                    total_length += self.calc_line_length(e)
                elif e.dxftype == 'LWPOLYLINE':
                    total_length += self.calc_polyline_length(e)
                elif e.dxftype == 'ARC':
                    total_length += self.calc_arc_length(e)
                elif e.dxftype == 'CIRCLE':
                    # 管径标注圆
                    pass
            
            if total_length > 0:
                self.results.append(QuantityItem(
                    name=f"给排水管道 [{layer}]",
                    layer=layer,
                    qty=total_length,
                    unit="m",
                    detail=f"共 {len(entities)} 段"
                ))
        
        return self.results
    
    def extract_elv_systems(self) -> List[QuantityItem]:
        """提取弱电系统（网络、电话、监控等）"""
        print("正在提取弱电系统...")
        
        elv_entities = self._get_entities_by_layer_pattern(
            self.ELV_PATTERNS, ['LINE', 'LWPOLYLINE', 'INSERT']
        )
        
        # 分开统计布线和设备
        wires = [e for e in elv_entities if e.dxftype in ('LINE', 'LWPOLYLINE')]
        devices = [e for e in elv_entities if e.dxftype == 'INSERT']
        
        # 统计布线
        wire_counter = Counter()
        for e in wires:
            wire_counter[e.dxf.layer] += 1
        
        for layer, count in sorted(wire_counter.items()):
            total_len = sum(
                self.calc_line_length(e) if e.dxftype == 'LINE'
                else self.calc_polyline_length(e)
                for e in wires if e.dxf.layer == layer
            )
            if total_len > 0:
                self.results.append(QuantityItem(
                    name=f"弱电布线 [{layer}]",
                    layer=layer,
                    qty=total_len,
                    unit="m",
                    detail=f"共 {count} 段"
                ))
        
        # 统计设备
        dev_counter = Counter()
        for ref in devices:
            dev_counter[(ref.dxf.name, ref.dxf.layer)] += 1
        
        for (block_name, layer), count in sorted(dev_counter.items()):
            self.results.append(QuantityItem(
                name=f"弱电设备 [{block_name}]",
                layer=layer,
                qty=count,
                unit="个",
                detail=""
            ))
        
        return self.results
    
    # ---- 主流程 ----
    
    def run_full_extraction(self) -> Dict:
        """执行全量提取"""
        print(f"\n{'='*50}")
        print(f"开始提取: {self.doc.filename}")
        print(f"模型空间实体总数: {len(list(self.msp))}")
        print(f"图层总数: {len(list(self.doc.layers))}")
        print(f"{'='*50}\n")
        
        self.extract_electrical_conduit()
        self.extract_cable_tray()
        self.extract_equipment_count()
        self.extract_fire_alarm()
        self.extract_water_pipes()
        self.extract_elv_systems()
        
        return self.get_summary()
    
    def get_summary(self) -> Dict:
        """生成汇总报告"""
        # 按单位分组
        by_unit = defaultdict(list)
        for item in self.results:
            by_unit[item.unit].append(item)
        
        report = {
            'total_items': len(self.results),
            'by_unit': {},
            'by_layer': defaultdict(list),
            'full_list': self.results
        }
        
        for unit, items in by_unit.items():
            total_qty = sum(item.qty for item in items)
            report['by_unit'][unit] = {
                'count': len(items),
                'total_qty': round(total_qty, 2)
            }
        
        for item in self.results:
            report['by_layer'][item.layer].append(item)
        
        return report
    
    def print_report(self, summary: Dict):
        """打印报告"""
        print(f"\n{'='*60}")
        print(" 工程量提取报告")
        print(f"{'='*60}")
        
        print("\n▶ 按单位汇总:")
        for unit, info in sorted(summary['by_unit'].items()):
            print(f"  [{unit}] 共 {info['count']} 项, 总量={info['total_qty']}")
        
        print("\n▶ 明细清单:")
        for item in sorted(self.results, key=lambda x: (x.unit, x.name)):
            if item.unit == 'm':
                print(f"  {item.name:40s} {item.qty:12.2f} {item.unit}  ({item.detail})")
            else:
                print(f"  {item.name:40s} {item.qty:12.0f} {item.unit}  ({item.detail})")
        
        print(f"\n总计: {summary['total_items']} 项")
        print(f"{'='*60}")
```

### 3.2 使用示例

```python
# 基本用法
extractor = DWGMEPQuantityExtractor("example.dwg")
summary = extractor.run_full_extraction()
extractor.print_report(summary)

# 只提取特定系统
extractor2 = DWGMEPQuantityExtractor("electrical_plan.dwg")
extractor2.extract_electrical_conduit()
extractor2.extract_cable_tray()
extractor2.extract_equipment_count()

# 导出到 Excel
import pandas as pd
df = pd.DataFrame([
    {
        '名称': item.name,
        '图层': item.layer,
        '数量': item.qty,
        '单位': item.unit,
        '明细': item.detail
    }
    for item in extractor.results
])
df.to_excel("工程量清单.xlsx", index=False)
```

---

## 四、常见图纸识别模式

### 4.1 配电系统识别

```
配电箱 (PANEL BOARD)
  → Block名: PANEL_*, DIST_*, PDB_*
  → 图层: E-POWER, E-DIST
  → 属性: RATING (额定电流), NAME (配电箱编号)

配电干线 (RISER / TRUNK)
  → 图层: E-TRUNK, E-TRAY, E-POWER
  → 实体: LWPOLYLINE 或 LINE（表示桥架中心线）
  → 宽度信息: 通常标注在线旁或在线内

插座/开关
  → Block名: RECEP_*, SW_*, OUTLET_*
  → 属性: WATTAGE, VOLTAGE, TYPE
```

### 4.2 火灾报警识别

```
感烟探测器 (SMOKE DETECTOR)
  → Block名: SD_*, DET_*, SMK_*
  → 图层: FA-*, F-ALARM, ELV-FA

手动报警按钮 (PULL STATION)
  → Block名: PS_*, PULL_*, MANUAL_*

声光报警器 (HORN/STROBE)
  → Block名: HORN_*, STR_*, ALARM_*

火灾报警控制柜 (FACP)
  → Block名: FACP_*, CTRL_*, FIRE_*
```

### 4.3 桥架/线槽识别

```
电缆桥架 (CABLE TRAY)
  → 图层: E-TRAY, E-TRUNK, TRAY-*
  → 实体: LWPOLYLINE (闭合矩形/多边形表示桥架轮廓)
  → 识别方式: 闭合 PLINE，按长度统计；标注处查宽度

吊装线槽 (CONDUIT)
  → 图层: E-CON, E-CONduit
  → 实体: LINE 或 PLINE
  → 通常标注规格: SC100, MT25 等
```

### 4.4 给排水识别

```
给水管
  → 图层: W-SUPPLY, W-HOT, P-SUPPLY
  → 实体: LINE 或 PLINE
  → 标注: 管径如 DN50, DN100

排水管
  → 图层: W-DRAIN, P-DRAIN
  → 实体: LINE 或 PLINE
  → 标注: De75, De110

卫生器具 (FIXTURES)
  → Block名: TOILET_*, LAV_*, SINK_*, WATER_HEATER_*
  → 按设备类型分类统计
```

---

## 五、图元长度/面积计算工具函数

```python
"""
常用几何计算工具
"""
import math
from typing import Tuple, List, Union


def line_length(start: Tuple, end: Tuple) -> float:
    """两点间距离"""
    return math.sqrt((end[0]-start[0])**2 + (end[1]-start[1])**2)


def polyline_length(vertices: List[Tuple]) -> float:
    """多段线总长度"""
    if len(vertices) < 2:
        return 0.0
    return sum(line_length(vertices[i], vertices[i+1]) 
               for i in range(len(vertices)-1))


def arc_length(radius: float, start_angle: float, 
               end_angle: float) -> float:
    """圆弧弧长（角度制）"""
    if end_angle < start_angle:
        end_angle += 360
    angle_rad = math.radians(end_angle - start_angle)
    return radius * angle_rad


def circle_circumference(radius: float) -> float:
    """圆周长"""
    return 2 * math.pi * radius


def circle_area(radius: float) -> float:
    """圆面积"""
    return math.pi * radius * radius


def polygon_area_shoelace(vertices: List[Tuple]) -> float:
    """鞋带算法计算多边形面积（2D顶点列表）"""
    n = len(vertices)
    if n < 3:
        return 0.0
    area = 0.0
    for i in range(n):
        j = (i + 1) % n
        area += vertices[i][0] * vertices[j][1]
        area -= vertices[j][0] * vertices[i][1]
    return abs(area) / 2.0


def segment_midpoint(p1: Tuple, p2: Tuple) -> Tuple:
    """线段中点"""
    return ((p1[0]+p2[0])/2, (p1[1]+p2[1])/2)


def point_to_line_distance(point: Tuple, 
                            line_start: Tuple, line_end: Tuple) -> float:
    """点到线段的垂直距离"""
    px, py = point[0], point[1]
    x1, y1 = line_start[0], line_start[1]
    x2, y2 = line_end[0], line_end[1]
    
    # 向量
    dx = x2 - x1
    dy = y2 - y1
    
    if dx == 0 and dy == 0:
        return line_length(point, line_start)
    
    t = max(0, min(1, ((px-x1)*dx + (py-y1)*dy) / (dx*dx + dy*dy)))
    proj_x = x1 + t * dx
    proj_y = y1 + t * dy
    
    return line_length(point, (proj_x, proj_y))
```

---

## 六、参考资料

### 官方文档
- **ezdxf 官方文档**: https://ezdxf.readthedocs.io/
- **ezdxf GitHub**: https://github.com/mozman/ezdxf
- **PyPI**: https://pypi.org/project/ezdxf/

### 学习资源
- **CAD2BIM 自动化**: https://gis.stackexchange.com/questions/tagged/ezdxf
- **Python CAD Users Group**: https://github.com/pythoncad

### 标准规范
- **GB/T 50114-2010**: 建筑制图统一标准
- **GB/T 50106-2010**: 建筑给水排水制图标准
- **GB/T 50114-2001**: 暖通空调制图标准
- **JGJ/T 16-2008**: 民用建筑电气设计规范

### 相关工具补充
- **ODA File Converter**: 将 DWG 转为 DXF（免费工具，非开源）
- **FreeCAD**: 可导入 DWG/DXF 进行三维建模
- **QCAD / LibreCAD**: 开源 2D CAD 查看器
- **DraftSight**: 达索系统出品的 DWG 查看器（免费版可用）

### 进阶方向
1. **Revit Dynamo** → 配合 Revit API 批量处理
2. **IfcOpenShell** → 处理 IFC 格式（BIM数据更结构化）
3. **pyg不如** → 将 CAD 图纸转为 PDF/A 进行 OCR 识别
4. **建筑信息模型 (BIM)** → IFC 格式可直接提取算量数据

---

## 七、框架使用注意事项

### 常见问题与解决方案

**Q1: 圆形实体周长被误当导线长度**
```python
# 排除方法：过滤掉表示管径标注的圆形
# 规则：半径 < 某阈值 + 所在图层为管径标注图层
if entity.dxftype == 'CIRCLE':
    if entity.dxf.radius < 5:  # 小圆 = 标注圆，不计
        continue
    # 大圆 = 设备基础，单独处理
```

**Q2: 多段线桥架宽度计算**
```python
# 闭合 PLINE 桥架宽度估算：
# 找到桥架中心线（通常是粗线层的偏移线）
# 或通过标注文字识别宽度（SC100 = 100mm宽）
```

**Q3: 中文字体/编码问题**
```python
# 某些 DWG 中文字体可能无法正确读取
# 尝试：
text_content = entity.dxf.text.encode('latin-1', errors='replace').decode('gbk')
# 或使用 mtext 替代 text
```

**Q4: 比例因子（单位换算）**
```python
# 大多数建筑图纸单位为 mm（1图单位=1mm）
# 如果导出 Excel 时需要转换为米：
length_in_meters = length_in_mm / 1000

# 查看图纸单位
units = doc.units
print(f"图纸单位: {units}")  # 4 = mm, 6 = m, 等
```

**Q5: 过滤模型空间外的实体**
```python
# 有些图纸在图纸空间（Layout）也有内容
# 只处理模型空间：
for layout in doc.layouts:
    if layout.name == 'Model':
        entities = list(layout)
```

---

*本框架为实操研究版本，生产环境使用前请务必用真实图纸进行测试校核。*
