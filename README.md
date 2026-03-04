# ESP32 闭环式 CO2 监测系统（带漂移补偿）

[English Documentation](README_EN.md)

基于气泵的闭环 CO2 监测系统，适用于密封腔体（如微生物培养瓶），内置多锚点漂移补偿算法弥补低成本装置的气密性导致的漏气问题。

## 效果演示

| 原始数据（泄漏导致持续下降） | 补偿后数据（还原真实波动） |
| ---------------------------- | -------------------------- |
| ![原始](docs/images/before_compensation.png) | ![补偿后](docs/images/after_compensation.png) |

## 核心特性

### 气泵闭环采样

PWM 控制气泵进行闭环气路采样：

1. 气泵循环（可调时长和强度）
2. 停泵，等待气压稳定
3. 读取传感器数据
4. 实时上传服务器

### 自动启动运行

- 上电自动启动，无需人工干预
- 多 WiFi 热点支持，自动切换
- 断电恢复后自动继续
- 时间戳对齐到整 10 分钟

### 时间戳对齐机制

系统确保数据时间戳整齐一致：

- 所有数据时间戳对齐到整 10 分钟（`00:00`, `00:10`, `00:20`...）
- 虽然测量需要约 45 秒，但上传的时间戳是**计划时间**，而非完成时间
- 断电恢复后，系统计算下一个最近的对齐时间
- 如果距离下一个间隔不足 10 秒，则跳到再下一个

### 多锚点漂移补偿算法

**问题**：密封气路不可避免会随时间泄漏，导致 CO2 读数持续下降。

**解决方案**：前端实现的非线性漂移补偿算法：

1. **锚点选择**：用户选择多个 CO2 应该稳定的"稳定期"（如夜间藻类呼吸稳定时段）
2. **斜率计算**：对每个锚点时段进行线性回归，得到漂移速率（ppm/h）
3. **插值过渡**：锚点之间线性插值，实现漂移速率平滑过渡
4. **累积修正**：对漂移速率积分，从原始读数中减去

**数学表达**：

- 锚点漂移速率（线性回归斜率）：
  
  $$slope = \frac{n \sum x_i y_i - \sum x_i \sum y_i}{n \sum x_i^2 - (\sum x_i)^2}$$

- 修正值（累积积分）：
  
  $$CO_{2,corrected} = CO_{2,raw} - \int_0^t slope(\tau) \, d\tau$$

### 完整物联网数据流

```
ESP32 采样 → HTTP 上传 → PHP 存储 → ECharts 可视化
```

## 硬件需求

| 组件 | 型号 | 说明 |
| ---- | ---- | ---- |
| 主控 | ESP32-S3-DevKitC-1 | 主控制器 |
| CO2 传感器 | ZG09SR | NDIR 红外，Modbus RTU |
| 气泵 | 5V 微型气泵 | PWM 控制，闭环采样 |
| 气泵驱动 | MOS 驱动模块（四脚） | PWM 信号放大 |
| 温湿度 | DHT22/SHT30 等 | **预留接口，暂未使用** |

### 引脚连接

| ESP32 引脚 | 连接设备 |
| ---------- | -------- |
| GPIO 18 | MOS 驱动模块 PWM（气泵控制） |
| GPIO 16 | ZG09SR RX |
| GPIO 17 | ZG09SR TX |

详见 [硬件文档](docs/HARDWARE.md) 获取完整接线说明。

## 快速开始

### 1. 固件配置

```bash
cd firmware
cp src/config.h.example src/config.h
# 编辑 config.h，填入你的 WiFi 和服务器信息
```

使用 PlatformIO 编译上传：

```bash
pio run -t upload
```

### 2. 服务器部署

```bash
cd web
cp config.php.example config.php
# 编辑 config.php，填入数据库信息
```

导入示例数据（包含表结构和 1 月 3-9 日测试数据）：

```bash
mysql -u your_user -p co2data < data/sample_data.sql
```

或手动创建空表：

```sql
CREATE TABLE co2_measurements (
  id INT AUTO_INCREMENT PRIMARY KEY,
  timestamp DATETIME NOT NULL,
  co2_ppm INT NOT NULL,
  temperature FLOAT DEFAULT NULL,  -- 预留，暂未使用
  humidity FLOAT DEFAULT NULL,     -- 预留，暂未使用
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_timestamp (timestamp)
);
```

### 3. 使用漂移补偿

1. 浏览器打开 `index.html`
2. 选择时间范围查询数据
3. 在"漂移补偿"面板添加锚点时段（选择 CO2 应该稳定的时段）
4. 点击"应用漂移补偿"查看效果

## 串口命令

| 命令 | 功能 |
| ---- | ---- |
| auto | 恢复自动监测模式 |
| stop | 暂停自动，进入手动模式 |
| single | 单次测量（手动模式不上传） |
| vent | 启动通风功能 |
| status | 查看当前状态 |

## 配置参数

在 `config.h` 中可调整：

| 参数 | 默认值 | 说明 |
| ---- | ------ | ---- |
| PUMP_DUTY_MEASURE | 120 | 采样气泵 PWM 占空比 (0-255) |
| PUMP_DUTY_VENT | 150 | 通风气泵 PWM 占空比 |
| PUMP_TIME_MEASURE | 25 | 采样气泵运行时间（秒） |
| PUMP_TIME_VENT | 50 | 通风气泵运行时间（秒） |
| SENSOR_STABILIZE_TIME | 20 | 气压稳定等待时间（秒） |
| INTERVAL_SECONDS | 600 | 测量间隔（固定10分钟，对齐整点）* |

> *注意：修改 `INTERVAL_SECONDS` 需同时修改对齐逻辑，参见源码中 `getNextAlignedEpoch()` 函数。

## 项目结构

```
esp32-co2-drift-compensator/
├── firmware/              # ESP32 固件
│   ├── src/
│   │   ├── main.cpp
│   │   └── config.h.example
│   └── platformio.ini
├── web/                   # Web 前端 + PHP 后端
│   ├── index.html         # 数据可视化页面
│   ├── get_data.php       # 数据查询 API
│   ├── sensor_data.php    # 数据接收 API
│   └── config.php.example
├── data/
│   └── sample_data.sql    # 示例数据（1月3-9日测试）
├── docs/
│   ├── HARDWARE.md        # 硬件接线指南
│   └── images/
└── README.md
```

## 应用场景

本项目最初为监测密封腔体内藻类呼吸而开发，但该项目的一些思路适用于其他存在以下情况的场景：

- 气路密封性较差
- 需要长期连续监测 CO2 或其他气体
- 传感器长期漂移问题

## 技术栈

- 固件：PlatformIO + Arduino 框架
- 传感器：ZG09SR（Modbus RTU）
- 前端：ECharts + 原生 JS
- 后端：PHP + MySQL
- 硬件：ESP32-S3 + PWM 气泵控制

## 许可证

MIT License - 详见 [LICENSE](LICENSE)
