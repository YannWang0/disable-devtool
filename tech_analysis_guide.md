# disable-devtool 技术原理分析文档

## 项目概述

`disable-devtool` 是一个用于禁用Web开发者工具的JavaScript库，通过多种检测机制来检测开发者工具的开启，并采取相应的防护措施。

## 核心技术架构

### 1. 主要组件

```
src/
├── main.ts              # 主入口逻辑
├── detector/            # 检测器模块
│   ├── index.ts         # 检测器管理
│   ├── detector.ts      # 检测器基类
│   └── sub-detector/    # 各种子检测器
├── utils/               # 工具函数
└── plugins/             # 插件系统
```

### 2. 检测器体系

项目采用**多检测器并行**的策略，通过8种不同的检测机制来监控开发者工具：

#### 2.1 尺寸检测器 (Size Detector)
**原理**: 监控窗口尺寸变化
```javascript
// 核心逻辑：当开发者工具打开时，浏览器窗口会变小
const widthUneven = window.outerWidth - window.innerWidth * screenRatio > 200;
const heightUneven = window.outerHeight - window.innerHeight * screenRatio > 300;
```
- **触发条件**: 外部窗口与内部窗口尺寸差异过大
- **适用场景**: 桌面浏览器，不适用于iframe和Edge浏览器

#### 2.2 调试器检测器 (Debugger Detector)
**原理**: 利用`debugger`语句的执行时间差
```javascript
const date = now();
(() => {debugger;})();
if (now() - date > 100) {
    this.onDevToolOpen();
}
```
- **触发条件**: debugger语句执行时间超过100ms
- **适用场景**: 仅在iOS Chrome/Edge真机环境有效

#### 2.3 性能检测器 (Performance Detector)
**原理**: 利用console.table和console.log的性能差异
```javascript
const tablePrintTime = calculateTime(() => {table(largeObjectArray);});
const logPrintTime = calculateTime(() => {log(largeObjectArray);});
// 当开发者工具打开时，table打印会显著慢于log打印
if (tablePrintTime > logPrintTime * 10) {
    this.onDevToolOpen();
}
```
- **触发条件**: table打印时间超过log打印时间的10倍

#### 2.4 函数字符串检测器 (Function toString Detector)
**原理**: 重写函数的toString方法，检测调用次数
```javascript
this.func.toString = () => {
    this.count++;
    return '';
};
log(this.func);
if (this.count >= 2) {
    this.onDevToolOpen();
}
```
- **触发条件**: 函数toString被调用2次以上（开发者工具会额外调用）

#### 2.5 其他检测器
- **RegToString**: 正则表达式toString检测
- **DefineId**: DOM元素ID检测
- **DateToString**: Date对象toString检测
- **DebugLib**: 第三方调试工具检测（eruda、vconsole等）

### 3. 防护机制

#### 3.1 快捷键禁用
```javascript
// 禁用F12、Ctrl+Shift+I/J、Ctrl+U等快捷键
const KEY = {J: 74, I: 73, U: 85, S: 83, F12: 123};
// 区分macOS和Windows的快捷键组合
const isOpenDevToolKey = IS.macos ?
    ((e, code) => (e.metaKey && e.altKey && (code === KEY.I || code === KEY.J))) :
    ((e, code) => (e.ctrlKey && e.shiftKey && (code === KEY.I || code === KEY.J)));
```

#### 3.2 右键菜单禁用
```javascript
target.addEventListener('contextmenu', (e) => {
    if (e.pointerType === 'touch') return; // 保留触摸设备的功能
    return preventEvent(target, e);
});
```

#### 3.3 复制粘贴禁用
可选择性禁用选择、复制、剪切、粘贴功能。

### 4. 定时检测机制

```javascript
// 每500ms执行一次检测
interval = window.setInterval(() => {
    if (dd.isSuspend || _pause || isIgnored()) return;
    for (const detector of calls) {
        detector.detect(time++);
    }
}, config.interval);
```

**优化策略**:
- 移动端5秒后停止检测（性能优化）
- 支持暂停/恢复检测
- 支持ignore插件控制

### 5. 绕过机制

#### 5.1 Token验证
```javascript
function checkTk() {
    if (!config.md5) return false;
    const tk = getUrlParam(config.tkName);
    return md5(tk) === config.md5; // 通过MD5验证绕过
}
```

#### 5.2 SEO优化
```javascript
if (config.seo && IS.seoBot) return; // SEO爬虫绕过
```

### 6. 配置系统

支持高度可配置化：
```javascript
const config = {
    md5: '',                    // 绕过token的MD5值
    interval: 500,              // 检测间隔
    detectors: [0,1,3,4,5,6,7], // 启用的检测器
    disableMenu: true,          // 禁用右键菜单
    ondevtoolopen: closeWindow, // 检测到时的回调
    // ... 更多配置项
};
```

### 7. 兼容性处理

#### 7.1 环境识别
```javascript
// 识别操作系统、浏览器、设备类型
const IS = {
    mobile: /mobile/i.test(navigator.userAgent),
    macos: /macintosh|mac os x/i.test(navigator.userAgent),
    chrome: /chrome/i.test(navigator.userAgent),
    // ...
};
```

#### 7.2 iframe支持
支持在iframe中禁用父页面的开发者工具。

### 8. 事件处理

#### 8.1 开发者工具打开事件
```javascript
onDevToolOpen() {
    console.warn(`You don't have permission to use DEVTOOL!【type = ${this.type}】`);
    config.ondevtoolopen(this.type, closeWindow);
    markDevToolOpenState(this.type);
}
```

#### 8.2 开发者工具关闭事件
支持检测开发者工具关闭并执行回调。

## 技术特点

### 优势
1. **多重检测**: 8种检测器并行工作，提高检测准确率
2. **高兼容性**: 支持IE、Chrome、Firefox、Edge、Safari等主流浏览器
3. **性能优化**: 移动端自动停止检测，减少性能消耗
4. **高度可配**: 支持细粒度的功能配置
5. **绕过机制**: 为开发者提供合法的绕过方式

### 局限性
1. **绕过可能**: 有经验的开发者可能找到绕过方法
2. **误报风险**: 某些特殊环境可能导致误触发
3. **兼容性**: 部分检测器在特定浏览器中可能不工作

## 部署建议

1. **npm方式**: 推荐使用，不易被单独拦截
2. **检测器选择**: 根据目标环境选择合适的检测器组合
3. **配置调优**: 根据实际需求调整检测间隔和触发阈值
4. **测试验证**: 在目标浏览器环境中充分测试

## 总结

`disable-devtool`通过多种技术手段的组合，实现了较为完善的开发者工具检测和禁用功能。其核心思想是利用开发者工具打开时浏览器环境的各种变化特征，通过多个检测器并行监控来提高检测的准确性和覆盖面。 