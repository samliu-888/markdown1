# PCIe PERST 中断处理与端口初始化流程图

## 整体流程概览

```
PERST中断触发
    ↓
pciePortPerstIsrHandler (中断处理入口)
    ↓
判断: PERST Assert 还是 Deassert?
    ↓
如果是 Deassert → 调用 port->ops->init() → 建立链路
如果是 Assert  → 调用钩子函数处理复位
```

---

## 详细流程图

### 1. PERST 中断处理层

``mermaid
graph TD
    A([PERST中断触发]) --> B[pciePortPerstIsrHandler<br/>中断入口]
    B --> C{port指针有效?}
    C -->|否| D[记录错误日志<br/>返回-1]
    C -->|是| E[pciePortPerstIntClr<br/>清除PERST中断标志]
    
    E --> F[pciePortPerstSync<br/>读取PERST同步状态]
    F --> G[pciePortPerstTrigModeGet<br/>获取触发模式]
    
    G --> H{是否为高电平触发?}
    H -->|是| I[不支持高电平触发<br/>返回]
    H -->|否| J[计算触发条件<br/>下降沿=0x0, 上升沿=0x1]
    
    J --> K{perstSyncStatus == trigCondition?<br/>判断Deassert/Assert}
    
    K -->|否: PERST Assert| L[记录日志: Perst assert detected]
    L --> M[pcieCtrlCallIrqHooks<br/>调用PERST_ASSERT钩子]
    M --> N([结束])
    
    K -->|是: PERST Deassert| O[记录日志: Perst deassert detected]
    O --> P{是否使用ISR任务队列?}
    
    P -->|是| Q[pcieIsrEnqueueEvent<br/>事件入队<br/>PCIE_CTRL_EVT_PERST_DEASSERT]
    Q --> R([结束<br/>由后台任务处理])
    
    P -->|否| S[<b>port->ops->init(port)</b><br/>调用端口初始化]
```

**核心代码位置**: `pcie_ctrl/pi/tianxie/pcie_ctrl_tianxie_irq.c:632-697`

**关键逻辑说明**:
- **中断快速响应**: ISR中只做必要的寄存器读写和状态判断
- **两种处理模式**:
  - **直接模式** (`#ifndef USE_ISR_TASK_QUEUE`): 立即调用 `port->ops->init()` 和链路建立
  - **队列模式** (`#ifdef USE_ISR_TASK_QUEUE`): 将事件放入队列，由后台任务 `pcieIsrBottWorkerTask` 异步处理
- **触发模式判断**: 支持上升沿和下降沿触发，暂不支持高电平触发

---

### 2. 端口初始化层 (_pciePortInit)

#### 2.1 平台选择入口

``mermaid
graph TD
    S --> T[_pciePortInit<br/>根据硬件平台选择实现]
    T --> U{硬件平台宏定义?}
    
    U -->|PS3_HARDWARE_ASIC| V[pciePortInitASIC]
    U -->|PS3_HARDWARE_FPGA| W[pciePortInitFPGA]
    U -->|PS3_HARDWARE_EMU| X[pciePortInitEMU]
    
    V --> CommonInit
    W --> CommonInit
    X --> CommonInit
    
    CommonInit[通用初始化流程<br/>见下方详细步骤] --> End([返回初始化结果])
```

#### 2.2 通用初始化流程（提取共性操作）

所有平台共享的核心步骤如下：

``mermaid
graph TD
    Start[进入平台特定init函数] --> Step1{是否需要<br/>PHY PCLK检查?}
    
    Step1 -->|ASIC/FPGA需要| CheckPhyPclk[Step1: 检查FPGA PHY PCLK状态<br/>CMD_PORT_FPGA_PHY_PCLK_STATUS<br/>等待phypclk释放]
    Step1 -->|EMU不需要| SkipPhyPclk
    
    CheckPhyPclk --> Step2
    SkipPhyPclk --> Step2[Step2: TOP_CRG时钟稳定检查<br/>确保系统时钟就绪]
    
    Step2 --> Step3[Step3: 禁用LTSSM<br/>CMD_PORT_LTSSM_DISABLE<br/>设置CTRL寄存器bit31=0<br/>防止意外启动]
    
    Step3 --> Step4{是否有<br/>特殊配置?}
    
    Step4 -->|ASIC: PCS&PMA| HookConfig[Step4: Hook回调配置PCS&PMA<br/>待实现的外部回调]
    Step4 -->|FPGA: PROC调整| ProcModify{是否定义<br/>PCIE_PROC_NOT_READY?}
    Step4 -->|EMU: 无| SkipSpecial
    
    ProcModify -->|是| ModifyProc[修改PROC设置<br/>CRG_CTRL_EXT寄存器<br/>val &= ~0x70; val |= 0x50]
    ProcModify -->|否| SkipProc
    ModifyProc --> Step5
    SkipProc --> Step5
    
    HookConfig --> Step5
    SkipSpecial --> Step5
    
    Step5[Step5: 释放软件PERST#<br/>CMD_PORT_PERST_SOFT_RST_VALID<br/>设置CTRL寄存器bit9=1] --> Step6
    
    Step6[Step6: 解除PHY复位保持<br/>CMD_PORT_APPHOLD_PHY_RST_INVALID<br/>设置CTRL寄存器bit28=0] --> Step7
    
    Step7[Step7: 等待CRG状态正常<br/>CMD_PORT_CHECK_CRG_STATUS<br/>轮询直到所有信号复位完成<br/>超时保护机制] --> Step8
    
    Step8{是否需要<br/>Serdes时钟通知?}
    Step8 -->|是| SerdesNotify[Step8: Serdes参考时钟稳定通知<br/>TODO: 需要回调机制]
    Step8 -->|否| SkipSerdes
    
    SerdesNotify --> Step9
    SkipSerdes --> Step9
    
    Step9{是否需要<br/>PMA固件重载?}
    Step9 -->|ASIC需要| PmaReload[Step9: 重载PMA固件<br/>ASIC特有步骤]
    Step9 -->|FPGA/EMU不需要| SkipPma
    
    PmaReload --> Step10
    SkipPma --> Step10
    
    Step10[Step10: 配置4K寄存器<br/>CMD_PORT_4K_REG_INIT_SET<br/>详见下一节详细展开] --> Step11
    
    Step11[Step11: 启动LTSSM Trace<br/>CMD_PORT_LTSSM_TRACE_START<br/>如果ltssmTraceEnable=true] --> Step12
    
    Step12[Step12: 调用初始化钩子<br/>CMD_PORT_CALL_HOOKS<br/>遍历并执行sInitHooks数组中的回调] --> End([init函数返回0<br/>初始化成功])
```

#### 2.3 各平台差异对比

| 步骤 | ASIC | FPGA | EMU | 说明 |
|------|------|------|-----|------|
| Step1: PHY PCLK检查 | ✅ | ✅ | ❌ | ASIC/FPGA需验证物理层时钟 |
| Step2: CRG时钟检查 | ✅ | ✅ | ✅ | 所有平台都需要 |
| Step3: LTSSM禁用 | ✅ | ✅ | ✅ | 防止意外启动 |
| Step4: 特殊配置 | ⚠️ PCS&PMA(Hook) | ⚠️ PROC调整 | ❌ | 平台特定配置 |
| Step5: 释放软件PERST# | ✅ | ✅ | ✅ | 软件复位释放 |
| Step6: 解除PHY复位 | ✅ | ✅ | ✅ | 硬件复位释放 |
| Step7: 等待CRG状态 | ✅ | ✅ | ✅ | 轮询直到正常 |
| Step8: Serdes时钟通知 | ⚠️ TODO | ✅ | ✅ | ASIC待实现回调 |
| Step9: PMA固件重载 | ✅ | ❌ | ❌ | 仅ASIC需要 |
| Step10: 4K寄存器配置 | ✅ | ✅ | ✅ | 核心配置步骤 |
| Step11: LTSSM Trace启动 | ✅ | ✅ | ✅ | 状态跟踪 |
| Step12: 调用Hooks | ✅ | ✅ | ✅ | 用户扩展点 |

**符号说明**:
- ✅: 已实现且必需
- ⚠️: 部分实现或可选
- ❌: 不适用

#### 2.4 关键步骤详解：Step1 - PHY PCLK状态检查（增强版）

**原始实现问题分析**:
- 原超时时间固定为 1ms (`PCIE_READ_REG_TIMEOUT_CNT`)
- 超时后直接返回 `-ETIMEDOUT`，导致初始化失败
- 在某些场景下（如仿真环境、慢速时钟），1ms可能不足

**增强方案**:

```
/**
 * @brief 检查FPGA PHY PCLK状态（增强版，支持兼容性配置）
 * 
 * @param port 端口指针
 * @param timeoutMs 最大超时时间（毫秒），默认1ms
 * @param warnThresholdMs 警告阈值（毫秒），超过此时间打印警告但继续等待
 * @return S32 0成功，负值失败
 */
S32 pciePortGetPhypclkStatus(PcieCtrlPort_s *port, 
                              U32 timeoutMs, 
                              U32 warnThresholdMs)
{
    U32 status;
    U32 timeCnt;
    U32 warnPrinted = 0;  // 标记是否已打印警告
    
    if (!port) {
        PCIE_CTRL_LOG(PCIE_LOG_LEVEL_ERROR, "[%s] port is NULL\n", __func__);
        return -ENODEV;
    }
    
    // 参数有效性检查
    if (timeoutMs == 0) {
        timeoutMs = PCIE_READ_REG_TIMEOUT_CNT;  // 使用默认值
    }
    if (warnThresholdMs == 0 || warnThresholdMs > timeoutMs) {
        warnThresholdMs = timeoutMs / 2;  // 默认在超时一半时警告
    }
    
    // 计算循环次数
    timeCnt = timeoutMs * 1000 / PCIE_READ_REG_TIME_INTERVAL;
    
    PCIE_CTRL_LOG(PCIE_LOG_LEVEL_DEBUG, 
        "[%s] Start waiting for phypclk, timeout=%ums, warn_at=%ums\n",
        __func__, timeoutMs, warnThresholdMs);
    
    while (timeCnt > 0) {
        port->regOps->readReg(port, 4, PCIE_PORT_SPACE_CTRL_REG, 0, &status);
        
        if (status != PCIE_INVALID_REG_VALUE) {
            PCIE_CTRL_LOG(PCIE_LOG_LEVEL_INFO, 
                "[%s] phypclk released successfully after %d iterations\n", 
                __func__, timeoutMs * 1000 / PCIE_READ_REG_TIME_INTERVAL - timeCnt);
            return 0;
        }
        
        // 检查是否达到警告阈值
        U32 elapsedMs = (timeoutMs * 1000 / PCIE_READ_REG_TIME_INTERVAL - timeCnt) 
                        * PCIE_READ_REG_TIME_INTERVAL / 1000;
        
        if (!warnPrinted && elapsedMs >= warnThresholdMs) {
            PCIE_CTRL_LOG(PCIE_LOG_LEVEL_WARN, 
                "[%s] WARNING: phypclk not released after %ums (threshold reached), "
                "continuing to wait until %ums timeout\n",
                __func__, elapsedMs, timeoutMs);
            warnPrinted = 1;
        }
        
        timeCnt--;
        udelay(PCIE_READ_REG_TIME_INTERVAL);
    }
    
    // 最终超时
    PCIE_CTRL_LOG(PCIE_LOG_LEVEL_ERROR, 
        "[%s] ERROR: phypclk wait timeout (%ums expired). "
        "Last read status=0x%x (expected non-0x%lx)\n",
        __func__, timeoutMs, status, (unsigned long)PCIE_INVALID_REG_VALUE);
    
    return -ETIMEDOUT;
}
```

**使用示例**:

```
// ASIC平台调用示例
S32 pciePortInitASIC(PcieCtrlPort_s *port)
{
    S32 ret = 0;
    
    // ... 其他初始化代码 ...
    
    // Step1: 检查FPGA PHY PCLK状态（增强版）
    // 配置：最大超时10ms，超过2ms时打印警告
    ret = pciePortGetPhypclkStatus(port, 10, 2);
    if (ret != 0) {
        PCIE_CTRL_LOG(PCIE_LOG_LEVEL_ERROR, 
            "[%s] pcie port get fpga phy pclk status failed, ret=%d\n", 
            __func__, ret);
        return ret;
    }
    
    // ... 后续步骤 ...
}

// EMU平台可以跳过此步骤
S32 pciePortInitEMU(PcieCtrlPort_s *port)
{
    // EMU环境不需要检查phypclk
    PCIE_CTRL_LOG(PCIE_LOG_LEVEL_INFO, 
        "[%s] Skip phypclk check in EMU mode\n", __func__);
    
    // ... 直接从Step2开始 ...
}
```

**配置建议**:

| 场景 | timeoutMs | warnThresholdMs | 说明 |
|------|-----------|-----------------|------|
| **ASIC生产环境** | 10 | 2 | 允许较慢的硬件启动，提前预警 |
| **FPGA调试环境** | 50 | 10 | FPGA时序不稳定，给予更大余量 |
| **EMU仿真环境** | 跳过 | - | 仿真环境无需检查 |
| **快速启动模式** | 1 | 0 | 严格超时，不打印警告 |
| **保守模式** | 100 | 20 | 极端情况下保证成功率 |

**优势**:
1. ✅ **向后兼容**: 默认参数保持原有1ms行为
2. ✅ **灵活配置**: 通过参数调整适应不同场景
3. ✅ **早期预警**: 在完全超时前打印警告，便于诊断
4. ✅ **详细日志**: 记录实际等待时间和最终状态
5. ✅ **容错性强**: 即使超过1ms也能继续等待，提高成功率

---

## 总结

### 架构设计亮点

1. **分层抽象**:
   - 中断层: 快速响应，最小化处理
   - 初始化层: 顺序执行，平台差异化
   - 命令层: 统一接口，表驱动分发

2. **平台隔离**:
   - ASIC/FPGA/EMU 独立实现，互不影响
   - 共性操作提取到通用流程
   - 差异操作通过条件编译或参数配置

3. **可扩展性**:
   - Hook机制允许用户自定义扩展
   - 命令表驱动易于添加新功能
   - 超时参数化支持灵活配置

4. **健壮性**:
   - 每步都有返回值检查
   - 超时保护防止无限等待
   - 详细的日志便于问题定位

**核心文件索引**:
- 中断处理: `pcie_ctrl/pi/tianxie/pcie_ctrl_tianxie_irq.c`
- 平台初始化: `pcie_ctrl/pi/tianxie/pcie_ctrl_tianxie_core.c`
- 命令定义: `pcie_ctrl/include/pcie_ctrl_common.h`

---

**文档版本**: v2.0  
**最后更新**: 2026-05-30  
**变更说明**: 
- 提取共性操作流程
- 增强phypclk检查的兼容性
- 简化文档结构，聚焦核心流程
