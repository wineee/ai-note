# 统信OA加班申请填写经验

## 加班提前申请表单填写指南

### 基本信息
- **流程名称**: 加班提前申请
- **流程ID**: 1419
- **URL**: `https://oa.uniontech.com/spa/workflow/static4form/index.html#/main/workflow/req?iscreate=1&workflowid=1419`

### 表单字段对应表

| 字段名称 | 字段ID | 填写值示例 | 说明 |
|---------|--------|-----------|------|
| 预计加班开始日期 | field33271 | 2026-03-12 | 格式: YYYY-MM-DD |
| 预计加班开始时间 | field33272 | 19:00 | 格式: HH:MM |
| 预计加班结束日期 | field33273 | 2026-03-12 | 格式: YYYY-MM-DD |
| 预计加班结束时间 | field33274 | 21:00 | 格式: HH:MM |
| 预计加班时长 | field32984 / field33275 | 2 | 单位为小时 |
| 预计加班地点 | field32982 | 0 | 0=公司内, 1=公司外 |
| 加班类型 | field32989 | 1 | 1=工作日, 2=双休日, 3=法定节假日 |
| 补偿方式 | field32991 | 0 | 0=调休假 |
| 加班事由 | field32988 | 工作任务需要... | 文本描述 |
| 工时项目编码 | field51206 | YFXMCY032025080004 | 选择工时项目后自动填充 |
| 工时项目名称 | field51201 | 桌面操作系统专业版V25... | 选择工时项目后自动填充 |

### 常用工时项目

| 项目编码 | 项目名称 |
|---------|---------|
| YFXMCY032025080004 | 桌面操作系统专业版V25 2500 U1（deepin25.1.0） |

### 下拉菜单选项值

**预计加班地点 (field32982)**:
- 公司内 = 0
- 公司外 = 1

**加班类型 (field32989)**:
- 工作日加班 = 1
- 双休日加班 = 2
- 法定节假日加班 = 3

**补偿方式 (field32991)**:
- 调休假 = 0

### JavaScript 快速填写脚本

```javascript
// 设置时间
const tmrw = '2026-03-12';
document.getElementById('field33271').value = tmrw; // 开始日期
document.getElementById('field33272').value = '19:00'; // 开始时间
document.getElementById('field33273').value = tmrw; // 结束日期
document.getElementById('field33274').value = '21:00'; // 结束时间

// 设置地点
document.getElementById('field32982').value = '0';
document.getElementById('field32982span').textContent = '公司内';

// 设置加班类型和补偿方式
document.getElementById('field32989').value = '1';
document.getElementById('field32991').value = '0';

// 设置时长
document.getElementById('field32984').value = '2';
document.getElementById('field33275').value = '2';

// 设置工时项目
const code = 'YFXMCY032025080004';
const name = '桌面操作系统专业版V25 2500 U1（deepin25.1.0）';
document.getElementById('field51206').value = code;
document.getElementById('field51201').value = name;
document.getElementById('field51202').value = code;
document.getElementById('field51206span').textContent = name;

// 填写事由
document.getElementById('field32988').value = '工作任务需要，申请加班完成相关工作内容。';
```

### 注意事项
1. 工作日加班开始时间为正常下班后的1小时起算
2. 有效加班时长不少于2小时
3. 需提前申请，审批通过后根据实际打卡时间填写加班登记
4. 工时项目选择弹窗需要点击列表项后确认
5. 表单提交可能需要手动点击提交按钮，系统自动计算部分字段

---
*记录时间: 2026-03-11*
*申请人: 陆红旭 (UT004971)*
