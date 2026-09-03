# 修复：耳机插入时自动切换输出设备

## 问题描述

当 EarPods 或 3.5mm 耳机插入 Mac 的耳机接口时：
1. **不会自动切换到耳机输出** - 系统仍然使用之前的输出设备
2. **音量偏低** - 可能因为没有正确识别为耳机设备

## 根本原因

当耳机插入内建音频接口时，CoreAudio 的设备列表不会变化。相反：
- **Built-in Output** 设备保持存在
- 只有设备的 **DataSource** 属性会改变：
  - 从 `spkr`（扬声器）变为 `hdpn`（耳机，FourCC: `0x6864706E`）

FineTune 已经在监听 DataSource 变化，但只是触发设备列表刷新，**没有触发设备切换逻辑**。

## 解决方案

### 1. AudioDeviceMonitor.swift 的更改

#### 添加新的回调
```swift
/// Called when built-in device data source changes (headphone jack plug/unplug)
var onBuiltInDataSourceChanged: ((_ deviceID: AudioDeviceID, _ uid: String, _ name: String) -> Void)?
```

#### 跟踪 DataSource 状态
```swift
/// Tracks last known data source for each built-in device to detect actual changes
@ObservationIgnored private var lastKnownDataSources: [AudioDeviceID: UInt32] = [:]
```

#### 改进 DataSource 监听器
- **存储初始状态**：在添加监听器时记录当前 DataSource
- **检测实际变化**：只在 DataSource 真正改变时触发回调
- **调用新回调**：通知 AudioEngine 发生了 DataSource 变化

#### 新增处理函数
```swift
private func handleDataSourceChanged(deviceID: AudioDeviceID) {
    // 读取新的 DataSource
    // 与旧值比较
    // 如果真的改变了，调用 onBuiltInDataSourceChanged
}
```

### 2. AudioEngine.swift 的更改

#### 注册 DataSource 回调
在设备监听器初始化时：
```swift
if let monitor = deviceMonitor as? AudioDeviceMonitor {
    monitor.onBuiltInDataSourceChanged = { [weak self] deviceID, deviceUID, deviceName in
        self?.handleBuiltInDataSourceChanged(deviceID: deviceID, uid: deviceUID, name: deviceName)
    }
}
```

#### 新增处理函数
```swift
private func handleBuiltInDataSourceChanged(deviceID: AudioDeviceID, uid deviceUID: String, name deviceName: String)
```

这个函数会：
1. **检测耳机状态**：使用 `deviceID.builtInHasHeadphonesActive()` 判断是否插入耳机
2. **判断优先级**：检查 Built-in Output 是否是优先级列表中的最高设备
3. **自动切换**：
   - 如果 Built-in 是最高优先级且耳机已插入 → 切换到它
   - 如果 macOS 自动切换到 Built-in 但它不是最高优先级 → 恢复之前的设备
4. **防止干扰**：进入 `PENDING_AUTOSWITCH` 状态，防止 macOS 的自动切换被误判为用户操作

## 工作流程

### 耳机插入时：

```
1. 用户插入 3.5mm 耳机
   ↓
2. macOS 改变 Built-in Output 的 DataSource: spkr → hdpn
   ↓
3. AudioDeviceMonitor 的 DataSource 监听器触发
   ↓
4. handleDataSourceChanged() 检测到实际变化
   ↓
5. 调用 onBuiltInDataSourceChanged 回调
   ↓
6. AudioEngine.handleBuiltInDataSourceChanged() 被调用
   ↓
7. 检测 builtInHasHeadphonesActive() = true
   ↓
8. 评估优先级并切换设备（如果 Built-in 是最高优先级）
   ↓
9. 进入 PENDING_AUTOSWITCH 状态（2秒）
```

### 耳机拔出时：

DataSource 会从 `hdpn` 变回 `spkr`，同样触发处理逻辑，但因为 `hasHeadphones = false`，不会触发切换。

## 测试建议

1. **基本功能测试**：
   - 插入 EarPods → 应该自动切换到 Built-in Output（如果它是最高优先级）
   - 拔出 EarPods → 应该保持在 Built-in Output 或切换到下一个设备
   - 插入普通 3.5mm 耳机 → 同上

2. **优先级测试**：
   - 如果有更高优先级的设备（如 USB DAC）正在使用
   - 插入耳机 → 应该保持在 USB DAC，不切换
   - 在设置中调整优先级，使 Built-in 最高
   - 再次测试插入 → 应该切换到 Built-in

3. **音量测试**：
   - 插入耳机后检查音量是否正常
   - 检查 AutoEQ 是否正确应用到耳机

4. **边缘情况**：
   - 快速插拔耳机多次
   - 在插入耳机的同时连接/断开其他设备
   - 系统偏好设置中手动切换设备

## 关键技术点

### DataSource FourCC 值
- `0x6864706E` = `'hdpn'` = 耳机（headphone）
- `0x73706B72` = `'spkr'` = 扬声器（speaker）

### 为什么需要跟踪 lastKnownDataSources
CoreAudio 的 DataSource 监听器有时会在没有实际变化时触发（如系统唤醒）。跟踪上一个值可以避免误触发。

### PENDING_AUTOSWITCH 状态
这个状态防止将 macOS 的自动切换误判为"用户手动选择"。在 2 秒窗口期内的系统切换会被识别为自动切换，不会更新 `lastConfirmedDefaultUID`。

## 相关文件

- `FineTune/Audio/Monitors/AudioDeviceMonitor.swift` - DataSource 监听和检测
- `FineTune/Audio/Engine/AudioEngine.swift` - 设备切换逻辑
- `FineTune/Audio/Extensions/AudioDeviceID+Classification.swift` - 耳机检测辅助函数

## 注意事项

1. 此修复只影响 **Built-in Output** 设备的 DataSource 变化
2. 不影响其他类型的设备（USB、蓝牙等）
3. 尊重用户的优先级设置 - 如果 Built-in 不是最高优先级，不会强制切换
4. 与现有的设备连接/断开逻辑兼容

## 后续可能的改进

1. **音量调整**：可以在检测到耳机时应用特定的音量配置
2. **通知提示**：插入耳机时显示通知
3. **输入设备**：类似处理 Built-in Input 的 DataSource 变化（麦克风切换）
4. **用户设置**：添加选项控制是否在耳机插入时自动切换
