# การหา Density Image ใน Point Cloud Rasterization

## ภาพรวม

**Density Image (ภาพความหนาแน่น)** คือ Channel ที่ 0 ของ multi-channel grid ที่แสดงจำนวนจุด (points) ที่ตกอยู่ในแต่ละ pixel ของภาพ 2D

## กระบวนการคำนวณ

### 1. การแบ่ง Grid
```
Point Cloud (3D)  →  2D Grid (320×320 pixels)
- ขนาด pixel: 0.125 m/pixel
- พื้นที่ครอบคลุม: 40×40 m (≈ 1 ไร่)
```

### 2. การแปลงพิกัด XYZ เป็น Grid Index

```python
# จาก pointcloud_rasterization.py (บรรทัด ~147-153)

# หาค่า bounds
x_min, y_min = points[:, 0].min(), points[:, 1].min()
x_max, y_max = points[:, 0].max(), points[:, 1].max()

# คำนวณขนาด grid
width = int(np.ceil((x_max - x_min) / pixel_size))
height = int(np.ceil((y_max - y_min) / pixel_size))

# แปลงพิกัดจริง (meters) เป็น grid index (pixels)
x_idx = ((points[:, 0] - x_min) / pixel_size).astype(int)
y_idx = ((points[:, 1] - y_min) / pixel_size).astype(int)

# จำกัดให้อยู่ในขอบเขต grid
x_idx = np.clip(x_idx, 0, width - 1)
y_idx = np.clip(y_idx, 0, height - 1)
```

**คำอธิบาย NumPy Functions:**

#### `np.ceil()` - ปัดเลขขึ้น
```python
np.ceil(3.2)  # = 4.0
np.ceil(3.8)  # = 4.0
np.ceil(-2.1) # = -2.0

# ในโค้ด:
width = int(np.ceil((50.7 - 0.0) / 0.125))
# = int(np.ceil(405.6))
# = int(406.0)
# = 406 pixels
```
**ทำไมต้องใช้?** เพื่อให้ grid ใหญ่พอครอบคลุมทุกจุด (ถ้าใช้ floor อาจมีจุดตกหล่น)

#### `.astype(int)` - แปลงชนิดข้อมูล
```python
arr = np.array([1.2, 3.7, 5.9])
arr.astype(int)  # = [1, 3, 5] (ตัดทศนิยมทิ้ง)
```

#### `np.clip()` - จำกัดค่าให้อยู่ในช่วง [min, max]
```python
# Syntax: np.clip(array, min_value, max_value)
np.clip([0, 5, 10, -3, 320], 0, 319)
# = [0, 5, 10, 0, 319]
#            ↑     ↑
#          ตัดเป็น 0  ตัดเป็น 319

# ในโค้ด: ป้องกันไม่ให้ index เกินขอบ grid
x_idx = np.clip([0, 5, 320, -1], 0, 319)
# = [0, 5, 319, 0]  ← ปลอดภัย ไม่ index out of bounds!
```
**ทำไมต้องใช้?** ป้องกัน error เวลา access array นอกขอบเขต

**ตัวอย่าง:**
```
Point ที่พิกัด (X=10.0, Y=20.0, Z=5.0)
- x_min = 0, pixel_size = 0.125
- x_idx = (10.0 - 0) / 0.125 = 80
- y_idx = (20.0 - 0) / 0.125 = 160
→ Point นี้ตกอยู่ใน pixel (row=160, col=80)
```

### 3. การสร้าง Linear Index

```python
# สร้าง index เดียวสำหรับ 2D grid (บรรทัด ~156)
linear_idx = y_idx * width + x_idx
```

**ทำไมต้องใช้ linear index?**
- แปลง 2D coordinate (row, col) เป็น 1D index
- ทำให้นับจำนวนจุดได้เร็วขึ้นด้วย `np.bincount()`
- Formula: `index = row * width + col`

**ตัวอย่าง:**
```
Grid ขนาด 320×320
Point ที่ (row=160, col=80)
→ linear_idx = 160 * 320 + 80 = 51,280
```

### 4. การนับจำนวนจุดด้วย bincount (CORE!)

```python
# จาก pointcloud_rasterization.py (บรรทัด ~165-167)

# นับจำนวนจุดในแต่ละ pixel (OPTIMIZED!)
flat_density = np.bincount(linear_idx, minlength=height*width)
channels[:, :, 0] = flat_density.reshape(height, width)
```

**คำอธิบาย NumPy Functions:**

#### `np.bincount()` - นับความถี่ของแต่ละค่าใน array
```python
# Syntax: np.bincount(x, weights=None, minlength=0)

# ตัวอย่างเบื้องต้น:
arr = np.array([0, 1, 1, 2, 2, 2])
count = np.bincount(arr)
# = [1, 2, 3]
#    ↑  ↑  ↑
#   0  1  2  ← index (ค่าใน arr)
#   เจอ 1 ครั้ง, 2 ครั้ง, 3 ครั้ง

# ตัวอย่างกับ minlength:
arr = np.array([0, 2])
np.bincount(arr, minlength=5)
# = [1, 0, 1, 0, 0]  ← ยาว 5 ช่อง (บังคับด้วย minlength)
```

**ทำไมเร็ว?** เพราะเป็น C implementation ไม่ใช่ Python loop!

#### `np.reshape()` - เปลี่ยนรูปร่าง array
```python
# Syntax: array.reshape(new_shape)

# ตัวอย่าง:
arr_1d = np.array([1, 2, 3, 4, 5, 6])
arr_2d = arr_1d.reshape(2, 3)
# = [[1, 2, 3],
#    [4, 5, 6]]

arr_2d = arr_1d.reshape(3, 2)
# = [[1, 2],
#    [3, 4],
#    [5, 6]]

# ในโค้ด:
flat_density.reshape(height, width)
# จาก [height*width] → [height, width]
# จาก [102400] → [320, 320]
```

**วิธีการทำงานของ `np.bincount()`:**

1. **Input**: `linear_idx` = array ของ index ที่แต่ละจุดตกอยู่
   ```
   เช่น linear_idx = [100, 100, 100, 200, 200, 350, ...]
   ```

2. **Process**: นับความถี่ของแต่ละ index
   ```
   bincount([100, 100, 100, 200, 200, 350])
   = [0, 0, ..., 3, 0, ..., 2, 0, ..., 1, ...]
     ↑          ↑          ↑          ↑
   idx 0    idx 100   idx 200   idx 350
   ```

3. **Output**: Array ที่ index i เก็บจำนวนครั้งที่เจอ index i
   ```
   flat_density[100] = 3  (มี 3 จุดตกใน pixel นี้)
   flat_density[200] = 2  (มี 2 จุดตกใน pixel นี้)
   flat_density[350] = 1  (มี 1 จุดตกใน pixel นี้)
   ```

4. **Reshape**: แปลง 1D array กลับเป็น 2D grid
   ```python
   flat_density.reshape(height, width)
   # จาก [height*width] → [height, width]
   ```

### 5. การ Normalize (ถ้าต้องการ)

```python
# จาก pointcloud_rasterization.py (บรรทัด ~196-201)

if normalize:
    for c in range(num_channels):
        channel = channels[:, :, c]
        if channel.max() > 0:
            # Scale ให้เป็นช่วง [0, 1]
            channels[:, :, c] = (channel - channel.min()) / (channel.max() - channel.min())
```

**คำอธิบาย NumPy Functions:**

#### `.min()` และ `.max()` - หาค่าต่ำสุด/สูงสุด
```python
arr = np.array([3, 7, 1, 9, 2])

arr.min()  # = 1 (ค่าต่ำสุด)
arr.max()  # = 9 (ค่าสูงสุด)

# ใช้กับ axis:
arr_2d = np.array([[1, 5, 3],
                   [2, 8, 1]])

arr_2d.min()         # = 1 (ทั้ง array)
arr_2d.min(axis=0)   # = [1, 5, 1] (แต่ละ column)
arr_2d.min(axis=1)   # = [1, 1] (แต่ละ row)
```

**Min-Max Normalization สูตร:**
```
normalized = (value - min) / (max - min)

ตัวอย่าง: [10, 50, 100]
min = 10, max = 100

(10 - 10) / (100 - 10) = 0/90 = 0.00
(50 - 10) / (100 - 10) = 40/90 = 0.44
(100 - 10) / (100 - 10) = 90/90 = 1.00

→ [0.00, 0.44, 1.00]
```

**ก่อน normalize:**
```
density = [0, 5, 10, 15, ..., 100]  (จำนวนจุดจริง)
```

**หลัง normalize:**
```
density = [0.00, 0.05, 0.10, 0.15, ..., 1.00]  (scale เป็น 0-1)
```

### 6. การบันทึกเป็นภาพ PNG

```python
# จาก pointcloud_rasterization.py (บรรทัด ~224-234)

for c, name in enumerate(channel_names):
    channel = multi_channel_grid[:, :, c]
    
    # Normalize เป็น 0-255 สำหรับภาพ
    if channel.max() > channel.min():
        channel_normalized = ((channel - channel.min()) / 
                            (channel.max() - channel.min()) * 255).astype(np.uint8)
    else:
        channel_normalized = np.zeros_like(channel, dtype=np.uint8)
    
    # บันทึกเป็น .png
    img_path = os.path.join(channel_dir, f"{name}.png")
    cv2.imwrite(img_path, channel_normalized)
```

**คำอธิบาย NumPy Functions:**

#### `np.zeros_like()` - สร้าง array เต็มด้วย 0 ขนาดเท่ากับ array อ้างอิง
```python
# Syntax: np.zeros_like(array, dtype=None)

original = np.array([[1, 2, 3],
                     [4, 5, 6]])

zeros = np.zeros_like(original)
# = [[0, 0, 0],
#    [0, 0, 0]]  ← รูปร่างเหมือน original

# กำหนด dtype:
zeros_uint8 = np.zeros_like(original, dtype=np.uint8)
# = [[0, 0, 0],
#    [0, 0, 0]]  ← แต่เป็น uint8 (0-255)
```

**ทำไมต้องใช้?** กรณีที่ channel มีค่าเดียวกันทั้งหมด (เช่น ทุก pixel = 0) จะหาร 0/0 ได้ NaN ดังนั้นใช้ zeros_like แทน

#### `np.uint8` - ชนิดข้อมูลจำนวนเต็มไม่มีติดลบ 8-bit (0-255)
```python
# ภาพ PNG ใช้ค่า 0-255 เท่านั้น
arr_float = np.array([0.0, 0.5, 1.0])
arr_uint8 = (arr_float * 255).astype(np.uint8)
# = [0, 127, 255]

# ถ้าไม่ convert จะผิดพลาด:
np.array([300]).astype(np.uint8)  # = [44] ← overflow!
np.array([-5]).astype(np.uint8)   # = [251] ← wraparound!
```

**การแปลงค่า:**
```
Normalized [0.0, 0.3, 0.6, 1.0]
    ↓ × 255
PNG values [0, 76, 153, 255]
```

## ตัวอย่างการทำงานทั้งหมด

### Input: Point Cloud
```
Point 1: (X=10.0, Y=20.0, Z=5.0)
Point 2: (X=10.1, Y=20.0, Z=5.5)
Point 3: (X=10.0, Y=20.1, Z=4.8)
Point 4: (X=15.0, Y=25.0, Z=6.0)
...
รวม 1,000,000 จุด
```

### Process:
```
1. แบ่ง grid 320×320 pixels
2. แปลงพิกัดเป็น index:
   - Point 1 → pixel (160, 80)
   - Point 2 → pixel (160, 80)  ← ตกใน pixel เดียวกัน!
   - Point 3 → pixel (161, 80)
   - Point 4 → pixel (200, 120)

3. นับจำนวนจุดด้วย bincount:
   - pixel (160, 80): 2 จุด
   - pixel (161, 80): 1 จุด
   - pixel (200, 120): 1 จุด
   - pixel อื่นๆ: 0 จุด

4. สร้าง density grid 320×320:
   grid[160, 80] = 2
   grid[161, 80] = 1
   grid[200, 120] = 1
   grid[...] = 0

5. Normalize (ถ้าต้องการ):
   max_density = 50 (pixel ที่มีจุดมากที่สุด)
   grid_normalized[160, 80] = 2/50 = 0.04
   grid_normalized[161, 80] = 1/50 = 0.02

6. บันทึกเป็นภาพ:
   density.png (grayscale 0-255)
```

### Output: Density Image
```
output/rasterization_result/
└── scans_test_optimized_1m_raster_channels/
    └── density.png  ← ภาพนี้!
```

## การแปลผลภาพ Density

### สีในภาพ
- **สีดำ (0)**: ไม่มีจุดเลย (empty)
- **สีเทา (50-150)**: มีจุดปานกลาง
- **สีขาว (255)**: มีจุดหนาแน่นมาก (dense)

### ความหมาย
1. **พื้นที่สีเข้ม**: พื้นที่ว่าง หรือไม่มีการสแกน
2. **พื้นที่สีสว่าง**: 
   - ต้นไม้หรือวัตถุที่มี point cloud หนาแน่น
   - พื้นที่ที่สแกนได้ดี overlap กันหลายรอบ
3. **เส้นขอบชัด**: ขอบของวัตถุหรือรอยต่อระหว่างพื้นที่

## ประโยชน์ของ Density Channel

### 1. แยกพื้นที่ว่างกับพื้นที่มีข้อมูล
```python
# หา threshold
density_threshold = 0.1

# แยกพื้นที่
has_data = density_channel > density_threshold
empty_area = density_channel <= density_threshold
```

### 2. หาต้นไม้ (ใช้ร่วมกับ channel อื่น)
```python
# ต้นไม้มักมีความหนาแน่นสูง + ความสูงสูง
tree_candidates = (density > 0.5) & (z_max > 3.0) & (hag_mean > 2.0)
```

### 3. ประเมินคุณภาพการสแกน
```python
# พื้นที่ที่สแกนไม่สมบูรณ์
low_quality = density < 0.2
print(f"Low quality area: {low_quality.sum() / density.size * 100:.1f}%")
```

### 4. ใช้เป็น Input สำหรับ YOLO
```python
# Multi-channel input สำหรับ CNN
input_tensor = np.stack([
    density,      # Channel 0: ความหนาแน่น
    z_mean,       # Channel 1: ความสูงเฉลี่ย
    hag_mean      # Channel 2: ความสูงเหนือพื้น
], axis=-1)  # Shape: (320, 320, 3)
```

## NumPy Functions Reference (สรุป)

### การจัดการ Array Structure

#### `np.vstack()` - Stack arrays แนวตั้ง (vertical)
```python
# จาก load_pointcloud: รวม X, Y, Z เป็น Nx3
x = np.array([1, 2, 3])
y = np.array([4, 5, 6])
z = np.array([7, 8, 9])

points = np.vstack((x, y, z)).T
# ขั้นตอน:
# 1. vstack: [[1, 2, 3],    ← x
#             [4, 5, 6],    ← y
#             [7, 8, 9]]    ← z
#
# 2. .T (transpose): [[1, 4, 7],  ← point 1
#                     [2, 5, 8],  ← point 2
#                     [3, 6, 9]]  ← point 3
```

#### `np.zeros()` - สร้าง array เต็มด้วย 0
```python
np.zeros((3, 4))
# = [[0, 0, 0, 0],
#    [0, 0, 0, 0],
#    [0, 0, 0, 0]]

np.zeros((320, 320, 6), dtype=np.float32)
# = array ขนาด 320×320×6 ทุกค่าเป็น 0.0
```

#### `np.arange()` - สร้างลำดับเลข
```python
np.arange(5)       # = [0, 1, 2, 3, 4]
np.arange(2, 7)    # = [2, 3, 4, 5, 6]
np.arange(0, 10, 2) # = [0, 2, 4, 6, 8] (step=2)
```

### การจัดการค่าพิเศษ

#### `np.nan_to_num()` - แปลง NaN/Inf เป็นเลขปกติ
```python
arr = np.array([1.0, np.nan, np.inf, -np.inf, 5.0])
np.nan_to_num(arr)
# = [1.0, 0.0, 1.79e+308, -1.79e+308, 5.0]
#         ↑    ↑          ↑
#       NaN→0  Inf→large  -Inf→large negative

# ในโค้ด: ป้องกัน error จากการหาร 0/0
np.nan_to_num(arr, 0)  # แทน NaN/Inf ด้วย 0
```

#### `np.errstate()` - ปิด warning ชั่วคราว
```python
# กรณีปกติ: หาร 0 จะมี warning
a = np.array([1, 0])
b = 1 / a  # RuntimeWarning: divide by zero

# ใช้ errstate ปิด warning:
with np.errstate(divide='ignore', invalid='ignore'):
    b = 1 / a  # ไม่มี warning
# = [1., inf]
```

### การจัดการ Index

#### `np.where()` - หา index ที่ตรงเงื่อนไข
```python
arr = np.array([1, 5, 3, 8, 2])
indices = np.where(arr > 3)
# = (array([1, 3]),)  ← index 1 และ 3 มีค่า > 3

# ใช้ 2D:
arr_2d = np.array([[1, 5],
                   [3, 8]])
y, x = np.where(arr_2d > 3)
# y = [0, 1]  ← row indices
# x = [1, 1]  ← col indices
# ตำแหน่ง: (0,1)=5 และ (1,1)=8
```

### SciPy Functions ที่ใช้

#### `ndimage.maximum()` / `ndimage.minimum()` - หาค่า max/min แบบ grouped
```python
from scipy import ndimage

values = np.array([5, 10, 3, 8, 2, 7])
indices = np.array([0, 0, 1, 1, 2, 2])

# หา max แต่ละกลุ่ม:
max_vals = ndimage.maximum(values, indices, index=[0, 1, 2])
# = [10, 8, 7]
#    ↑   ↑  ↑
#  group 0: max(5,10)=10
#  group 1: max(3,8)=8
#  group 2: max(2,7)=7
```

**ใช้ใน rasterization:**
```python
# หา z_max แต่ละ pixel
z_max = ndimage.maximum(z_values, linear_idx, 
                        index=np.arange(height*width))
```

## Optimization ที่สำคัญ

### เดิม (Slow):
```python
# Loop ทุก pixel - ช้ามาก!
for i in range(height):
    for j in range(width):
        mask = (y_idx == i) & (x_idx == j)
        density[i, j] = mask.sum()  # นับจุดที่ตก pixel นี้
```
**เวลา**: ~30-60 วินาที สำหรับ 1M points

### ใหม่ (Fast):
```python
# Vectorized - เร็วมาก!
flat_density = np.bincount(linear_idx, minlength=height*width)
density = flat_density.reshape(height, width)
```
**เวลา**: ~0.5-1 วินาที สำหรับ 1M points

**เร็วขึ้น 30-60 เท่า!** 🚀

## สรุป

**Density Image** = จำนวนจุดในแต่ละ pixel ของ 2D grid

**ขั้นตอนหลัก:**
1. แปลงพิกัด 3D → 2D grid indices
2. ใช้ `np.bincount()` นับจำนวนจุดในแต่ละ pixel (vectorized!)
3. Reshape เป็น 2D array
4. Normalize และบันทึกเป็นภาพ

**ไฟล์ Output:**
- `output/rasterization_result/.../density.png` - ภาพ grayscale
- `output/rasterization_result/..._raster.npy` - ข้อมูลแบบ raw (multi-channel)

**การใช้งาน:**
- แยกพื้นที่ว่าง/มีข้อมูล
- ประเมินคุณภาพการสแกน
- Input สำหรับ Machine Learning (YOLO)

---

## ตัวอย่างการใช้งาน

### ดูภาพ Density
```bash
# บน Mac/Linux
open output/rasterization_result/scans_test_optimized_1m_raster_channels/density.png
```

### Load และวิเคราะห์
```python
import numpy as np
import matplotlib.pyplot as plt

# Load multi-channel data
data = np.load('output/rasterization_result/scans_test_optimized_1m_raster.npy')
density = data[:, :, 0]  # Channel 0 = density

# แสดงสถิติ
print(f"Min density: {density.min()}")
print(f"Max density: {density.max()}")
print(f"Mean density: {density.mean():.2f}")
print(f"Pixels with data: {(density > 0).sum():,}")

# Plot histogram
plt.hist(density[density > 0].flatten(), bins=50)
plt.xlabel('Density (normalized)')
plt.ylabel('Frequency')
plt.title('Density Distribution')
plt.show()
```

## คำถามที่พบบ่อย

**Q: ทำไมบางพื้นที่เป็นสีดำ?**  
A: ไม่มีจุด point cloud ตกในพื้นที่นั้น (ไม่ได้สแกน หรืออยู่นอกขอบเขต)

**Q: Normalize กับไม่ Normalize ต่างกันอย่างไร?**  
A: 
- ไม่ normalize: เก็บจำนวนจุดจริง (0, 5, 10, ..., 100)
- Normalize: scale เป็น 0-1 (0.0, 0.05, 0.10, ..., 1.0)

**Q: bincount ทำงานเร็วขนาดไหน?**  
A: เร็วมาก! สำหรับ 1M points ใช้เวลาแค่ ~0.5 วินาที (เทียบกับ loop ที่ใช้ 30+ วินาที)

**Q: ใช้ Density อย่างเดียวพอไหม?**  
A: ไม่พอ ควรใช้ร่วมกับ channel อื่น เช่น z_max, hag_mean เพื่อแยกต้นไม้ได้แม่นยำขึ้น
