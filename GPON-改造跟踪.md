# GPON 协议改造跟踪

> 分支: `AN7581-gpon`
> 设备: Nokia XG-040G-MD (AN7581)
> 最后更新: 2026-10-13

---

## 一、ITU-T G.984 ONU 状态机 (O1-O5)

| 状态 | 名称 | 含义 | 关键动作 |
|------|------|------|----------|
| **O1** | Initial | 初始状态 | SerDes PMA 校准(PrCal/EO scan)、FreqDet 锁定、XGTC PSync hunt |
| **O2** | Standby | 收到下行信号 | XGTC 帧同步(PSync)、收到首个 ALOC 分配 |
| **O3** | Serial Number | 序列号上报 | OLT 广播 SN_REQ -> ONU 回复 SN_PLOAM -> OLT 测量 EqD |
| **O4** | Ranging | 测距/分配 | OLT 分配 ONU-ID、激活密钥交换、上行时隙对齐 |
| **O5** | Operational | 完全运行 | OMCI 配置 T-CONT/GEM、carrier up、正常数据转发 |

> O2 和 O3 在驱动日志中合并为 `O2_3`，因为 XGTC 同步后 SN 上报几乎同时进行。

### 状态转换流程

```
O1 (Initial)
  |
  | 光信号检测 + PCS PMA 校准 + FreqDet 锁定
  v
O2 (Standby)  <-- XGTC PSync 建立
  |
  | 收到首个下行分配
  v
O3 (Serial Number)  <-- SN_PLOAM 发送
  |
  | OLT 分配 ONU-ID (Assign_ONU_ID PLOAMd)
  v
O4 (Ranging)  <-- 上行时隙对齐、EqD 测量
  |
  | EqD 绝对值设置完成
  v
O5 (Operational)  <-- OMCI 配置 T-CONT/GEM、carrier up
```

---

## 二、当前进展 (截至 2026-10-13)

### 2.1 第一阶段: XG-PON 模式 (mode=1) -- 成功到达 O5

首次启动使用 XG-PON 模式，PCS 层完全正常，O1->O5 完整走通。

| 时间 | 状态 | 事件 |
|------|------|------|
| [20.7s] | O1 | PON 资源绑定, mode=xgpon, WAN_SEL=9 |
| [29.3s] | O1 | PCS PrCal 成功: lock=1, fl=0xa49d ~ target=0xa49a |
| [29.4s] | O1 | PMA 校准完成, 完整 SerDes analog init |
| [29.5s] | O2_3 | XGTC 帧同步, MAC 进入 O2_3 |
| [35.0s] | O3 | 上行 TX 使能, PLOAMd result=0 (OLT 开始响应) |
| [35.8s] | O4 | ONU-ID=12 分配, OMCC GEM 提交 |
| [36.0s] | **O5** | EqD=0x00084879, 进入 O5! "service GEM/T-CONT mapping still required" |
| [36.2s] | 回退 | OLT Deactivate -> MAC epoch reset -> 回退 O2_3 |
| [41~92s] | 循环 | 重复 O1->O5 三次, 每次都被 OLT Deactivate 打回 |
| [93.2s] | 切换 | "PON line mode gpon will take effect on next activation" |

**结论**: XG-PON 模式下 PCS 层完全正常(FreqDet 锁定、PMA 校准通过)，O1->O5 完整走通。但 OLT 在 O5 后立即 Deactivate，循环反复 -- 这是 OLT 侧行为(可能是 SN 不在白名单或 OMCI 不匹配)。

### 2.2 第二阶段: GPON 模式 (mode=0) -- 卡在 O1

切换到 GPON 模式后，PCS 物理层无法锁定 GPON 2.488Gbps 下行频率，始终停留在 O1。

| 时间 | 状态 | 事件 |
|------|------|------|
| [106s] | -- | PON line restart, 切换到 GPON 模式 |
| [108.0s] | O1 | PCS PrCal 失败: lock=0, fl=0x738c >> target=0x2926 |
| [108.1s] | O1 | PMA 校准失败, prcal=-11, WAN_SEL=0 |
| [111s~] | O1 | 无限循环: "optical signal present without XGTC sync" |
| | | recovery 1~22+ (每 ~3.6s 短恢复, 每 5 次全量重初始化) |
| | | 始终 lock=0, fl~0x7350~0x739e |

**当前状态**: 卡在 **O1(Initial)**，PCS 物理层无法锁定 GPON 2.488Gbps 下行频率。

---

## 三、根因分析: GPON PCS FreqDet 失锁

### 3.1 关键参数对比

| 参数 | XG-PON (正常) | GPON (异常) | 分析 |
|------|---------------|-------------|------|
| PrCal target | 0xa49a | 0x2926 | GPON 2.5G 的目标频率 |
| 实际 fl | 0xa49d (~target) | 0x738c (2.5x target!) | GPON 频率偏差 ~180% |
| lock 状态 | 1 | 0 | 未进入锁定窗口 |
| idac | 0x46d (收敛) | 0x7ff (饱和到顶) | 校准电流搜到最大值仍未收敛 |
| calibration_error | 0 | -11 | 持续报错 |

### 3.2 问题定位

`fl=0x738c` 约等于 `0x2926 x 2.8`，说明 CDR 反馈环路测量的频率远高于 GPON 2.488Gbps 的预期。可能原因:

1. **CDR 速率配置不对**: OSR=1 可能不够，或 `PXP_AEQ_SPEED` 的覆盖顺序被后续寄存器写入冲掉
2. **FreqDet 窗口错误**: 0x150/0x154 的 lock/unlock 窗口 `(0x298a<<16)|0x28c2` 可能计算有误
3. **GPON 下行实际不是 2.488G**: 如果 OLT 发的是 GPON 但 AN7581 SerDes 按别的速率解调，FreqDet 就会偏
4. **寄存器覆盖冲突**: `an7581_xgpon_static()` 先写完整的 XG-PON 表再做 GPON override，但部分 XG-PON 表项可能覆盖了 GPON 需要的值

### 3.3 涉及的关键代码

| 文件 | 位置 | 说明 |
|------|------|------|
| `608-net-pcs-airoha-add-AN7581-PON-line-modes.patch` | `an7581_pcs_pon_xgpon_static()` GPON override 段 | GPON CDR/OSR/LPF ratio 覆盖 |
| 同上 | `an7581_pcs_pon_xgpon_prcal_once()` | PrCal target=0x2926 |
| 同上 | FreqDet 窗口设置 (0x150/0x154) | lock/unlock 边界 |
| 同上 | `an7581_xgpon_rx[]` 表 | XG-PON RX 基础表(GPON 在此基础上 override) |

### 3.4 近期相关提交

| 提交 | 说明 |
|------|------|
| `94eabf9c8c` | 修正 GPON OSR 寄存器地址(0x374->0x76c)和值(2->1)，补 CDR LPF ratio |
| `c40fc98c14` | 添加 GPON OSR 参数修复频率检测 |
| `ca25626f33` | AN7581 GPON 完整 PCS 初始化支持 |
| `b5e5dc06f0` | GPON 调用完整 an7581_pcs_pon_rx_ready 序列 |
| `b707be1dc3` | AN7581 GPON PCS 补充 FORCE_RX_RDY 门控 |
| `26139d22e6` | AN7581 GPON PCS 设置 DIG_RESERVE_0=0x400 |

---

## 四、改造计划

### Phase 1: PCS 物理层锁定 [当前卡点]

| # | 任务 | 优先级 | 状态 | 说明 |
|---|------|--------|------|------|
| 1.1 | 验证 GPON FreqDet target 计算 | P0 | 待做 | 对比 AN7581 datasheet 确认 0x2926 是否正确; fl=0x738c 对应的实际频率 |
| 1.2 | 检查 GPON CDR/OSR 寄存器覆盖顺序 | P0 | 待做 | 确认 `PXP_AEQ_SPEED` BIT(24)+bits[17:16]=1 没被后续 XG-PON 表项冲掉; 加 debugfs dump 验证 |
| 1.3 | 检查 GPON DIG_RESERVE_0=0x400 效果 | P1 | 待做 | 确认这个值是否足够让 2.5G CDR 正常工作 |
| 1.4 | 尝试不同的 GPON PrCal idac 起始值 | P1 | 待做 | 当前 idac 饱和到 0x7ff，可能需要调整搜索起点或 DAC 范围 |
| 1.5 | 对比 AN7581 原厂 GPON 初始化序列 | P0 | 待做 | 如果有 MTK/Airoha 原厂 BSP，逐寄存器对比 GPON 模式下的 PMA/ANA 配置 |

### Phase 2: GPON MAC 层 (PCS 锁定后)

| # | 任务 | 状态 | 说明 |
|---|------|------|------|
| 2.1 | GPON XGTC PSync 验证 | 待做 | PCS lock 后确认 XGTC 帧同步能建立 |
| 2.2 | GPON PLOAM 通道验证 | 待做 | 确认 GPON 速率下的 PLOAM 消息收发(1.244G 上行) |
| 2.3 | GPON SN 注册流程 | 待做 | 验证 O2->O3->O4 的 SN_PLOAM 和 Assign_ONU_ID |
| 2.4 | GPON 上行时隙对齐 | 待做 | EqD 测量和 burst mode TX 参数 |

### Phase 3: OMCI 与数据面

| # | 任务 | 状态 | 说明 |
|---|------|------|------|
| 3.1 | OMCI over GPON | 待做 | 确认 OMCC GEM Port 在 GPON 模式下正确建立 |
| 3.2 | T-CONT/GEM 映射 | 待做 | O5 后 airoha-pond 配置 service path |
| 3.3 | 上行数据验证 | 待做 | burst TX 在 1.244Gbps 下的正确性 |
| 3.4 | 下行数据验证 | 待做 | 2.488Gbps -> 2500BASEX 的 PCS 到 MAC 数据通路 |

### Phase 4: 稳定性与回归

| # | 任务 | 状态 | 说明 |
|---|------|------|------|
| 4.1 | OLT Deactivate 循环问题 | 待做 | XG-PON 模式下 O5->Deactivate 循环需要排查(SN/LOID/OMCI 匹配) |
| 4.2 | GPON/XG-PON 热切换 | 待做 | 模式切换后 PCS 重新校准的可靠性 |
| 4.3 | Recovery 机制 | 待做 | 当前 GPON 模式无限 recovery 循环，需要设上限或降级策略 |

---

## 五、下一步行动

**最紧急: Phase 1.1 和 1.2** -- GPON PCS 不锁是一切问题的根源。

1. 在 `an7581_pcs_pon_xgpon_static()` 的 GPON override 段之后，加 debugfs 或 `pr_info` dump 关键寄存器(`PXP_AEQ_SPEED`、`DIG_RESERVE_0`、FreqDet 0x150/0x154)，确认覆盖没被冲掉
2. 对比 `fl=0x738c` 和 `target=0x2926` 的比值(~2.8x)，确认是否对应某个已知的速率错配(比如 6.22G 或 2.488G x 某个分频系数)
3. 如果有原厂 GPON 初始化代码，逐表对比 `an7581_xgpon_*` 系列中哪些寄存器在 GPON 模式下不应该被写

---

## 六、日志关键指标速查

### debug-line.txt

```
configured_mode: gpon          # 配置的 PON 模式
active_mode: gpon              # 实际生效的模式
lifecycle: wait-line-sync      # 生命周期状态
optical_signal: 1              # 光信号检测
rx_active: 1                   # RX 激活
upstream_tx_armed: 0           # 上行 TX 未就绪
data_path_configured: 0        # 数据面未配置
service_ready: 0               # 服务未就绪
rx_start_count: 25             # RX 启动次数(含 recovery)
pcs_profile: gpon              # PCS profile
```

### debug-registration.txt

```
onu_state: O1                  # 当前 ONU 状态
onu_id: 1023                   # ONU-ID (1023=无效/广播)
serial_configured: 1           # SN 已配置
registration_id_configured: 1  # Registration ID 已配置
ploamd_accepted: 40            # 接受的 PLOAMd 数
ploamd_rejected: 4             # 拒绝的 PLOAMd 数
```

### debug-pcs.txt

```
profile: gpon                  # PCS profile
rx_active: 1                   # RX 激活
signal_detected: 1             # 信号检测
freqdet_locked: 0              # FreqDet 未锁定 <-- 问题所在
frequency_level: 0x7354        # 当前频率电平
frequency_target: 0xa49a       # 目标频率(注意: 这是 XG-PON 的值!)
calibration_stage: 10          # 校准阶段
calibration_error: -11         # 校准错误
```

> **注意**: `frequency_target` 显示 0xa49a (XG-PON 值) 而非 0x2926 (GPON 值)，可能是 debugfs 读取的是 XG-PON 的寄存器地址，需要确认 GPON 模式下 target 寄存器是否正确切换。
