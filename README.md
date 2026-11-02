# ESP32-S3 表情包时钟

基于 ESP32-S3、1.54 寸 240x240 IPS 屏（ST7789）和 LVGL 9 的小桌面时钟。
默认显示模拟表盘 + 数字时间，按 BOOT 键可切换到 8 张图片（表情包）轮播。

## 功能

- 模拟表盘：时/分/秒指针、数字时间、日期、WiFi/NTP 状态
- 图片轮播：`1.png` ~ `8.png` 编译进固件，每 5 秒切换一张
- 网页配网：首次上电自动开热点 `ESP-Clock-xxxx`，手机访问 `http://192.168.4.1` 填写 WiFi
- 自动对时：联网后通过 `ntp.aliyun.com` 同步时间，显示东八区时间
- 断网也可用：未对时前用开机计时显示时间，指针照常走动

## 硬件

| 屏幕引脚 | ESP32-S3 | 说明 |
| --- | --- | --- |
| GND | GND | 地 |
| VCC | 3V3 | 3.3V 供电 |
| SCL | GPIO12 | SPI2 SCK |
| SDA | GPIO11 | SPI2 MOSI |
| RES | GPIO8 | 复位 |
| DC | GPIO9 | 数据/命令 |
| CS | GPIO10 | 片选 |
| BLK | GPIO14 | 背光（高电平点亮） |

- SPI 时钟 40MHz，屏幕分辨率 240x240，RGB565
- 开发板需为 N16R8（16MB Flash + 8MB PSRAM）

## 目录结构

```
.
├── CMakeLists.txt              # 工程入口（project: ESP32-S3-LVGL）
├── partitions.csv              # 自定义分区表，app 分区 4MB
├── sdkconfig.defaults          # LVGL / 16MB Flash / 自定义分区默认配置
├── 1.png ~ 8.png               # 轮播图片源文件（240x240 效果最佳）
├── main/
│   ├── ESP32-S3-LVGL.c         # app_main、SPI/ST7789 初始化、可选屏幕自检 demo
│   ├── clock_ui.c/.h           # LVGL 表盘、图片轮播、BOOT 按键
│   ├── wifi_mgr.c/.h           # WiFi 配网、Captive Portal、SNTP 对时
│   ├── CMakeLists.txt          # 组件依赖 + 图片 bin 打包
│   └── images/                 # 由脚本生成的 img1.bin ~ img8.bin（不提交）
└── tools/
    └── png_to_rgb565.py        # PNG -> RGB565 转换脚本（仅用标准库）
```

## 编译与烧录

环境要求：ESP-IDF 5.x（已验证 5.5.6）、Python 3、LVGL 9.2（由组件管理器自动拉取）。

```bash
# 1. 生成图片二进制（main/images/img1.bin ~ img8.bin，每张 115200 字节）
python3 tools/png_to_rgb565.py

# 2. 编译、烧录、串口监视
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/tty.usbmodemXXXX flash monitor
```

`main/images` 中的 bin 文件被 `.gitignore` 忽略，首次编译前必须执行第 1 步，否则 CMake 会因找不到图片文件报错。

## 使用

### 配网

1. 首次上电（或 WiFi 连接失败）时，屏幕显示 `SETUP: 192.168.4.1` 和热点名 `AP: ESP-Clock-xxxx`
2. 手机连接该热点，浏览器打开 `http://192.168.4.1`
3. 填写 WiFi 名称与密码并保存，设备自动重启并连接路由器

### BOOT 键

| 操作 | 效果 |
| --- | --- |
| 短按（≥100ms，<5s） | 表盘 ↔ 图片轮播切换，轮播每 5 秒换一张 |
| 长按 5 秒 | 清除已保存的 WiFi 配置并重启，回到配网模式 |

## 常用配置

均在 [main/ESP32-S3-LVGL.c](main/ESP32-S3-LVGL.c) 顶部：

| 宏 | 默认 | 说明 |
| --- | --- | --- |
| `LCD_PIN_*` | 12/11/8/9/10/14 | SPI 与屏幕控制引脚 |
| `LCD_PCLK_HZ` | 40MHz | 花屏或不稳定时降到 20MHz |
| `LCD_X_GAP` / `LCD_Y_GAP` | 0 / 0 | 画面整体偏移时调整；上下出现雪花通常需设 80 |
| `LCD_INVERT` | 1 | IPS 屏一般需反色，颜色像负片时改 0 |
| `LCD_BGR` | 1 | 红蓝互换时改 0 |
| `DEMO_ENABLE` | 0 | 改为 1 进入屏幕自检 demo（彩条 / 网格 / 动画 / 特效），不再启动 LVGL |

其他：

- 轮播图片数量与切换间隔：`main/clock_ui.c` 中的 `IMG_COUNT`、`lv_timer_create(..., 5000, ...)`
- LVGL 字体与内存：`sdkconfig.defaults`（`CONFIG_LV_FONT_MONTSERRAT_28`、`CONFIG_LV_MEM_SIZE_KILOBYTES`）
- Flash 与分区：`sdkconfig.defaults` + `partitions.csv`（8 张图占固件约 900KB，app 分区留 4MB）

## 替换轮播图片

1. 用同名 `1.png` ~ `8.png` 覆盖根目录图片（支持 RGB/RGBA/灰度/调色板 PNG，8bit 且非隔行）
2. 重新执行 `python3 tools/png_to_rgb565.py`
3. `idf.py build flash`

脚本会等比缩放并居中裁切到 240x240，透明区域按黑底合成。

## 常见问题

| 现象 | 排查 |
| --- | --- |
| 编译报找不到 `images/img*.bin` | 先运行 `tools/png_to_rgb565.py` |
| 上电花屏 / 颜色异常 | 调整 `LCD_Y_GAP`、`LCD_INVERT`、`LCD_BGR`，或降低 SPI 时钟 |
| 屏幕只显示部分画面 | `LCD_Y_GAP` 设置错误（240x320 显存的面板常见） |
| 图片显示红蓝互换 | 把 `LCD_BGR` 改为 0 |
| 固件超出 app 分区 | 减少图片数量或压缩图片，确认 `partitions.csv` 为 4MB app |
| 配网页面打不开 | 确认手机连接的是 `ESP-Clock-xxxx` 热点，访问 `http://192.168.4.1` |