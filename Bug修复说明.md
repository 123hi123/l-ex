# 🔧 Bug 修复说明 (v1.1.8)

## 修复的问题

### 1. ❌ 问题：快捷输入按钮重复创建

**现象：**
- 每次点击 textarea 时，快捷输入按钮（⎘）和其他按钮会重复出现
- 离开焦点再点击会继续添加新按钮
- 导致工具栏出现多个相同的按钮

**原因：**
- 点击事件处理器直接移除并重新注入按钮
- 没有检查按钮是否已存在

**解决方案：**
```javascript
// 修改前：
textarea.addEventListener('click', () => {
    const toolbars = findAllToolbars();
    toolbars.forEach((toolbar) => {
        const oldButtons = toolbar.querySelectorAll('.emoji-extension-button');
        oldButtons.forEach(btn => btn.remove());
        injectEmojiButton(toolbar);  // 每次都会创建新按钮
    });
});

// 修改后：
textarea.addEventListener('click', () => {
    setTimeout(() => {
        attemptInjection();  // 使用现有逻辑，会自动检查是否已存在
    }, 50);
});
```

**效果：**
- ✅ 按钮不会重复创建
- ✅ 已存在的按钮会被保留
- ✅ 只在需要时才创建新按钮

---

### 2. ❌ 问题：图片粘贴功能不工作

**现象：**
- 复制图片链接后粘贴到 textarea，没有任何反应
- 图片链接无法正确插入到输入框

**原因：**
1. `getAsString` 是异步的，但没有正确等待结果
2. 没有监听外层容器 `.chat-composer__input-container`
3. 图片链接的正则匹配不够全面
4. 事件触发不够完整

**解决方案：**

#### 改进 1：支持外层容器监听
```javascript
// 同时监听 textarea 和外层容器
const textareas = document.querySelectorAll('textarea.chat-composer__input');
const containers = document.querySelectorAll('.chat-composer__input-container');

// 为外层容器添加粘贴监听
containers.forEach((container) => {
    container.addEventListener('paste', (e) => {
        const textarea = container.querySelector('textarea');
        if (textarea) {
            handlePaste(e, textarea);
        }
    });
});
```

#### 改进 2：正确处理异步文本获取
```javascript
// 修改前（错误）：
item.getAsString((text) => {
    if (text.match(/\.(jpg|jpeg|png|gif|webp)$/i)) {
        insertImageLink(textarea, text);  // 无法阻止默认行为
    }
});

// 修改后（正确）：
const text = await new Promise((resolve) => {
    item.getAsString((str) => resolve(str));
});

if (text && text.match(/图片链接正则/)) {
    e.preventDefault();  // 正确阻止默认粘贴
    insertImageLink(textarea, text.trim());
    return;
}
```

#### 改进 3：更全面的图片链接检测
```javascript
// 支持多种图片链接格式
if (text && (
    text.match(/\.(jpg|jpeg|png|gif|webp|bmp)(\?.*)?$/i) ||  // 带查询参数
    text.includes('/uploads/') ||                             // Discourse 上传路径
    text.match(/https?:\/\/.*\.(jpg|jpeg|png|gif|webp|bmp)/i) // 完整 URL
)) {
    // 插入图片链接
}
```

#### 改进 4：增强的插入函数
```javascript
function insertImageLink(textarea, imageUrl) {
    // 1. 处理相对路径
    let fullUrl = imageUrl;
    if (imageUrl.startsWith('/uploads/')) {
        fullUrl = window.location.origin + imageUrl;
    }
    
    // 2. 智能添加换行
    let prefix = '';
    let suffix = '';
    if (textBefore.length > 0 && !textBefore.endsWith('\n')) {
        prefix = '\n';
    }
    if (textAfter.length > 0 && !textAfter.startsWith('\n')) {
        suffix = '\n';
    }
    
    // 3. 触发多种事件以确保 Ember 应用能检测到
    textarea.dispatchEvent(new Event('input', { bubbles: true }));
    textarea.dispatchEvent(new Event('change', { bubbles: true }));
    textarea.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true }));
    
    console.log('[Emoji Extension] Image link inserted successfully');
}
```

**效果：**
- ✅ 图片链接能正确粘贴
- ✅ 支持多种链接格式
- ✅ 自动处理相对路径
- ✅ 智能添加换行
- ✅ Discourse 能正确检测到内容变化

---

## 技术改进

### 统一的粘贴处理函数

创建了独立的 `handlePaste` 函数来统一处理所有粘贴事件：

```javascript
async function handlePaste(e, textarea) {
    const items = e.clipboardData?.items;
    if (!items) return;
    
    console.log('[Emoji Extension] Paste event detected');
    
    // 1. 检查文本（图片链接）
    for (let item of items) {
        if (item.type === 'text/plain') {
            const text = await new Promise(resolve => 
                item.getAsString(str => resolve(str))
            );
            
            if (isImageUrl(text)) {
                e.preventDefault();
                insertImageLink(textarea, text.trim());
                return;
            }
        }
    }
    
    // 2. 检查图片文件
    for (let item of items) {
        if (item.type.indexOf('image') !== -1) {
            e.preventDefault();
            const file = item.getAsFile();
            const url = await uploadImageToDiscourse(file);
            if (url) insertImageLink(textarea, url);
            return;
        }
    }
}
```

### 调试日志增强

添加了更详细的日志输出：

```javascript
console.log('[Emoji Extension] Paste event detected, items count:', items.length);
console.log('[Emoji Extension] Image URL detected:', text.trim());
console.log('[Emoji Extension] Image file detected, uploading...');
console.log('[Emoji Extension] Image uploaded and inserted:', uploadUrl);
console.log('[Emoji Extension] No image or image URL detected in paste');
```

---

## 测试方法

### 测试 1：按钮不重复
```
1. 打开 Discourse 聊天
2. 点击 textarea 输入框
3. 再次点击
4. 重复多次
✅ 应该：每个按钮只出现一次
```

### 测试 2：粘贴图片链接
```
1. 复制图片链接，如：
   https://linux.do/uploads/default/original/4X/f/4/7/f479dbd.webp
2. 在 textarea 中粘贴（Ctrl+V）
✅ 应该：自动插入 ![image](链接)
```

### 测试 3：粘贴相对路径
```
1. 复制：/uploads/default/original/4X/test.jpg
2. 粘贴到 textarea
✅ 应该：自动补全为完整 URL
```

### 测试 4：粘贴图片文件
```
1. 截图或复制图片文件
2. 粘贴到 textarea
✅ 应该：自动上传并插入链接
```

---

## 版本信息

**版本：** 1.1.8  
**修复日期：** 2025年1月  
**状态：** ✅ Bug 已修复

### 修改统计

- 修改函数：3 个
  - `setupTextareaClickHandlers()` - 防止按钮重复
  - `setupPasteHandlers()` - 改进粘贴处理
  - `insertImageLink()` - 增强插入功能
- 新增函数：1 个
  - `handlePaste()` - 统一粘贴处理
- 新增日志：5 条

---

## 已知限制

1. 图片上传依赖 Discourse 的 `/uploads.json` API
2. 需要有效的 CSRF Token
3. 粘贴的图片文件大小受服务器限制

---

## 后续优化

- [ ] 添加上传进度提示
- [ ] 支持拖拽上传
- [ ] 添加图片预览
- [ ] 支持批量粘贴

---

<div align="center">

**✅ Bug 修复完成！请重新安装脚本测试。**

</div>
