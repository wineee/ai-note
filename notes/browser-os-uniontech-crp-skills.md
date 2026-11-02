# UOS CRP 平台操作技巧

## 常用链接

- **测试主题页面**: https://crp.uniontech.com/#/theme/publishTheme/2

---

## 创建测试主题

在 CRP 平台创建测试主题的步骤：

1. 进入 **测试主题** 页面：https://crp.uniontech.com/#/theme/publishTheme/2

2. 点击 **创建主题** 按钮

3. 在弹出的对话框中填写以下**必填项**：
   - **分组名称**：输入主题名称，如 `treeland-ai-test`
   - **主题类型**：选择 `公开` 或 `私有`
   - **描述**：填写主题的相关描述信息（如用途、范围等）

4. 点击 **确定** 按钮完成创建

5. 创建成功后，新主题会出现在主题列表中

### 注意事项
- **分组名称**、**主题类型**、**描述** 均为必填项
- 主题名称建议使用简洁、有意义的命名
- 主题类型选择"公开"后所有成员可见，"私有"仅创建者可见
- 创建后可以通过"主题配置"链接修改主题设置
- 不使用的主题可以通过"放弃主题"链接删除

## 删除测试主题

1. 在测试主题列表中找到要删除的主题
2. 点击该主题右侧的 **放弃主题** 链接
3. 在弹出的确认对话框中点击 **确定** 确认删除
4. 刷新页面验证主题已删除

---

## 项目打包 API

### 打包请求

**接口**: `POST /api/topics/{topic_id}/new_release`

**请求体示例**:
```json
{
  "Arches": "amd64;mips64el;arm64;sw64;loong64",
  "BaseTag": null,
  "Branch": "upstream/master",
  "BuildID": 0,
  "BuildState": null,
  "Changelog": ["chore(debian): update version to 0.5.5"],
  "Commit": "1f890f0a578fbbfb5309fccd0acaa8c2ee3823c1",
  "History": null,
  "ID": 0,
  "ProjectID": 4443,
  "ProjectName": "treeland-protocols",
  "ProjectRepoUrl": null,
  "SlaveNode": null,
  "Tag": "1",
  "TagSuffix": null,
  "TopicID": 25660,
  "TopicType": "test",
  "ChangeLogMode": true,
  "RepoType": "deb",
  "Custom": true,
  "BranchID": "123"
}
```

**关键字段说明**:
- `Arches`: 架构列表，多个用分号分隔（如 `amd64;mips64el;arm64;sw64;loong64`）
- `Branch`: 代码分支（如 `upstream/master`）
- `Commit`: Git commit hash
- `Changelog`: 变更日志数组
- `ProjectID`: 项目 ID
- `ProjectName`: 项目名称
- `TopicID`: 主题 ID（测试主题 ID）
- `TopicType`: 主题类型（`test` 表示测试主题）
- `ChangeLogMode`: `true` 表示打 patch 模式（通过 changelog 管理版本号）
- `Custom`: `true` 表示自定义版本
- `Tag`: 自定义版本号

### 页面打包流程

1. 进入 **项目** 页面，搜索要打包的项目（如 `treeland-protocols`）
2. 点击项目右侧的 **打包** 链接
3. 在弹出的对话框中配置：
   - **主题**: 选择测试主题（如 `treeland-ai-test`）
   - **架构**: 选择需要的架构（可多选）
   - **版本**: 选择或输入自定义版本号
   - **代码分支**: 输入分支名称（如 `upstream/master`）
   - **CommitID**: 输入 Git commit hash
   - **模式**: 选择 `打patch(changelog管理版本号)`
   - **日志**: 填写变更日志
4. 点击 **确定** 提交打包任务

### API 直接调用示例

从浏览器控制台直接调用 API 提交打包任务：

```javascript
// 获取 JWT token
const token = localStorage.getItem('token');

// 发送打包请求
fetch('/api/topics/25660/new_release', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ' + token
  },
  body: JSON.stringify({
    "Arches": "amd64;mips64el;arm64;sw64;loong64",
    "BaseTag": null,
    "Branch": "upstream/master",
    "BuildID": 0,
    "BuildState": null,
    "Changelog": ["chore(debian): update version to 0.5.5"],
    "Commit": "1f890f0a578fbbfb5309fccd0acaa8c2ee3823c1",
    "History": null,
    "ID": 0,
    "ProjectID": 4443,
    "ProjectName": "treeland-protocols",
    "ProjectRepoUrl": null,
    "SlaveNode": null,
    "Tag": "1",
    "TagSuffix": null,
    "TopicID": 25660,
    "TopicType": "test",
    "ChangeLogMode": true,
    "RepoType": "deb",
    "Custom": true,
    "BranchID": "123"
  })
}).then(async response => {
  const data = await response.json();
  console.log('Status:', response.status);
  console.log('Build ID:', data);
  return data;
});
```

**返回结果**:
- 成功: 返回 HTTP 201，响应体为构建任务 ID (如 `97159`)
- 失败: 返回 HTTP 400/401/500 及错误信息
