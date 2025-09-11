---
title: 友链机器人
date: 2025-09-05 22:25:00
updated: 2025-09-05 22:35:00
tags:
  - tgbot
  - workers
categories:
  - 后端
description: "tgbot-workers-github自动化友链机器人"
---

# 🌐 10分钟搭建 友链机器人

无需复杂编程，跟着步骤走，轻松实现 Hexo 博客**AnZhiYu 主题**友链的远程增删改查，再也不用手动改 YAML 文件！


## 📋 前置准备（3样关键信息）
先收集以下3个核心凭证，后续直接复制使用，避免中途卡顿：

| 名称 | 作用 | 获取方式 |
|------|------|----------------|
| **Telegram Bot Token** | 机器人的“身份证”，用于对接 Telegram 接口 | 1. 打开 Telegram，私聊 `@BotFather` <br> 2. 发送 `/newbot`，按提示给机器人起名字（如“我的博客友链助手”） <br> 3. 生成后复制 `token`（格式：`123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`） |
| **管理员 Chat ID** | 限制仅你能操作机器人，防止他人篡改友链 | 1. 私聊 `@userinfobot`（Telegram 内置机器人） <br> 2. 机器人会直接回复你的 `ID`（纯数字，如 `12345678`） |
| **GitHub Token** | 让机器人有权读写你博客仓库的友链文件 | 1. 登录 GitHub，进入 [Token 生成页](https://github.com/settings/tokens) <br> 2. 点击 `Generate new token`，勾选 **`repo`** 权限（展开后全选） <br> 3. 点击底部生成，复制 Token（仅显示一次，建议临时存记事本） |


## 🚀 第一步：部署 Cloudflare Worker（免费服务器）
Cloudflare Worker 是免费的轻量运行环境，用来承载机器人代码，零服务器成本：

### 1. 创建 Worker 应用
1. 打开 [Cloudflare 控制台](https://dash.cloudflare.com/)，登录后左侧菜单找到 **Workers & Pages** → 点击 **创建应用** → 选择 **创建 Worker**。
2. 给 Worker 起个名字（如 `blog-friendlink-bot`），点击 **部署**（先默认部署，后续改代码）。

### 2. 配置环境变量（安全存储敏感信息）
⚠️ 关键步骤！避免 Token 硬编码泄露，所有敏感信息加密存储：
1. 部署后，点击顶部 **设置** 标签页 → 找到 **环境变量** → 点击 **添加变量**。
2. 按表格依次添加3个变量，**每个都要勾选“加密”选项**：

| 变量名 | 变量值 | 说明 |
|--------|--------|------|
| `TGBOT_TOKEN` | 前面获取的 Telegram Bot Token | 机器人对接 Telegram 的凭证 |
| `ADMIN_CHAT_ID` | 你的管理员 Chat ID（纯数字） | 仅该 ID 能执行增删改操作 |
| `GITHUB_TOKEN` | 前面获取的 GitHub Token | 机器人操作博客仓库的权限凭证 |

3. 点击 **保存**，环境变量配置完成。

### 3. 粘贴机器人代码
1. 回到顶部 **代码** 标签页，删除默认的示例代码。
2. 复制下方的完整机器人代码，粘贴到代码编辑器中。
```js
// 主函数中获取环境变量，避免全局依赖
export default {
  async fetch(request, env) {
    // 从环境变量初始化配置
    const CONFIG = {
      TGBOT_TOKEN: env.TGBOT_TOKEN,
      ADMIN_CHAT_ID: env.ADMIN_CHAT_ID,
      GITHUB_REPO: "newstartsw/hexo-action",
      GITHUB_FILE_PATH: "source/_data/link.yml",
      GITHUB_BRANCH: "main",
      GITHUB_TOKEN: env.GITHUB_TOKEN
    };

    // 验证环境变量是否配置完整
    function checkEnvConfig() {
      const requiredVars = ['TGBOT_TOKEN', 'ADMIN_CHAT_ID', 'GITHUB_TOKEN'];
      const missing = requiredVars.filter(varName => !CONFIG[varName]);
      
      if (missing.length > 0) {
        console.error(`缺少环境变量：${missing.join(', ')}`);
        return false;
      }
      return true;
    }

    // 发送Telegram消息
    async function sendTgMessage(chatId, text) {
      const tgApiUrl = `https://api.telegram.org/bot${CONFIG.TGBOT_TOKEN}/sendMessage`;
      try {
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), 8000);
        
        const response = await fetch(tgApiUrl, {
          method: "POST",
          headers: { "Content-Type": "application/json; charset=utf-8" },
          body: JSON.stringify({
            chat_id: chatId,
            text: text,
            disable_web_page_preview: true
          }),
          signal: controller.signal
        });
        
        clearTimeout(timeoutId);
        return response.ok;
      } catch (error) {
        console.error("发送消息失败:", error);
        return false;
      }
    }

    // 验证管理员权限
    function isAdmin(chatId) {
      return String(chatId) === String(CONFIG.ADMIN_CHAT_ID);
    }

    // 解析YAML
    function parseYaml(yamlText) {
      try {
        const cleanYaml = yamlText
          .replace(/^\uFEFF/, '')
          .replace(/\r\n/g, '\n')
          .replace(/\t/g, '  ')
          .split('\n')
          .filter(line => line.trim() !== '');

        const categories = [];
        let currentCategory = null;
        let currentLink = null;
        let inLinkList = false;

        for (const line of cleanYaml) {
          const trimmedLine = line.trim();
          if (trimmedLine.startsWith('#')) continue;

          const indent = line.length - line.trimStart().length;

          if (indent === 0 && trimmedLine.startsWith('- ')) {
            if (currentCategory) {
              if (currentLink) currentCategory.link_list.push(currentLink);
              categories.push(currentCategory);
            }
            currentCategory = { link_list: [] };
            inLinkList = false;
            currentLink = null;

            const classMatch = trimmedLine.match(/class_name:\s*["']?([^"']+)["']?/);
            if (classMatch) currentCategory.class_name = classMatch[1];
            continue;
          }

          if (currentCategory && indent === 2 && !inLinkList) {
            const [key, ...valueParts] = trimmedLine.split(':')
              .map(item => item.trim().replace(/["']/g, ''));
            const value = valueParts.join(':').trim();
            
            if (key === 'link_list') {
              inLinkList = true;
            } else if (key && value !== undefined) {
              currentCategory[key] = value;
            }
            continue;
          }

          if (currentCategory && inLinkList && indent === 4 && trimmedLine.startsWith('- ')) {
            if (currentLink) {
              currentCategory.link_list.push(currentLink);
            }
            currentLink = {};
            const nameMatch = trimmedLine.match(/name:\s*["']?([^"']+)["']?/);
            if (nameMatch) currentLink.name = nameMatch[1];
            continue;
          }

          if (currentCategory && inLinkList && currentLink && indent === 6) {
            const [key, ...valueParts] = trimmedLine.split(':')
              .map(item => item.trim().replace(/["']/g, ''));
            const value = valueParts.join(':').trim();
            if (key && value !== undefined) {
              currentLink[key] = value;
            }
            continue;
          }
        }

        if (currentLink && currentCategory) {
          currentCategory.link_list.push(currentLink);
        }
        if (currentCategory) {
          categories.push(currentCategory);
        }

        return categories;
      } catch (error) {
        console.error("YAML解析错误:", error);
        return null;
      }
    }

    // 转换为YAML
    function categoriesToYaml(categories) {
      let yaml = "# 友链数据（自动生成）\n";
      
      categories.forEach((category, catIndex) => {
        yaml += `- class_name: ${category.class_name || `分类${catIndex + 1}`}\n`;
        
        const categoryKeys = ['flink_style', 'hundredSuffix', 'class_desc'];
        categoryKeys.forEach(key => {
          if (category[key] !== undefined) {
            yaml += `  ${key}: ${category[key]}\n`;
          }
        });
        
        yaml += `  link_list:\n`;
        category.link_list.forEach(link => {
          yaml += `    - name: ${link.name || ''}\n`;
          
          const linkKeys = ['link', 'avatar', 'descr', 'siteshot', 'color', 'tag', 'recommend'];
          linkKeys.forEach(key => {
            if (link[key] !== undefined) {
              yaml += `      ${key}: ${link[key]}\n`;
            }
          });
        });
        
        if (catIndex < categories.length - 1) {
          yaml += "\n";
        }
      });
      
      return yaml;
    }

    // 获取GitHub文件
    async function getGitHubFile() {
      if (!checkEnvConfig()) {
        return { error: "环境变量配置不完整" };
      }
      
      try {
        const metaUrl = `https://api.github.com/repos/${CONFIG.GITHUB_REPO}/contents/${CONFIG.GITHUB_FILE_PATH}?ref=${CONFIG.GITHUB_BRANCH}`;
        
        const metaResponse = await fetch(metaUrl, {
          headers: {
            "Authorization": `token ${CONFIG.GITHUB_TOKEN}`,
            "User-Agent": "FriendLinkBot"
          },
          timeout: 10000
        });
        
        if (!metaResponse.ok) {
          const error = await metaResponse.text().catch(() => `状态码: ${metaResponse.status}`);
          return { error: `获取元数据失败: ${error}` };
        }
        
        const metaData = await metaResponse.json();
        const fileSha = metaData.sha;
        
        const contentResponse = await fetch(metaUrl, {
          headers: {
            "Authorization": `token ${CONFIG.GITHUB_TOKEN}`,
            "User-Agent": "FriendLinkBot",
            "Accept": "application/vnd.github.v3.raw"
          },
          timeout: 10000
        });
        
        if (!contentResponse.ok) {
          const error = await contentResponse.text().catch(() => `状态码: ${contentResponse.status}`);
          return { error: `获取内容失败: ${error}` };
        }
        
        return {
          content: await contentResponse.text(),
          sha: fileSha,
          error: null
        };
      } catch (err) {
        return { error: `获取文件错误: ${err.message}` };
      }
    }

    // 更新GitHub文件
    async function updateGitHubFile(content, sha, message) {
      if (!checkEnvConfig()) {
        return false;
      }
      
      try {
        if (!sha) {
          console.error("更新失败：SHA值为空");
          return false;
        }
        
        const url = `https://api.github.com/repos/${CONFIG.GITHUB_REPO}/contents/${CONFIG.GITHUB_FILE_PATH}`;
        const encodedContent = btoa(unescape(encodeURIComponent(content)));
        
        const response = await fetch(url, {
          method: "PUT",
          headers: {
            "Authorization": `token ${CONFIG.GITHUB_TOKEN}`,
            "Content-Type": "application/json",
            "User-Agent": "FriendLinkBot"
          },
          body: JSON.stringify({
            message: message,
            content: encodedContent,
            sha: sha,
            branch: CONFIG.GITHUB_BRANCH
          }),
          timeout: 15000
        });
        
        const responseText = await response.text().catch(() => "无响应内容");
        console.log(`更新响应状态: ${response.status}, 内容: ${responseText}`);
        
        if (!response.ok) return false;
        return true;
      } catch (error) {
        console.error("更新文件异常:", error);
        return false;
      }
    }

    // 命令处理
    async function handleCommand(command, chatId) {
      const [action, ...paramsParts] = command.trim().split(' ') || [];
      const params = paramsParts.join(' ');
      
      if (!checkEnvConfig()) {
        return "❌ 机器人配置不完整，请检查环境变量";
      }
      
      // 帮助命令
      if (!action || action === '帮助' || action === 'help') {
        return `📝 友链机器人命令：
1. 查询 → /友链 查询
2. 添加分类 → /友链 添加分类 名称|样式|后缀|描述
3. 添加友链 → /友链 添加友链 分类序号|名称|link|avatar|descr|siteshot|color|tag|recommend
4. 修改友链 → /友链 修改友链 分类序号|友链序号|...（同添加格式）
5. 删除友链 → /友链 删除友链 分类序号|友链序号
6. 删除分类 → /友链 删除分类 分类序号`;
      }
      
      // 获取友链数据
      const fileData = await getGitHubFile();
      if (!fileData || fileData.error) {
        const errorMsg = fileData?.error || "未知错误";
        return `❌ 无法获取友链数据：${errorMsg}`;
      }
      
      // 解析YAML
      const categories = parseYaml(fileData.content);
      if (!categories) {
        return "❌ 解析友链数据失败";
      }
      
      // 查询命令
      if (action === '查询') {
        let result = "📋 友链列表（完整参数）：\n";
        categories.forEach((cat, catIdx) => {
          result += `\n${catIdx + 1}. 分类：${cat.class_name}\n`;
          result += `   样式：${cat.flink_style || '默认'}\n`;
          result += `   友链数量：${cat.link_list.length}\n`;
          
          if (cat.link_list.length > 0) {
            result += "   友链：\n";
            cat.link_list.forEach((link, linkIdx) => {
              result += `   ${linkIdx + 1}. ${link.name}\n`;
              result += `      链接：${link.link}\n`;
              result += `      头像：${link.avatar}\n`;
              result += `      描述：${link.descr || '无'}\n`;
              if (link.siteshot) result += `      截图：${link.siteshot}\n`;
              if (link.color) result += `      颜色：${link.color}\n`;
              if (link.tag) result += `      标签：${link.tag}\n`;
              if (link.recommend) result += `      推荐：${link.recommend}\n`;
            });
          }
        });
        return result;
      }
      
      // 管理员验证
      if (!isAdmin(chatId)) {
        return "❌ 权限不足，仅管理员可执行增删改";
      }
      
      // 添加分类
      if (action === '添加分类') {
        const [name, flink_style, hundredSuffix = "", class_desc = ""] = params.split('|');
        if (!name || !flink_style) {
          return "❌ 格式错误！需要：名称|flink_style";
        }
        
        if (categories.some(cat => cat.class_name === name)) {
          return "❌ 分类已存在";
        }
        
        categories.push({
          class_name: name,
          flink_style: flink_style,
          hundredSuffix: hundredSuffix,
          class_desc: class_desc,
          link_list: []
        });
        
        const yaml = categoriesToYaml(categories);
        const success = await updateGitHubFile(yaml, fileData.sha, `添加分类: ${name}`);
        return success ? `✅ 添加分类成功：${name}` : "❌ 添加分类失败";
      }
      
      // 添加友链
      if (action === '添加友链') {
        const [
          catIdxStr, name, link, avatar, descr = "", 
          siteshot = "", color = "", tag = "", recommend = ""
        ] = params.split('|');
        const catIdx = parseInt(catIdxStr) - 1;
        
        if (isNaN(catIdx) || catIdx < 0 || catIdx >= categories.length) {
          return "❌ 分类序号错误";
        }
        if (!name || !link || !avatar) {
          return "❌ 格式错误！至少需要：分类序号|名称|link|avatar";
        }
        
        const newLink = { name, link, avatar };
        if (descr) newLink.descr = descr;
        if (siteshot) newLink.siteshot = siteshot;
        if (color) newLink.color = color;
        if (tag) newLink.tag = tag;
        if (recommend) newLink.recommend = recommend;
        
        categories[catIdx].link_list.push(newLink);
        
        const yaml = categoriesToYaml(categories);
        const success = await updateGitHubFile(yaml, fileData.sha, `添加友链: ${name}`);
        return success ? `✅ 成功添加友链到【${categories[catIdx].class_name}】` : "❌ 添加友链失败";
      }
      
      // 修改友链
      if (action === '修改友链') {
        const [
          catIdxStr, linkIdxStr, name, link, avatar, descr = "", 
          siteshot = "", color = "", tag = "", recommend = ""
        ] = params.split('|');
        const catIdx = parseInt(catIdxStr) - 1;
        const linkIdx = parseInt(linkIdxStr) - 1;
        
        if (isNaN(catIdx) || catIdx < 0 || catIdx >= categories.length) return "❌ 分类序号错误";
        const category = categories[catIdx];
        if (isNaN(linkIdx) || linkIdx < 0 || linkIdx >= category.link_list.length) return "❌ 友链序号错误";
        
        const updatedLink = { name, link, avatar };
        if (descr) updatedLink.descr = descr;
        if (siteshot) updatedLink.siteshot = siteshot;
        if (color) updatedLink.color = color;
        if (tag) updatedLink.tag = tag;
        if (recommend) updatedLink.recommend = recommend;
        
        category.link_list[linkIdx] = updatedLink;
        
        const yaml = categoriesToYaml(categories);
        const success = await updateGitHubFile(yaml, fileData.sha, `修改友链: ${name}`);
        return success ? `✅ 修改友链成功` : "❌ 修改友链失败";
      }
      
      // 删除友链
      if (action === '删除友链') {
        const [catIdxStr, linkIdxStr] = params.split('|');
        const catIdx = parseInt(catIdxStr) - 1;
        const linkIdx = parseInt(linkIdxStr) - 1;
        
        if (isNaN(catIdx) || catIdx < 0 || catIdx >= categories.length) return "❌ 分类序号错误";
        const category = categories[catIdx];
        if (isNaN(linkIdx) || linkIdx < 0 || linkIdx >= category.link_list.length) return "❌ 友链序号错误";
        
        const deletedLink = category.link_list.splice(linkIdx, 1)[0];
        
        const yaml = categoriesToYaml(categories);
        const success = await updateGitHubFile(yaml, fileData.sha, `删除友链: ${deletedLink.name}`);
        return success ? `✅ 删除友链成功：${deletedLink.name}` : "❌ 删除友链失败";
      }
      
      // 删除分类
      if (action === '删除分类') {
        const catIdx = parseInt(params) - 1;
        if (isNaN(catIdx) || catIdx < 0 || catIdx >= categories.length) return "❌ 分类序号错误";
        
        const deletedCat = categories.splice(catIdx, 1)[0];
        
        const yaml = categoriesToYaml(categories);
        const success = await updateGitHubFile(yaml, fileData.sha, `删除分类: ${deletedCat.class_name}`);
        return success ? `✅ 删除分类成功：${deletedCat.class_name}` : "❌ 删除分类失败";
      }
      
      return "❌ 未知命令，请发送 /友链 帮助";
    }

    // 处理请求路由
    const url = new URL(request.url);
    
    // 设置Webhook
    if (url.searchParams.has('setwebhook')) {
      if (!checkEnvConfig()) {
        return new Response("环境变量配置不完整", { status: 500 });
      }
      
      const webhookUrl = `https://api.telegram.org/bot${CONFIG.TGBOT_TOKEN}/setWebhook?url=${encodeURIComponent(request.url.replace('?setwebhook', ''))}`;
      const response = await fetch(webhookUrl);
      return new Response(JSON.stringify(await response.json()), {
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // GET请求验证
    if (request.method === "GET") {
      return new Response("友链机器人运行中 ✅", { status: 200 });
    }
    
    // 处理Telegram消息
    if (request.method === "POST") {
      try {
        const tgData = await request.json();
        if (!tgData?.message?.text) {
          return new Response("OK", { status: 200 });
        }
        
        const chatId = tgData.message.chat.id;
        const userText = tgData.message.text.trim();
        
        let command = "";
        if (userText.startsWith('/友链')) {
          command = userText.substring(3).trim();
        } else if (userText === '/help') {
          command = '帮助';
        }
        
        if (command) {
          const reply = await handleCommand(command, chatId);
          await sendTgMessage(chatId, reply);
        }
        
        return new Response("OK", { status: 200 });
      } catch (error) {
        console.error("处理请求错误:", error);
        return new Response("Error", { status: 500 });
      }
    }
    
    return new Response("不支持的方法", { status: 405 });
  }
};

```
3. 关键修改：找到代码中 `CONFIG` 部分，更新2个信息：
   - `GITHUB_REPO`：你的博客仓库路径（如 `newstartsw/hexo-action`，格式：`用户名/仓库名`）
   - `GITHUB_BRANCH`：仓库主分支（一般是 `main`，老仓库可能是 `master`，按自己 GitHub 仓库实际分支改）
4. 点击 **保存并部署**，代码部署完成。


## 🔗 第二步：配置 Telegram Webhook（打通消息通道）
让 Telegram 能把你的命令转发给 Worker 机器人：
1. 复制你的 Worker 域名（代码页面顶部“部署”下方，如 `blog-friendlink-bot.yourname.workers.dev`）。
2. 在浏览器地址栏输入以下链接（替换 `[Worker域名]` 为你的实际域名）：
   ```
   https://[Worker域名]/?setwebhook
   ```
3. 访问后，页面显示 `{"ok":true,"result":true}` 表示配置成功（如果报错，检查 Worker 域名是否正确）。


## 🎯 第三步：测试机器人功能
打开 Telegram，找到你创建的机器人，发送命令测试，所有操作实时同步到 GitHub 友链文件：

### 1. 基础命令（必测）
| 命令 | 效果 | 示例 |
|------|------|------|
| `/help` | 查看所有支持的命令 | 发送后会返回完整命令列表 |
| `/友链 查询` | 查看当前所有友链（含分类、完整参数） | 正常会显示“框架”“推荐博客”等分类及下属友链 |

### 2. 核心功能（增删改）
#### （1）添加分类
```
/友链 添加分类 分类名称|样式|后缀|分类描述（可选）
```
示例：`/友链 添加分类 技术博客|flexcard|""|程序员博客合集`  
→ 成功后发送 `/友链 查询` 可看到新分类。

#### （2）添加友链
```
/友链 添加友链 分类序号|友链名称|友链URL|头像URL|描述|截图URL（可选）|颜色标签（可选）|技术标签（可选）|是否推荐（true/false，可选）
```
示例：`/友链 添加友链 1|安知鱼博客|https://blog.anheyu.com/|https://avatar.jpg|生活明朗，万物可爱|https://shot.jpg|vip|技术|true`  
→ 序号从1开始，分类序号可通过 `/友链 查询` 查看。

#### （3）修改友链
```
/友链 修改友链 分类序号|友链序号|新名称|新URL|新头像|新描述|新截图|新颜色|新标签|新推荐状态
```
示例：`/友链 修改友链 1|1|安知鱼新版|https://blog.anheyu.com/|https://new-avatar.jpg|生活明朗，万物可爱|https://new-shot.jpg|vip|技术|true`  
→ 不需要修改的参数也需保留位置（空值可填 `""`）。

#### （4）删除友链/分类
```
# 删除友链
/友链 删除友链 分类序号|友链序号

# 删除分类（会删除分类下所有友链）
/友链 删除分类 分类序号
```
示例：`/友链 删除友链 1|2`（删除第1分类第2条友链）


## 🚨 常见问题排查
1. **发送命令无响应？**
   - 检查 Webhook 配置：重新访问 `https://[Worker域名]/?setwebhook`，确保返回 `ok: true`。
   - 查看 Worker 日志：代码页面顶部 **日志** 标签页，查看是否有错误（如“GitHub Token 无效”）。

2. **“无法获取友链数据”？**
   - 检查 `GITHUB_REPO` 是否正确（用户名/仓库名不要写错）。
   - 确认 GitHub Token 有 `repo` 权限（重新生成 Token 时务必勾选）。

3. **增删改提示“失败”？**
   - 检查分类/友链序号是否正确（序号从1开始，不是0）。
   - 确认你发送命令的 Telegram 账号 ID 与 `ADMIN_CHAT_ID` 一致（非管理员无权限）。


## ✨ 最终效果
- 远程管理：手机上发送命令，即可实时更新博客友链，无需手动改 YAML 文件。
- 安全可控：仅管理员能操作，敏感信息加密存储，无泄露风险。
- 完整兼容：支持 `siteshot`（截图）、`color`（标签色）等所有友链参数。

按以上步骤操作，10分钟内即可搭建完成，后续维护友链再也不用打开仓库修改啦！
