<template>
  <div class="job">
    <canvas id="canvas"></canvas>
    <h1>材料审批作业</h1>
    <div class="workspace" ref="workspace">
      <div class="left"></div>
      <div class="views" ref="docView">
        <p>
          在这个快节奏的世界里，人们常常忽略了细节的重要性。无论是在工作中还是生活中，细节决定成败。
          事实上，正是那些小事，往往带来最大的影响。相同的句子出现在不同的地方。人们喜欢追求速度，
          但如果不注重细节，结果往往不尽如人意。专注于小事的完成是成功的关键。这种态度不仅在职业生涯中至关重要，
          在个人生活中也同样适用。无论你在做什么，确保你关注到了每一个细节。当你与他人交流时，注意到每一个细节，
          会让你显得更加专业。相同的句子出现在不同的地方。成功的企业往往非常关注细节，他们知道，一个小的失误
          可能会造成巨大的损失。因此，关注细节并为其设定高标准，是实现卓越的必经之路。
        </p>

        <p>
          细节决定成败，这句话已经被证明无数次。相同的句子出现在不同的地方。一个成功的项目往往依赖于那些
          看似微不足道的小细节。优秀的领导者通常会深入关注每一个细节，而不仅仅是大局。相反，那些忽略细节的企业
          往往会在竞争中败下阵来。当细节得到了充分的关注，项目的质量和完成度往往会提升到一个新的水平。
          相同的句子出现在不同的地方。这个世界的本质是由无数的小事组成的，每一件小事都有其独特的作用和意义。
          在复杂的系统中，任何一个细节的忽略都可能引发连锁反应，导致整体失败。因此，养成重视细节的习惯，
          是迈向成功的重要一步。
        </p>

        <p>
          在竞争激烈的市场中，企业的成功往往取决于他们对细节的关注。相同的句子出现在不同的地方。
          无论是产品的设计、制造，还是服务的提供，关注细节是建立品牌声誉的关键因素。那些看似不重要的部分，
          实际上决定了客户的最终体验。一个小的失误可能会影响整个公司的声誉。相同的句子出现在不同的地方。
          所以，企业必须从上到下培养一种注重细节的文化。这种文化会帮助企业在复杂的环境中找到突破口，
          并在市场上获得竞争优势。
        </p>
        <n-dropdown
          placement="top-end"
          trigger="manual"
          :x="menuConfig.x"
          :y="menuConfig.y"
          :options="[{
            label: '新增意见',
            key: 'add'
          }]"
          :show="isMenuShow"
          :on-clickoutside="onClickoutside"
          @select="addComment"
        />
      </div>
      <div class="right">
        <div v-for="(comment, i) in commentList" :key="i" class="comment" ref="commentElement">
          <div>这是对“{{ comment.text }}”的意见</div>
          <n-button @click="deleteComment(comment)">移除意见</n-button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import { NDropdown, NButton } from 'naive-ui';

const menuConfig = ref({ x: 0, y: 0 });
const docView = ref(null);
const workspace = ref(null);
const isMenuShow = ref(false);
const ctx = ref(null);

const savedRange = ref(new Range());  // 保存选中的范围
const selectedText = ref('');  // 保存选中的文本
const commentList = ref([]);
const commentElement = ref(null);

onMounted(() => {
  setupCanvas();

  // TODO 这里要防抖
  window.addEventListener('resize', () => {
    setupCanvas();
    reflowAllConnection();
  });

  const view = docView.value;
  // 鼠标释放时处理文本选择
  view?.addEventListener('mouseup', mouseupHandler);

  // 监听选择变化，取消选中时隐藏菜单
  view?.addEventListener('selectionchange', () => {
    const selection = window.getSelection();
    if (selection.isCollapsed) {
      isMenuShow.value = false;
    }
  });
});

const mouseupHandler = () => {
  const selection = window.getSelection();
  // 如果有选中的文本
  if (!selection.isCollapsed) {
    const range = selection.getRangeAt(0);
    // 保存选中的范围
    savedRange.value = range;
    selectedText.value = range.toString();
    // 计算菜单显示的位置并显示
    showMenu(savedRange.value.getBoundingClientRect());
    return;
  }
  isMenuShow.value = false;
}

const setupCanvas = () => {
  const canvas = document.getElementById('canvas');
  // 设置高分辨率 (2x)
  const scaleFactor = 2;
  canvas.width = window.innerWidth * scaleFactor;
  canvas.height = window.innerHeight * scaleFactor;
  // 使用 CSS 缩放回 1x 尺寸
  canvas.style.width = `${window.innerWidth}px`;
  canvas.style.height = `${window.innerHeight}px`;
  const context = canvas.getContext('2d');
  // 按照缩放比例调整绘制
  context.scale(scaleFactor, scaleFactor);
  // 启用图像平滑
  context.imageSmoothingEnabled = true;
  ctx.value = context;
};

// 清除文本中的高亮
const clearHighlight = (text) => {
  const highlighted = [...document.querySelectorAll(`.highlight[data-text="${text}"]`)];
  highlighted.reduce((_, span) => {
    const parent = span.parentNode;
    parent.replaceChild(document.createTextNode(span.textContent), span);
    parent.normalize();  // 合并相邻文本节点
  }, null);
}

// 获取高亮文本的节点
function* getHighlightNodes(root) {
  if (root.nodeType === Node.ELEMENT_NODE && root.classList.contains('highlight')) {
    yield root;
  }
  for (let child of root.childNodes) {
    yield* getHighlightNodes(child);
  }
}

// 检查选中的范围是否完全在已高亮的文本内
const isWithinHighlight = range => range.startContainer.parentElement.classList.contains('highlight') && range.endContainer.parentElement.classList.contains('highlight');

// 检查选中的范围是否包含已高亮的部分文本
const isHighlightSelected = range => [...getHighlightNodes(range.commonAncestorContainer)].some(highlightNode => range.intersectsNode(highlightNode));

// 遍历文本节点
function* getTextNodes(root) {
  const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, null);
  let node;
  while (node = walker.nextNode()) {
    if (node.parentNode && !node.parentNode.classList.contains('highlight')) {
      yield node;  // 使用 yield 返回每个文本节点
    }
  }
}

// 高亮选中的文本
function highlightText(root, text) {
  const textRegex = new RegExp(`(${text})`, 'gi');
  // 将所有文本节点收集到数组中，避免在修改 DOM 时影响遍历
  const textNodes = [...getTextNodes(root)];
  // 遍历所有文本节点并进行高亮
  textNodes.forEach(node => {
    const textContent = node.textContent;
    const parts = textContent.split(textRegex);
    if (parts.length > 1) {
      const fragment = document.createDocumentFragment();
      parts.forEach(part => {
        if (textRegex.test(part)) {
          // 创建并插入高亮 span
          const highlightSpan = document.createElement('span');
          highlightSpan.dataset['text'] = part;
          highlightSpan.className = 'highlight';
          highlightSpan.textContent = part;
          fragment.appendChild(highlightSpan);
        } else {
          // 创建并插入普通文本节点
          fragment.appendChild(document.createTextNode(part));
        }
      });
      // 替换原节点
      node.parentNode.replaceChild(fragment, node);
    }
  });
}

// 显示菜单并定位到选中文本右上角
const showMenu = (rect) => {
  menuConfig.value = {
    x: rect.right + window.scrollX,
    y: rect.top + window.scrollY
  };
  isMenuShow.value = true;
}

const onClickoutside = () => {
  isMenuShow.value = false;
}

const addComment = (key) => {
  const text = selectedText.value;
  if (text) {
    // 清除旧的高亮
    // clearHighlight();
    if (isHighlightSelected(savedRange.value) || isWithinHighlight(savedRange.value)) {
      alert('不可重复选中！');
      isMenuShow.value = false
      return;
    }
    // 高亮选中的文本及其他相同文本
    highlightText(docView.value, text);
    // 取消选中文本
    window.getSelection().removeAllRanges();
    // 复制选中的文本到剪贴板
    // copyToClipboard(selectedText);
    // 提示用户
    console.log('已选中文本：' + text);
    // 隐藏菜单
    isMenuShow.value = false;
    const comment = {
      text: selectedText.value
    };
    commentList.value.push(comment);
    nextTick(() => {
      setTimeout(() => {
        makeConnection();
      }, 100);
    });
  }
}

const deleteComment = (comment) => {
  commentList.value = commentList.value.filter(item => item.text !== comment.text);
}

// 建立选中文本和意见的连接
const makeConnection = () => {
  // 选中文本的意见对象，总是最后一个
  const comment = commentElement.value[commentElement.value.length - 1];
  const endPosition = getSideCenterPosition(comment);
  drawCircle(ctx.value, endPosition, 2.5);
  const highlighted = [...document.querySelectorAll(`.highlight[data-text="${selectedText.value}"]`)];
  highlighted.reduce((_, element) => {
    const startPosition = getSideCenterPosition(element, 1);
    drawCircle(ctx.value, startPosition, 2.5);
    drawBezierCurve(
        ctx.value,
        startPosition,
        endPosition,
        { x: startPosition.x + 200, y: startPosition.y },
        { x: endPosition.x - 200, y: endPosition.y },
    );
  }, null);
}

// 重绘所有连接
const reflowAllConnection = () => {
  if(!commentList.value?.length) {
    return;
  }
  // TODO
};

// 获取元素的侧边中心点
function getSideCenterPosition(element, side = 0) {
  const { left, right, top, height } = element.getBoundingClientRect();
  return {
    x: side ? right : left,
    y: top + height / 2
  };
}

// 在给定的位置绘制圆形
function drawCircle(ctx, position, radius, color = '#444444') {
  ctx.beginPath();
  ctx.arc(position.x, position.y, radius, 0, Math.PI * 2);
  ctx.fillStyle = color;
  ctx.fill();
}

// 绘制贝塞尔曲线的函数
function drawBezierCurve(ctx, startPos, endPos, controlPoint1, controlPoint2, options) {
  ctx.beginPath();
  ctx.moveTo(startPos.x, startPos.y);
  ctx.bezierCurveTo(controlPoint1.x, controlPoint1.y, controlPoint2.x, controlPoint2.y, endPos.x, endPos.y);
  // 设置样式
  if (options?.strokeStyle) ctx.strokeStyle = options.strokeStyle;
  if (options?.lineWidth) ctx.lineWidth = options.lineWidth;
  if (options?.lineDash) ctx.setLineDash(options.lineDash);
  if (options?.lineCap) ctx.lineCap = options.lineCap;
  ctx.shadowBlur = 1;
  ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
  ctx.stroke();
  ctx.setLineDash([]);  // 重置虚线样式
}

// 绘制元素的端点
function drawEndpoint(element) {
  // 获取元素的边界矩形
  const rect = element.getBoundingClientRect();
  // 计算右侧中心点的 x 和 y 坐标
  const x = rect.right;           // x 坐标在元素右侧
  const y = rect.top + rect.height / 2; // y 坐标是元素的中心位置
  // // 创建一个圆点元素
  // const circle = document.createElement('div');
  // circle.classList.add('right-endpoint');
  // // 设置圆点的位置
  // circle.style.left = `${x}px`;   // 圆点的左侧紧贴元素的右边
  // circle.style.top = `${y}px`; // 圆心位置，需减去半径(5px)来居中
  // // 将圆点添加到 body 中
  // document.body.appendChild(circle);
}

</script>

<style lang="css" scoped>
.job {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  flex: 1;

}

.workspace {
  display: flex;
  flex: 1;
  background-color: #f0f0f0;
  position: relative;
}

#canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 1;
}

.left {
  width: 200px;
  display: flex;
  background-color: #fff;
}

.views {
  padding: 32px;
  margin: 0 200px;
  flex: 1;
  background-color: #fff;

}

.right {
  padding: 16px;
  background-color: #fff;
}

p {
  margin-bottom: 16px;
  text-align: justify;
}

.comment{
  width: 200px;
  height: 200px;
  padding: 16px;
  margin-bottom: 16px;
  border-radius: 10px;
  border: 1px solid #ccc;
}


</style>
<style>
.highlight {
  background-color: lightblue;
  border: 2px solid red;
  padding: 0 6px;
  border-radius: 3px;
  margin: 0 10px;
  position: relative;
}
</style>
