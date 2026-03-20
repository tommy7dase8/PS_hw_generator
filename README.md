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
* ✨ **AI 辅助 OCR 录题****：为了极大提升题目录入效率，本项目引入了基于大模型的 OCR 辅助录题功能。你可以直接将带有复杂数学公式的题目截图粘贴到系统中，AI 会自动将其转换为标准无损的 Markdown 和 LaTeX 公式代码。

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

## 调用API可能会出现的跨域 (CORS) 问题

尝试直接从托管在 GitHub Pages 的纯前端页面发起对大模型 API 的 `fetch` 请求时，可能遭遇经典的浏览器的安全拦截 (F12控制台日志)：

```
已拦截跨源请求：同源策略禁止读取位于 https://xxx/chat/completions 的远程资源。（原因：CORS 请求未能成功 / 缺少 'Access-Control-Allow-Origin' 头）
```

**🔍 问题根本原因分析**

**浏览器安全策略**：绝大多数官方的大模型 API 服务器出于防滥用目的，仅允许“服务器对服务器 (Server-to-Server)”的调用，拒绝来自不受信任的网页前端（Origin）的跨域请求。

 **✅ 我的解决方案：Supabase Edge Functions 边缘代理**

为了彻底解决跨域问题并保护 API Key，本项目利用现有的 Supabase 架构，部署了一个轻量级的 Serverless 边缘函数作为“中间人代理”。

**数据流转逻辑：** `GitHub Pages 前端` ➡️ `携带 Supabase Token 发起请求` ➡️ `Supabase Edge Function (解包并拼装真实 API Key)` ➡️ `请求大模型 API` ➡️ `原路返回识别结果`

1. 登录 Supabase 网页控制台，进入你的项目。
2. 在左侧菜单找到并点击 **Edge Functions**。
3. 进入 Edge Functions 页面后，在页面的顶部标签栏（Tabs）或者右上角，你会看到一个叫做 **“Secrets”** 的选项。
4. 点击进入 Secrets 管理界面，选择 **Add new secret**。
5. 填入你的大模型密钥：LLM_API_KEY 和 LLM_API_URL   然后点击保存。
6. 回到 Edge Functions 的主列表页面，点击右上角的 **Create a new Edge Function**（或者点击 Deploy a new function -> Via Editor）。
7. 在弹出的侧边栏或新窗口中，**Function name** 填 `ocr-proxy`。
8. 此时会进入网页版的代码编辑器。把里面的默认代码全删了，粘贴下面的那段完整代码。
9. 保持 JWT Verification 开启，点击创建。
10. 点击右上角的 **Deploy**。
11. 部署完成后，你会在这个函数的详情页看到一个类似于 `https://<你的项目ID>.supabase.co/functions/v1/ocr-proxy` 的 URL。复制它，替换github secret key里面的LLM_API_URL
12. index.html里面runOCR这个函数也别忘了改: `Bearer ${llmApiKey}` 这句话改成 `Bearer ${SUPABASE_URL}`

### 💻 核心代码存档

我们在 Supabase 云端部署了以下通用代理脚本。该脚本依赖于云端配置的 `LLM_API_KEY` 和 `LLM_API_URL` 两个环境变量，实现了极佳的模型解耦，随时可以零代码切换底层大模型：

```typescript
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

Deno.serve(async (req) => {
  // 1. 处理浏览器的 CORS 预检请求 (OPTIONS)
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    // 2. 从云端环境变量中读取 API Key 与目标接口地址（做到完全解耦，不限模型提供商）
    const LLM_API_KEY = Deno.env.get('LLM_API_KEY')
    if (!LLM_API_KEY) {
      throw new Error("后端未配置 LLM_API_KEY")
    }

    const LLM_API_URL = Deno.env.get('LLM_API_URL')
    if (!LLM_API_URL) {
      throw new Error("后端未配置 LLM_API_URL")
    }

    // 3. 读取前端发来的标准请求体
    const requestData = await req.json()

    // 4. 携带真实凭证，在服务端安全转发请求给大模型
    const response = await fetch(LLM_API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${LLM_API_KEY}`
      },
      body: JSON.stringify(requestData)
    })

    const data = await response.json()

    // 5. 将大模型处理结果携同跨域允许头，安全返回给前端网页
    return new Response(JSON.stringify(data), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    })
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    })
  }
})
```

