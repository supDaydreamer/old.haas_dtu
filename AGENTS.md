# Repository Guidelines

## 项目结构与模块
- `src/main.c` 为入口，启动 MQTT (`mqtt.c`) 与 HaaS MQTT (`haas_mqtt.c`/`haas_mqtt_new.c`)、UART（`uart.c`/`bf_uart.c`）、数据处理（`data.c`）与指令调度（`bf_cmd.c`）等线程。`udp.c` 负责 UDP 转串口桥接，`bfmsg.c`/`json.c` 处理消息封装。
- `data.c` 汇总运行态：读取 `/mnt/usr/device.conf` 与 `/mnt/usr/haas_energy.conf` 配置、维护 Modbus 寄存器映射与被动监测、能耗窗口统计、设备控制与 CRC 计算等。`bf_cmd.c` 协调 MQTT 下行与串口控制，`bfbr.c` 可作为 MQTT broker/client 的轻量测试实现。
- 第三方/公共库在 `src/lib/`（cJSON、Mongoose、Paho MQTT Embedded-C），设备协议相关在 `src/mcu_sdk/`（Tuya 协议、`mcu_api.c`、`protocol.c`、`system.c`、`wifi.h`）。`build/` 存放产物与 `version` 标记。
- `src/bfbr.test.sh` 提供本机快速 MQTT broker 回归脚本（依赖本机 gcc/pthread）。

## 构建与清理
- 运行 `make`（默认 `IS_DEBUG=1`，`-std=gnu99`，工具链前缀 `mipsel-openwrt-linux-`）生成 `build/haas_dtu`，并写入 `build/version`。构建依赖 `../liblfds/liblfds7.1.1/liblfds711`，工具链需在 `PATH`；如需本机编译可覆盖 `CROSS=`。
- `make` 会先执行 `make clean`，编译完成后清理 `mk.obj/`。`make clean` 删除产物与中间文件；`make clean_obj` 仅清理对象目录/二进制。
- MQTT/Mongoose 本地测试：执行 `sh src/bfbr.test.sh` 生成并运行 `test.bfbr`，`Ctrl+C` 退出后脚本自清理。

## 编码风格与命名
- C 代码保持现有 tab 缩进与紧凑 brace 风格（`function()\n{`），遵循 `-W -Wall` 基础上已有的 `-Wno-*` 抑制策略。
- 变量/函数使用 snake_case，常量与宏用全大写下划线；全局常以 `g_` 前缀，结构体字段保持现有命名（如 `energy_window_s`、`vt_gain`、`ct_gain`）。
- 头文件加防重包含，标准库在前、项目头在后；模块内静态数据/函数保持文件私有作用域。

## 配置与运行时注意
- 主要配置文件：`/mnt/usr/device.conf`（`config` 段含 `dev_type`、`RS485_type`、`upload_time`、`haas_dev_num`、`vt_gain`、`ct_gain`、`energy_window_s` 及 `dev_addXX/reg_addXX/data_lenXX/cmdXX/typeXX` 等映射；`device_id`、控制位等）；`/mnt/usr/haas_energy.conf`（`cfg.device_type`）。修改时保持键名兼容，考虑缺省值处理。
- `ini.c` 写入依赖已有文件并使用文件锁；避免并发修改或硬编码敏感信息（MQTT 凭据/终端地址应经配置或环境提供）。
- `dev_type`、`RS485_type` 会影响串口行为（被动监测/主动采集、能耗初始化流程），变更后需确认 Modbus 时序与 CRC 正确；`upload_time`/`energy_window_s` 影响上报节奏与清零窗口。
- 版本信息写入 `build/version`，设备码读取 `/mnt/usr/bf_code`；依赖文件缺失时逻辑会回退默认值，调试前确认路径存在。

## 测试与验证
- 最少执行 `make` 确认编译通过；涉及 MQTT/Mongoose 改动可跑 `sh src/bfbr.test.sh` 做本地烟测。
- 触及 UART/RS485、寄存器映射或能耗窗口逻辑时，需在目标硬件上联调，验证帧格式、CRC、响应匹配与上报间隔；记录配置与观察结果。
- 协议/配置项调整时补充说明默认值、升级兼容性，并在 MR/提交中写明验证方式（命令输出或抓包/日志）。

## 提交与评审
- 提交信息用简洁祈使句/范围前缀，例如 `mqtt: tighten reconnect backoff`、`data: guard modbus map init`。
- 合并请求需包含：变更概要、影响模块、测试证据（`make`/`bfbr.test.sh`/设备联调）、协议或配置影响（如寄存器表、上传周期、MQTT 主题/凭据变更）。
