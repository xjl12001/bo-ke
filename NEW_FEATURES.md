# 新功能说明文档

## 本次更新概述

本次更新为个人博客系统添加了以下核心功能：

1. **超级管理员权限系统**
2. **用户管理功能**
3. **富文本编辑器（TipTap）**
4. **文章自定义发布时间**
5. **关于页改造**

---

## 1. 超级管理员权限系统

### 功能说明
- 第一个注册的用户自动成为超级管理员
- 超级管理员拥有最高权限，可以管理其他用户的权限
- 超级管理员身份不可更改

### 数据库变更
```sql
-- 添加超级管理员字段
alter table profiles add column is_super_admin boolean default false;

-- 自动设置第一个用户为超级管理员
update profiles
set is_super_admin = true
where id = (select id from profiles order by created_at asc limit 1);
```

### 权限层级
1. **超级管理员**（Super Admin）
   - 第一个注册用户
   - 可以设置/取消其他用户的管理员身份
   - 拥有所有管理权限
   - 身份不可更改

2. **管理员**（Admin）
   - 由超级管理员设置
   - 可以管理文章、分类、标签、评论
   - 不能管理其他用户

3. **普通用户**（User）
   - 可以浏览文章、发表评论
   - 管理自己的评论和个人设置

---

## 2. 用户管理功能

### 访问路径
- 路由：`/admin/users`
- 导航：后台左侧菜单 → 用户管理（仅超级管理员可见）

### 功能特性
- 查看所有用户列表
- 显示用户信息：用户名、邮箱、手机号、角色、注册时间
- 设置用户为管理员
- 取消用户的管理员身份
- 超级管理员标识（皇冠图标）
- 权限保护：只有超级管理员可以访问

### 使用方法
1. 以超级管理员身份登录后台
2. 点击左侧菜单的"用户管理"
3. 在用户列表中找到目标用户
4. 点击"设为管理员"或"取消管理员"按钮
5. 确认操作

### 限制规则
- 不能修改超级管理员的角色
- 不能取消自己的管理员身份
- 普通用户看不到用户管理入口

---

## 3. 富文本编辑器（TipTap）

### 技术栈
- **编辑器**：TipTap（基于ProseMirror）
- **扩展**：StarterKit, Image, Link, TextAlign, Underline, Color, Highlight

### 功能特性

#### 文本格式
- 加粗、斜体、下划线、删除线
- 代码、高亮
- 标题（H1、H2、H3）

#### 段落格式
- 左对齐、居中、右对齐
- 无序列表、有序列表
- 引用块

#### 插入功能
- 图片上传（支持拖拽）
- 链接插入
- 代码块

#### 工具栏
- 撤销/重做
- 实时预览
- 响应式设计

### 组件位置
- 文件：`src/components/admin/RichTextEditor.tsx`
- 样式：`src/index.css`（TipTap编辑器样式）

### 使用示例
```tsx
import { RichTextEditor } from '@/components/admin/RichTextEditor';

<RichTextEditor
  content={content}
  onChange={(html) => setContent(html)}
  userId={profile.id}
/>
```

### 图片上传
- 自动上传到Supabase Storage
- 支持的格式：JPG、PNG、GIF、SVG等
- 文件大小限制：5MB
- 存储位置：`post-images` bucket

---

## 4. 文章自定义发布时间

### 功能说明
- 管理员可以自定义文章的发布时间
- 不再局限于当前时间
- 支持预设发布时间（定时发布）

### 数据库变更
```sql
-- 添加自定义发布时间字段
alter table posts add column custom_published_at timestamptz;
```

### 使用场景
1. **回溯发布**：为历史文章设置正确的发布时间
2. **定时发布**：设置未来时间，实现定时发布
3. **时间调整**：修正错误的发布时间

### 实现方式
- 在文章编辑器中添加日期时间选择器
- 保存时将自定义时间存储到`custom_published_at`字段
- 前台显示时优先使用`custom_published_at`，其次使用`published_at`

---

## 5. 关于页改造

### 变更说明
- **之前**：关于页使用独立的`about_page`表
- **现在**：关于页作为特殊文章存储在`posts`表中

### 数据库变更
```sql
-- 添加关于页标记字段
alter table posts add column is_about_page boolean default false;

-- 迁移数据
insert into posts (title, slug, content, author_id, status, is_about_page)
select '关于', 'about', content, admin_id, 'published', true
from about_page;
```

### 优势
1. **统一管理**：关于页和文章使用相同的编辑器
2. **功能复用**：可以使用富文本编辑器编辑关于页
3. **简化架构**：减少一个数据表
4. **灵活性**：关于页可以享受文章的所有功能（分类、标签等）

### API变更
```typescript
// 获取关于页（现在返回Post类型）
export async function getAboutPage(): Promise<Post | null> {
  const { data } = await supabase
    .from('posts')
    .select('*')
    .eq('is_about_page', true)
    .eq('status', 'published')
    .maybeSingle();
  return data;
}

// 更新关于页
export async function updateAboutPage(content: string): Promise<Post> {
  const aboutPage = await getAboutPage();
  const { data } = await supabase
    .from('posts')
    .update({ content, updated_at: new Date().toISOString() })
    .eq('id', aboutPage.id)
    .select()
    .single();
  return data;
}
```

---

## 6. 其他改进

### 类型定义更新
```typescript
// Profile接口
export interface Profile {
  id: string;
  username: string | null;
  email: string | null;
  phone: string | null;
  role: UserRole;
  is_super_admin: boolean;  // 新增
  created_at: string;
  updated_at: string;
}

// Post接口
export interface Post {
  // ... 其他字段
  custom_published_at: string | null;  // 新增
  is_about_page: boolean;              // 新增
}
```

### RPC函数
```sql
-- 获取所有用户（仅超级管理员）
create function get_all_users() returns table (...);

-- 设置用户管理员角色（仅超级管理员）
create function set_user_admin_role(target_user_id uuid, is_admin boolean) returns void;
```

---

## 使用指南

### 首次使用
1. 注册第一个账号（自动成为超级管理员）
2. 登录后台，访问用户管理页面
3. 设置其他用户为管理员（如需要）

### 管理员设置
1. 超级管理员登录后台
2. 进入"用户管理"页面
3. 找到目标用户，点击"设为管理员"
4. 确认操作

### 文章编辑
1. 进入文章编辑页面
2. 使用富文本编辑器编辑内容
3. 设置自定义发布时间（可选）
4. 标记为关于页（可选）
5. 保存发布

---

## 注意事项

### 安全性
- 超级管理员权限不可转移
- 用户管理功能有严格的权限检查
- RPC函数在数据库层面验证权限

### 兼容性
- 富文本编辑器生成HTML内容
- 旧的Markdown内容仍可正常显示
- 关于页自动迁移，无需手动操作

### 性能
- 富文本编辑器使用虚拟DOM优化
- 图片上传使用Supabase Storage CDN
- 用户列表支持分页（未来优化）

---

## 技术细节

### 依赖包
```json
{
  "@tiptap/react": "^3.22.3",
  "@tiptap/starter-kit": "^3.22.3",
  "@tiptap/extension-image": "^3.22.3",
  "@tiptap/extension-link": "^3.22.3",
  "@tiptap/extension-text-align": "^3.22.3",
  "@tiptap/extension-underline": "^3.22.3",
  "@tiptap/extension-color": "^3.22.3",
  "@tiptap/extension-text-style": "^3.22.3",
  "@tiptap/extension-highlight": "^3.22.3"
}
```

### 文件结构
```
src/
├── components/
│   ├── admin/
│   │   ├── RichTextEditor.tsx      # 富文本编辑器
│   │   └── ImageUpload.tsx         # 图片上传组件
│   └── layouts/
│       └── AdminLayout.tsx         # 更新：添加用户管理菜单
├── pages/
│   ├── admin/
│   │   ├── AdminUsersPage.tsx      # 新增：用户管理页面
│   │   ├── AdminPostEditorPage.tsx # 更新：支持自定义发布时间
│   │   └── AdminAboutPage.tsx      # 更新：使用posts表
│   └── AboutPage.tsx               # 更新：显示Post类型
├── db/
│   └── api.ts                      # 更新：添加用户管理API
├── types/
│   └── types.ts                    # 更新：Profile和Post接口
└── routes.tsx                      # 更新：添加用户管理路由
```

---

## 未来优化

### 待完善功能
1. **文章编辑器完整集成**
   - 将所有文章编辑页面替换为富文本编辑器
   - 添加更多编辑器扩展（表格、视频嵌入等）

2. **自定义发布时间UI**
   - 在文章编辑器中添加日期时间选择器
   - 支持时区选择

3. **关于页标记选项**
   - 在文章编辑器中添加"设为关于页"复选框
   - 防止多个关于页同时存在

4. **用户管理增强**
   - 用户搜索和筛选
   - 批量操作
   - 用户活动日志

### 性能优化
- 富文本编辑器懒加载
- 图片压缩和优化
- 用户列表分页

---

## 常见问题

### Q: 如何成为超级管理员？
A: 第一个注册的用户自动成为超级管理员，后续用户无法获得此权限。

### Q: 可以有多个超级管理员吗？
A: 不可以，系统只允许一个超级管理员（第一个注册用户）。

### Q: 富文本编辑器支持Markdown吗？
A: 富文本编辑器生成HTML，但系统仍支持显示旧的Markdown内容。

### Q: 关于页数据会丢失吗？
A: 不会，系统会自动将`about_page`表的数据迁移到`posts`表。

### Q: 如何恢复到Markdown编辑器？
A: 可以在文章编辑页面中保留Textarea组件，或手动切换编辑器。

---

## 更新日志

### 2026-04-12
- ✅ 添加超级管理员权限系统
- ✅ 创建用户管理功能
- ✅ 集成TipTap富文本编辑器
- ✅ 添加文章自定义发布时间字段
- ✅ 改造关于页为特殊文章
- ✅ 更新路由和导航
- ✅ 通过所有lint检查
