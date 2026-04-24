# copyZhihu
知乎复制内容

// 创建复制按钮（支持富文本和图片）
function createCopyButton() {
  // 移除已存在的按钮
  const existingBtn = document.getElementById('custom-copy-btn');
  if (existingBtn) existingBtn.remove();
  
  // 创建按钮
  const button = document.createElement('button');
  button.id = 'custom-copy-btn';
  button.textContent = '📋 复制选中内容';
  button.style.cssText = `
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 9999;
    padding: 12px 24px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 14px;
    font-weight: bold;
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
    transition: all 0.3s;
  `;
  
  button.onmouseover = () => {
    button.style.transform = 'translateY(-2px)';
    button.style.boxShadow = '0 6px 20px rgba(102, 126, 234, 0.6)';
  };
  
  button.onmouseout = () => {
    button.style.transform = 'translateY(0)';
    button.style.boxShadow = '0 4px 15px rgba(102, 126, 234, 0.4)';
  };
  
  // 点击复制（支持富文本+图片）
  button.onclick = async () => {
    const selection = window.getSelection();
    
    if (selection.rangeCount === 0) {
      showToast('请先选择要复制的内容', 'warning');
      return;
    }
    
    const range = selection.getRangeAt(0);
    
    // 检查是否包含图片
    const hasImages = range.cloneContents().querySelectorAll('img').length > 0;
    const hasText = selection.toString().trim().length > 0;
    
    if (!hasImages && !hasText) {
      showToast('请先选择要复制的内容', 'warning');
      return;
    }
    
    try {
      await copyRichContent(range);
      button.textContent = '✅ 已复制';
      showToast(hasImages ? '内容（含图片）已复制！' : '内容已成功复制！', 'success');
      setTimeout(() => {
        button.textContent = '📋 复制选中内容';
      }, 2000);
    } catch (err) {
      showToast('复制失败: ' + err.message, 'error');
    }
  };
  
  document.body.appendChild(button);
}

// 复制富文本内容（包含图片）
async function copyRichContent(range) {
  // 方法1: 使用 Clipboard API（推荐，支持富文本+图片）
  if (navigator.clipboard && navigator.clipboard.write) {
    const clipboardData = await buildClipboardData(range);
    await navigator.clipboard.write(clipboardData);
    return;
  }
  
  // 方法2: 使用 execCommand（降级方案）
  fallbackCopyRich(range);
}

// 构建剪贴板数据
async function buildClipboardData(range) {
  const clipboardItems = [];
  
  // 1. 准备 HTML 内容
  const fragment = range.cloneContents();
  const container = document.createElement('div');
  container.appendChild(fragment);
  const htmlContent = container.innerHTML;
  
  // 2. 准备纯文本内容
  const plainText = range.toString();
  
  // 3. 处理图片
  const images = container.querySelectorAll('img');
  const imagePromises = Array.from(images).map(async (img) => {
    try {
      // 如果是网络图片，转换为 base64
      if (img.src.startsWith('http') && !img.src.startsWith('data:')) {
        const blob = await fetchImageAsBlob(img.src);
        const base64 = await blobToBase64(blob);
        return { original: img.src, base64: base64 };
      }
      return null;
    } catch (err) {
      console.warn('图片转换失败:', img.src, err);
      return null;
    }
  });
  
  const imageResults = await Promise.all(imagePromises);
  
  // 4. 创建 ClipboardItem
  const item = new ClipboardItem({
    'text/html': new Blob([htmlContent], { type: 'text/html' }),
    'text/plain': new Blob([plainText], { type: 'text/plain' })
  });
  
  clipboardItems.push(item);
  
  // 如果有图片，添加图片数据
  const validImages = imageResults.filter(r => r !== null);
  if (validImages.length === 1) {
    try {
      const imgBlob = await fetch(validImages[0].base64).then(r => r.blob());
      clipboardItems.push(
        new ClipboardItem({
          [imgBlob.type]: imgBlob
        })
      );
    } catch (err) {
      console.warn('添加图片到剪贴板失败:', err);
    }
  }
  
  return clipboardItems;
}

// 降级复制方案（execCommand）
function fallbackCopyRich(range) {
  const selection = window.getSelection();
  
  // 创建一个临时容器
  const tempDiv = document.createElement('div');
  tempDiv.style.position = 'fixed';
  tempDiv.style.left = '-9999px';
  tempDiv.style.top = '-9999px';
  tempDiv.appendChild(range.cloneContents());
  document.body.appendChild(tempDiv);
  
  // 选中所有内容
  const newRange = document.createRange();
  newRange.selectNodeContents(tempDiv);
  selection.removeAllRanges();
  selection.addRange(newRange);
  
  try {
    const successful = document.execCommand('copy');
    if (!successful) throw new Error('execCommand 复制失败');
  } finally {
    // 清理
    selection.removeAllRanges();
    document.body.removeChild(tempDiv);
  }
}

// 获取网络图片为 Blob
async function fetchImageAsBlob(url) {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
  return await response.blob();
}

// Blob 转 Base64
function blobToBase64(blob) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onloadend = () => resolve(reader.result);
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}

// 提示消息
function showToast(message, type = 'info') {
  const existingToast = document.getElementById('copy-toast');
  if (existingToast) existingToast.remove();
  
  const toast = document.createElement('div');
  toast.id = 'copy-toast';
  toast.textContent = message;
  
  const colors = {
    success: '#4CAF50',
    warning: '#FF9800',
    error: '#f44336',
    info: '#2196F3'
  };
  
  toast.style.cssText = `
    position: fixed;
    top: 80px;
    right: 20px;
    z-index: 10000;
    padding: 12px 24px;
    background: ${colors[type] || colors.info};
    color: white;
    border-radius: 8px;
    font-size: 14px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    animation: slideIn 0.3s ease;
  `;
  
  document.body.appendChild(toast);
  
  setTimeout(() => {
    toast.style.animation = 'slideOut 0.3s ease';
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}

// 添加动画样式
const style = document.createElement('style');
style.textContent = `
  @keyframes slideIn {
    from { transform: translateX(100%); opacity: 0; }
    to { transform: translateX(0); opacity: 1; }
  }
  @keyframes slideOut {
    from { transform: translateX(0); opacity: 1; }
    to { transform: translateX(100%); opacity: 0; }
  }
`;
document.head.appendChild(style);

// 初始化
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', createCopyButton);
} else {
  createCopyButton();
}

// 监听选中内容变化
document.addEventListener('mouseup', () => {
  const selection = window.getSelection();
  const button = document.getElementById('custom-copy-btn');
  if (!button) return;
  
  const hasContent = selection.toString().trim() || 
    (selection.rangeCount > 0 && selection.getRangeAt(0).cloneContents().querySelectorAll('img').length > 0);
  
  button.style.opacity = hasContent ? '1' : '0.6';
  button.style.pointerEvents = hasContent ? 'auto' : 'none';
});
