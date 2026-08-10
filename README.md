# openvela 智能手环原生 UI（Gemini S1 2.8 英寸）

本目录是面向润芯微 Gemini S1 2.8 英寸开发板的独立 openvela 原生应用工程快照。项目以快应用版智能手环界面为视觉与交互参考，将核心页面和业务流程迁移到 NuttX/openvela、LVGL 和 Allwinner R528 板级环境。

这不是快应用 RPK，也不是在 DshanPi checkout 中切换出来的临时目标。本目录拥有独立的 Gemini 应用分支、板级配置、128 MiB NAND 打包输入/输出、网络代理和最终镜像；编译、链接和 pack 都必须在本目录完成。

> 本 README 只适用于 Gemini S1 2.8 英寸板。不要把本目录生成的镜像烧录到 DshanPi 256 MiB NAND 板，也不要把 DshanPi 的 `nsh.fex`、`out` 或 lunch 环境复制进来。

## 1. 适用硬件与软件基线

| 项目 | 本工程范围 |
| --- | --- |
| 目标板 | 润芯微 Gemini S1，板级目标 `r528s3-gemini-s1` |
| SoC | Allwinner R528 |
| 显示 | 2.8 英寸 ILI9341 SPI，320 × 240 横屏 |
| 触摸 | GT911 I2C 电容触摸 |
| 存储 | 128 MiB NAND |
| 系统 | NuttX/openvela，基线来自 `dev-ai-contest-2026` |
| 应用框架 | LVGL + NuttX framebuffer/input + libuv |
| 构建配置 | `vendor/allwinnertech/boards/r528/r528s3-gemini-s1/configs/openvela_ui_native` |
| 打包目标 | `r528s3-gemini-s1` / `gemini-s1_nand` |
| 串口 | 1,500,000 baud，Flow Control 为 `none` |

本目录是可构建源码快照，不包含可用于 `repo sync` 的完整 manifest 历史。GitHub 仓库不直接提交约 14 GiB 的通用 `prebuilts/`；首次构建前运行随仓库提供的脚本，从 openvela 官方仓库取得本目标使用的 Linux x86_64 工具。不要在此快照上执行 `repo sync -d`、`git clean -fdx` 或用主体板 vendor 覆盖 Gemini BSP。完整 vendor 依赖树仍会保留未参与 Gemini lunch/pack 的其他板级源码；“隔离”指 Gemini 的配置、`data/usrdata`、`nsh.fex`、`out` 和镜像不得与其他板交叉。

## 2. 当前实现范围

主干页面按环形顺序切换：

```text
主页 → 天气 → 运动 → 健康 → 音乐 → 通知 → 主页
```

已实现：

- 六页循环横向滑动和主页/天气/运动/健康/音乐详情入口。
- 按 432 × 514 逻辑坐标生成的主题、动作与轻量动画。
- 五个固定城市的天气实时数据和三日预报。
- 运动步数、热量、时长、心率、24 小时统计、目标设置和完成提示。
- 健康心率与血压演示测量以及按日同步记录。
- NxPlayer 原始 PCM 音乐播放、暂停、音量和曲目列表。
- Wi-Fi 天气代理、NTP/HTTP Date 校时以及健康/运动 Wi-Fi 上传。

与主体板不相同的范围：

- Gemini 使用 320 × 240 横屏。432 × 514 设计画布等比缩放为约 201 × 239 并居中，左右留黑是预期显示效果。
- 天气只有北京、上海、广州、深圳、武汉五个基础城市，没有自定义城市滚轮、添加、删除和 `cities.conf`。
- 本工程不编译 `openvela_ui_power.c`，没有主体板的 60 秒应用级待机、触摸消费或抬腕唤醒策略。
- 本版本未接入可用 BMI160/IMU、真实计步、心率或血压数据源。
- Gemini 的板级显示、触摸、NAND 分区和启动脚本均独立于 DshanPi，不能仅靠整体缩放替代板级适配。

共同功能边界：

- AI 助手页已移除。
- 通知页只是主干视觉入口，没有消息列表、手机接入或蓝牙推送。
- 电量是固定展示，未接入电池计量。
- 无 GPS/定位，“当前位置”已从天气流程移除。
- 健康与运动默认数据仅用于演示，不能用于医疗判断或真实运动评估。
- 原快应用的连续高心率判定与告警行为均未实现，当前只有提示文字。

## 3. 与快应用参考工程的区别

快应用参考工程由 `src/app.ux` 保存全局状态，`manifest.json` 声明路由和系统能力，页面位于 `src/pages/**`，服务位于 `src/common/**`。Gemini 原生版本是一个 NuttX 进程和一棵 LVGL 对象树。

| 快应用实现 | Gemini 原生实现 |
| --- | --- |
| JavaScript/UX、RPK | C、NuttX ELF 和 128 MiB NAND `.img` |
| `router.push()` 和页面生命周期 | 主干 tile 常驻；详情 overlay 进入时创建、正常退出时销毁 |
| DOM 手势 | LVGL input 位移判定和异步切换 |
| `app.ux` 跨路由状态 | `openvela_ui.c` 全局 context 和模块状态机 |
| `@system.audio` | NxPlayer + `/dev/audio/pcm0p` |
| `@system.storage` | `/data/etc/openvela_ui/` 文件 |
| `@system.geolocation` | 五个固定城市 |
| interconnect/HTTP | 当前仅 Wi-Fi HTTP(S) 同步 |
| ImageAnimator | LVGL animimg、timer、属性动画 |
| 七个主干页含 AI | 六个主干页，不含 AI |

### 3.1 迁移时的逻辑变化

1. **路由变为常驻主干和按需详情。** 六个真实主干页加首尾克隆页组成 tileview；详情 overlay 在进入时创建、正常退出时销毁，同一次会话内通过子视图状态切换。
2. **环形滑动只保留一份业务状态。** 到达克隆页后无动画跳回真实页，克隆页只承担视觉衔接。
3. **纵向手势统一分发。** 输入释放时根据水平/垂直位移确定动作，切换通过 `lv_async_call()` 延迟执行，避免事件回调中删除控件造成数据访问异常。
4. **设计坐标与物理屏解耦。** 业务布局统一使用 `scale_1000`，Gemini 只在顶层计算缩放和居中；小屏不会重新定义全部页面坐标。
5. **动画受 SPI 和内存约束。** 共享图片动画解码器，减少透明大对象移动；重图片延后显示。专项运动临时返回主页时会隐藏并保留当前 sport 对象，正常退出则写盘并销毁。
6. **网络与 LVGL 隔离。** 天气 worker 和同步 worker 不访问 LVGL；UI 线程从不可变快照刷新页面。
7. **天气取消定位。** 板端只传固定城市 ID/名称到代理，QWeather 密钥留在电脑端。
8. **音频预转换。** MP3 素材在主机侧转为 S16LE、24 kHz、单声道 PCM，板端直接通过 NxPlayer 输出。
9. **持久化取代快应用 storage。** 运动状态、健康日文件、同步游标和 outbox 写入 `/data`，关键同步文件使用临时文件、`fsync` 和 rename。
10. **同步只保留 Wi-Fi 路径。** 原先设想的蓝牙/interconnect 没有迁移；当前协议使用 begin/chunk/commit、768 字符分片和 FNV-1a 校验。

## 4. 运行架构

```mermaid
flowchart TD
    R[Gemini rcS] --> W[Wi-Fi / DHCP / NTP]
    R --> M[openvela_ui_main]
    M --> L[LVGL + libuv UI 线程]
    L --> U[六页 tileview / overlay]
    L --> S[运动状态机]
    M --> T[HTTP Date 后备]
    M --> Y[天气 worker]
    M --> D[同步 worker]
    L --> A[NxPlayer worker]
    Y --> H[电脑天气/同步代理]
    D --> H
    A --> C[/dev/audio/pcm0p]
    U --> F[/data 资源与状态]
```

`openvela_ui_main.c` 的应用初始化顺序为：

```text
lv_init
→ lv_nuttx_init
→ openvela_ui_create
→ openvela_ui_timesync_start
→ openvela_ui_sync_start
→ LVGL/libuv event loop
```

Gemini 没有 `openvela_ui_power_init()`。所有 LVGL 操作必须发生在 UI 线程；后台网络、同步或未来的传感器线程只能发布数据，再由 UI 线程消费。

### 4.1 当前板级启动链说明

当前 Gemini 板级 `rcS.nsh` 会启动 `mediad`、NTP 和一次 Wi-Fi 脚本，外层 `rcS` 又会启动 Wi-Fi 和 `openvela_ui`；Wi-Fi helper 在 DHCP 成功后还可能再次请求 NTP。因此单次启动周期内会执行重复的网络/校时启动路径，并存在 mediad 与 NxPlayer 争用 PCM 的风险。

不要在 NSH 中再次手工启动 `openvela_ui`、Wi-Fi、NTP 或 mediad。重启可用于恢复异常状态，但重启后的单次启动周期内仍可能看到脚本自身产生的重复 Wi-Fi/NTP 日志。该启动链是 Gemini 当前镜像的已知限制，不代表主体板也有同样行为。

## 5. 代码目录

| 路径 | 作用 |
| --- | --- |
| `packages/demos/openvela_ui/openvela_ui_main.c` | 进程入口和服务生命周期 |
| `packages/demos/openvela_ui/openvela_ui.c` | 主干 UI、手势、天气/健康/音乐详情 |
| `packages/demos/openvela_ui/openvela_ui_sport.c` | 运动状态机和统计页面 |
| `packages/demos/openvela_ui/openvela_ui_weather.c` | Gemini 天气 worker 和快照 |
| `packages/demos/openvela_ui/openvela_ui_sync.c` | 持久化、控制轮询和分块上传 |
| `packages/demos/openvela_ui/openvela_ui_timesync.c` | HTTP Date 后备校时 |
| `tools/weather_proxy/` | 随 Gemini 工程提供的 Node.js 天气/同步代理 |
| `vendor/allwinnertech/boards/r528/r528s3-gemini-s1/` | Gemini 板级配置 |
| `vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/` | 128 MiB NAND 打包配置与素材 |
| `release/8.3/` | 已验证 Gemini 8.3 镜像和校验文件 |

Gemini 目录没有主体板的 `openvela_ui_power.c` 和 `openvela_ui_city_data.c`。后续若同步主体功能，必须明确移植模块、Kconfig、板级资源和数据，而不是直接覆盖整个 `openvela_ui` 目录。

快应用到原生的代码映射：

| 快应用位置/能力 | 原生承接位置 |
| --- | --- |
| `src/app.ux` | `openvela_ui.c` context + 各模块状态 |
| `src/manifest.json` | tile/overlay 枚举和手势路由 |
| `src/pages/**` | `openvela_ui.c` 与 `openvela_ui_sport.c` |
| `src/common/**` | weather/sync/timesync 模块 |
| 快应用资源 | `data/usrdata/openvela_ui/` 打包素材 |

## 6. 模块接口与对接过程

### 6.1 UI 生命周期

```c
void openvela_ui_create(void);
```

该函数只在 display/input 初始化成功后调用一次。Gemini 没有全局低功耗入口；不要照抄主体板的 `openvela_ui_power_init()` 调用。

### 6.2 天气接口

```c
int openvela_ui_weather_start(void);
int openvela_ui_weather_request(const char *location_id,
                                const char *location,
                                const char *administrative_area);
int openvela_ui_weather_snapshot(
    struct openvela_ui_weather_snapshot_s *snapshot);
```

对接顺序：

1. 应用创建一个天气 worker。
2. UI 从五个固定城市中选择一项并提交请求。
3. worker 读取 `/data/etc/openvela_ui/weather.conf`，通过 IPv4 HTTP 访问电脑代理。
4. 代理使用主机端 QWeather 凭据查询第三方 HTTPS API。
5. worker 发布不可变快照；UI 按 `revision` 复制并刷新控件。

板端只保存非秘密代理地址，`weather.conf` 必须使用 `http://`：

```ini
proxy_url=http://YOUR_PUBLIC_HOST/api/weather
```

### 6.3 运动/传感器接口

运动模块主要接口：

```c
openvela_ui_sport_t *openvela_ui_sport_create(...);
void openvela_ui_sport_shown(openvela_ui_sport_t *sport);
void openvela_ui_sport_hidden(openvela_ui_sport_t *sport);
bool openvela_ui_sport_gesture(openvela_ui_sport_t *sport,
                               int32_t delta_x, int32_t delta_y);
void openvela_ui_sport_set_data(openvela_ui_sport_t *sport,
                                uint32_t steps, uint16_t heart_rate);
void openvela_ui_sport_set_event_cb(...);
```

`openvela_ui_sport_set_data()` 是未来接入真实数据的边界，不会自动读取 BMI160。驱动线程应先把数据写入线程安全缓冲，再由 LVGL timer 或 `lv_async_call()` 在 UI 线程调用；不要从 sensor worker 直接访问 LVGL。

### 6.4 健康与同步接口

```c
int openvela_ui_sync_start(void);
void openvela_ui_sync_stop(void);
void openvela_ui_sync_record_heart_rate(uint16_t bpm);
void openvela_ui_sync_record_blood_pressure(uint16_t systolic,
                                             uint16_t diastolic,
                                             uint16_t pulse);
```

完整测量结束后才记录健康结果。同步 worker 等日期有效和 DNS 可用后轮询控制 URL，读取运动状态与健康日文件并写入 outbox。收到服务端精确的 `ok` 或 `duplicate` 才删除本地上传文件。

主要存储位置：

```text
/data/etc/openvela_ui/sport_state.conf
/data/etc/openvela_ui/sync/health/YYYYMMDD.conf
/data/etc/openvela_ui/sync/outbox/*.json
/data/etc/openvela_ui/sync/state.conf
```

当前只实现 Wi-Fi/libcurl 上传，没有 BLE GATT 数据通道。

### 6.5 音频接口

曲目格式：

```text
/data/openvela_ui/music/tracks/track-*.pcm
signed 16-bit little-endian / 24000 Hz / mono
```

应用使用 NxPlayer 和 `/dev/audio/pcm0p`。播放状态属于全局 UI context，关闭详情后仍可保持，重进时刷新同一状态。

播放中直接切歌仍可能因为异步 stop/new worker 竞争产生 XRUN、`-EPIPE` 或静音；Gemini 启动链中的 mediad 还可能增加 PCM 争用。建议使用“暂停 → 切歌 → 播放”，异常时重启。

## 7. 网络与配置

构建前只编辑 Gemini 的打包目录：

```text
vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/data/usrdata/etc/wifi/wapi.conf
vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/data/usrdata/etc/openvela_ui/weather.conf
vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/data/usrdata/etc/openvela_ui/sync.conf
```

上述三个 Gemini 打包配置已使用占位符。WPA2-Personal 配置示例：

```json
{"wlan0":{"mode":2,"auth":4,"cmode":8,"alg":3,"ssid":"YOUR_WIFI_SSID","bssid":"00:00:00:00:00:00","psk":"YOUR_WIFI_PASSWORD"}}
```

同步配置：

```ini
enabled=1
endpoint=https://YOUR_PUBLIC_HOST/api/sync/frame
control_url=https://YOUR_PUBLIC_HOST/api/sync/control?deviceId=openvela-gemini-s1-01
device_id=openvela-gemini-s1-01
```

不同设备必须使用不同 `device_id`。Wi-Fi 密码会明文进入镜像，只应在可信环境中填写。

电脑代理要求 Node.js 12 或更高版本：

```bash
cd tools/weather_proxy
QWEATHER_API_HOST='YOUR_QWEATHER_HOST' \
QWEATHER_PROJECT_ID='YOUR_PROJECT_ID' \
QWEATHER_CREDENTIAL_ID='YOUR_CREDENTIAL_ID' \
QWEATHER_PRIVATE_KEY_PATH='/absolute/path/to/ed25519-private.pem' \
WEATHER_SERVER_PORT=8790 \
node server.js
```

检查代理：

```bash
curl http://127.0.0.1:8790/health
curl -G http://127.0.0.1:8790/api/weather \
  --data-urlencode 'location=北京' \
  --data-urlencode 'adm=北京'
curl http://127.0.0.1:8790/api/sync/status
```

需要跨不同 Wi-Fi 时可使用：

```bash
command -v cloudflared
cloudflared tunnel --url http://127.0.0.1:8790
```

Quick Tunnel 重启后地址通常改变，随后需要更新 Gemini 自己的 weather/sync 配置并重新 pack。当前接收端未验证可选 token，只适合比赛演示模拟数据，不应上传真实健康隐私。

## 8. 编译、链接与打包

推荐 Ubuntu 22.04 x86_64、Bash、8–16 GiB RAM 和至少 35–40 GiB 可用空间。工具链脚本会在 Gemini 工程内部生成独立的 `prebuilts/`，不依赖 DshanPi 目录。

详细系统依赖、PhoenixSuit 烧录和串口步骤见[Gemini S1 2.8 部署与用户使用手册](docs/gemini_s1_28_deployment_and_user_manual.md)。

预检：

```bash
cd /path/to/openvela-contest-gemini-s1-28
./tools/bootstrap_openvela_prebuilts.sh
test -x build.sh
test -x prebuilts/gcc/linux-x86_64/arm-none-eabi/bin/arm-none-eabi-gcc
test -f packages/demos/openvela_ui/openvela_ui_main.c
test -f vendor/allwinnertech/boards/r528/r528s3-gemini-s1/configs/openvela_ui_native/defconfig
test -f tools/weather_proxy/server.js
```

首次或移动目录后先清理旧配置，再完整构建：

```bash
cd /path/to/openvela-contest-gemini-s1-28
./build.sh \
  vendor/allwinnertech/boards/r528/r528s3-gemini-s1/configs/openvela_ui_native \
  -j2 distclean
./build.sh \
  vendor/allwinnertech/boards/r528/r528s3-gemini-s1/configs/openvela_ui_native \
  -j2
```

Gemini 使用 LTO，建议先保持 `-j2`。POSTBUILD 会把适合 pack 的剥离结果写入：

```text
vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/configs/nsh.fex
```

不要手工把约百 MiB 的 raw `nuttx.bin` 当成 `nsh.fex`。检查：

```bash
test -s nuttx/nuttx
test -s nuttx/nuttx.elf
test -s nuttx/nuttx.map
test -s vendor/allwinnertech/lichee/board/r528s3/gemini-s1_nand/configs/nsh.fex
```

打包：

```bash
cd /path/to/openvela-contest-gemini-s1-28/vendor/allwinnertech/lichee
source envsetup.sh
lunch_nuttx r528s3-gemini-s1
pack
```

成功标志包含：

```text
Dragon execute image.cfg SUCCESS
pack finish
```

输出镜像：

```text
vendor/allwinnertech/lichee/out/r528s3/gemini-s1_nand/
rtos_nuttx_r528s3-gemini-s1_uart0_128Mnand.img
```

pack 会把 Gemini `data/usrdata` 中的字体、PNG、PCM、Wi-Fi、天气和同步配置一起制成完整镜像。只编译不 pack，不能得到带素材的可烧录文件。

## 9. 已验证 8.3 镜像

本目录包含已验证 Gemini 镜像：

- [openvela-ui-gemini-s1-2.8-8.3.img](release/8.3/openvela-ui-gemini-s1-2.8-8.3.img)
- [SHA256SUMS](release/8.3/SHA256SUMS)
- 大小：`68,690,944` 字节
- SHA-256：`c1926cf37ffb0ac0176676bbd3a567fca80077be46c669ad7032496cea2dfaef`

校验：

```bash
cd release/8.3
sha256sum -c SHA256SUMS
```

该镜像是原实板验证版本。源码中的可重打包配置已使用占位值；网络环境不同时，请填写自己的参数后重新构建/pack。该镜像只能用于 Gemini S1 128 MiB NAND 板。

## 10. 首次启动检查

串口进入 NSH 后执行：

```sh
ps
mount
ls -l /data/openvela_ui
ls -l /data/etc/openvela_ui
ifconfig
date
ntpcstatus
```

`ps` 中一个 `openvela_ui` Task 下出现同进程组的 worker pthread 是正常现象，不等同于启动了第二个应用实例。`ntpcstatus` 样本为 0 但日期正确时，可能是 HTTP Date 后备已经校时。

全盘擦除烧录会清空 `/data` 中的 Wi-Fi、运动目标、历史和同步队列。Gemini 没有主体板的自定义城市文件和待机/抬腕功能。

## 11. 已知限制

- 320 × 240 横屏上左右留黑是设计画布等比缩放的结果。
- 只有五个固定城市，没有城市添加和删除。
- 无 60 秒应用待机、抬腕唤醒或真实 IMU 数据。
- 当前启动链会执行重复的 Wi-Fi/NTP 启动路径，并启动 mediad，网络和音乐稳定性可能低于主体板。
- 播放中直接切歌可能 XRUN/静音，建议先暂停。
- 通知、电池、健康和运动传感器均未完成真实硬件接入。
- 临时 Tunnel 停止后，本地 UI 仍可使用，但天气和同步会重试退避。
- Gemini 与主体板是两个独立源码分支；后续改动必须显式评估并分别同步，不能直接共享构建输出。

## 12. 进一步文档

- [Gemini S1 2.8 部署与用户使用手册](docs/gemini_s1_28_deployment_and_user_manual.md)
- [应用模块说明](packages/demos/openvela_ui/README.md)
- [电脑端天气/同步代理](tools/weather_proxy/README.md)
