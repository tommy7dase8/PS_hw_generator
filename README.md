# 📝 习题生成器
2026 Spring, ECNU DaSE Undergraduate, Probability Theory and Mathematical Statistics Homework homework generator.

## ✨ 为什么这个项目有用

这个项目可以最大程度节省老师/助教整理习题/答案的时间, 工作流前后对比: 

原来: 选择题目 -> 大模型OCR图片转文本 -> 核对 -> 导出习题 -> 根据习题答案再OCR一次 -> 再次核对 -> 发布答案

现在: 核对+实时编辑 -> 选择题目 -> 直接导出习题和答案

------

一个轻量级、云端同步的习题生成系统。本项目采用**纯静态前端架构**（Zero-Build），无需本地 Node.js/npm 环境，通过 GitHub Actions 自动注入凭证并部署至 GitHub Pages，数据与图片统一托管于 Supabase。

## ✨ 核心特性

* 📚 **树状目录组织**：支持按“章 - 节”（如：`第一章 - 1.1习题`）两级目录结构分类管理题库，左侧菜单可折叠。
* ✍️ **实时公式渲染**：深度集成 Marked.js 与 MathJax，完美支持单行 `$` 与多行跨行 `$$` 的 LaTeX 复杂数学公式渲染。
* 🖼️ **云端图床直传**：支持在编辑器中一键上传图片，自动存储至 Supabase Storage 并转换为 Markdown 链接插入。
* 🖨️ **智能剥离导出**：支持勾选题目导出为 **Markdown** 或 **PDF**。导出时可自动识别以“解”或“证”开头的段落，自由选择导出“纯净习题版”或“完整解析版”。
* 📖 **双屏参考核对**：右侧集成独立 PDF 预览面板，可根据当前题集自动映射并打开对应的本地/云端参考答案。
* ⏱️ **无感历史备份**：利用数据库底层 Trigger（触发器）实现自动版本控制，哪怕修改覆盖无数次，也可通过云端历史表随时找回。

## 🛠️ 技术栈

* **前端**：HTML5 + Vue 3 (CDN 引入)
* **解析**：Marked.js (自定义块级/内联公式扩展) + MathJax 3
* **后端 & 数据库**：Supabase (PostgreSQL + Storage)
* **部署 & CI/CD**：GitHub Pages + GitHub Actions

## 🚀 快速部署指南

本系统无需在本地运行任何打包命令（如 `npm run build`），只需配置好云端环境并推送到 GitHub 即可点开即用。

### 1. 配置 Supabase

1. 新建一个 Supabase 项目。
2. 在 SQL Editor 中运行以下语句创建题库表：
   ```sql
   -- 创建题目表
   CREATE TABLE questions (
       id SERIAL PRIMARY KEY,
       chapter_title TEXT NOT NULL, -- 章节名
       question_order INTEGER NOT NULL, -- 题号顺序
       raw_md TEXT NOT NULL, -- 题目的完整 Markdown（含解答）
       created_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW())
   );
   
   -- 开启匿名访问（为了部署 GitHub Pages 方便，可以开启 RLS 并允许匿名读写，或者配置 Row Level Security 仅限你自己）
   ALTER TABLE questions ENABLE ROW LEVEL SECURITY;
   CREATE POLICY "Allow anonymous read/write" ON questions FOR ALL USING (true) WITH CHECK (true);
   ```
3. 在 **Storage** 中新建一个名为 `images` 的公开（Public）存储桶，用于存放题目图片。

### 2. 配置 GitHub 自动化部署

1. Fork 或 Clone 本仓库。
2. 进入 GitHub 仓库主页，点击 `Settings` -> `Secrets and variables` -> `Actions`。
3. 添加两个 **Repository secrets**：
   * `SUPABASE_URL`：填入你的 Supabase Project URL。
   * `SUPABASE_KEY`：填入你的 Supabase `anon public` Key。
4. 将代码推送到 `main` 分支。GitHub Actions 会自动替换 `index.html` 中的安全占位符，并将网页发布到 GitHub Pages。

*(注：系统绝不会在源码中暴露数据库 Key，保障绝对安全。)*

## 📂 高级功能配置

### 配置 PDF 对照表

如果需要在使用时右侧弹出对应的参考答案，请在项目根目录新建 `pdf_mapping.json` 文件，并按以下格式配置：

```json
{
  "第一章 - 1.1习题": "./pdfs/chapter1.pdf",
  "第二章 - 2.1补充": "https://xxxxx.supabase.co/storage/v1/object/public/pdfs/ch2.pdf"
}
```

### 开启数据库“后悔药”（历史备份）

在 Supabase 的 SQL Editor 中运行以下代码，即可开启防误删/防覆盖保护机制：

```sql
-- 第 1 步：创建一张历史记录表（相当于你的“回收站”或“时光机”）
CREATE TABLE questions_history (
    history_id SERIAL PRIMARY KEY,            -- 历史记录的独立ID
    question_id INTEGER REFERENCES questions(id) ON DELETE CASCADE, -- 关联到原题目的ID（如果原题被彻底删了，历史记录也跟着删）
    old_raw_md TEXT NOT NULL,                 -- 被覆盖前的旧 Markdown 代码
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) -- 修改发生的时间
);

-- 第 2 步：编写一个触发器函数（告诉数据库“当发生修改时，具体要做什么”）
CREATE OR REPLACE FUNCTION log_question_update()
RETURNS TRIGGER 
SECURITY DEFINER -- 新增这一行，赋予触发器最高执行权限
AS $$
BEGIN
    -- 只有当 raw_md 字段的内容真的发生了改变时，才记录（防止无意义的保存操作触发记录）
    IF OLD.raw_md IS DISTINCT FROM NEW.raw_md THEN
        INSERT INTO questions_history (question_id, old_raw_md) VALUES (OLD.id, OLD.raw_md);
    END IF;
    -- 返回新数据，允许正常的更新操作继续进行
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 第 3 步：将触发器绑定到你的 questions 表上（设置监听器）
CREATE TRIGGER trigger_log_question_update
AFTER UPDATE ON questions
FOR EACH ROW
EXECUTE FUNCTION log_question_update();
```
当发生误操作时，进入 Supabase 左侧的 **Table Editor**，打开 `questions_history` 表。你会看到所有的修改历史都在这里，按时间排好了。找到那道题被改坏前的 `old_raw_md`，双击复制，然后回你的网页里重新粘贴保存即可。

### 图片上传的权限问题

1. 登录你的 Supabase 控制台，进入你的项目。

2. 在左侧菜单栏点击 Storage。

3. 在 Storage 页面左侧的菜单中，点击 Policies（策略）。

4. 在右侧列表中找到你创建的 images 存储桶，点击它旁边的 New Policy 按钮。

5. 在弹出的窗口中，选择 "For full customization"（完全自定义）或者 "Create a policy from scratch"。

6. 按照以下设置填写：

    - Policy name: 随便起个名字，比如 Allow public uploads

    - Allowed operations: 勾选 INSERT（重要！这是允许上传的权限）

    - Target roles: 点击下拉框，勾选 anon（代表允许匿名用户操作）

7. 其它地方留空或者保持默认，直接点击右下角的 Review，然后点击 Save policy。