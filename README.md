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
