---
title: cherry studio v1.1.7 配置mcp
date: 2025-03-18 11:11:39
categories:
    - cherry-studio
tags:
    - mcp
---

# Problem

安装 cherry-studio v1.1.7，在 MCP Server 的设置界面能看到如下提示。由于本地已有`uv`和`bun`，因此不想在此处再次安装。

![](/images/2025/0abc4d171f93723038a1781cf228a51e_MD5.jpeg)

根据[如何在 Cherry Studio 中使用 MCP](https://vaayne.com/posts/2025/how-to-use-mcp-in-cherry-studio/)中的描述，配置了`uv`/`bun`的绝对路径。

![](/images/2025/82c7d39492238841c08c1c76f1c36f8c_MD5.jpeg)

但无法添加 server，报错为，

```
Error invoking remote method 'mcp:add-server': McpError: MCP error -32000: Connection closed.
```

terminal 中直接运行是正常的，

![](/images/2025/f6c1479fbe6a6ca493b3b3c752be8d00_MD5.jpeg)

翻了一下[\[BUG\]: MCP 错误反馈汇总 · Issue #3264 · CherryHQ/cherry-studio](https://github.com/CherryHQ/cherry-studio/issues/3264)，猜测是路径的问题。查看源码找到判断 binary 是否安装的逻辑如下，

```
export async function getBinaryPath(name: string): Promise<string> {
  let cmd = process.platform === 'win32' ? `${name}.exe` : name
  const binariesDir = path.join(os.homedir(), '.cherrystudio', 'bin')
  const binariesDirExists = await fs.existsSync(binariesDir)
  cmd = binariesDirExists ? path.join(binariesDir, cmd) : name
  return cmd
}

export async function isBinaryExists(name: string): Promise<boolean> {
  const cmd = await getBinaryPath(name)
  return await fs.existsSync(cmd)
}
```

因此 cherry-studio 是在`~/cherrystudio/bin`路径判断是否存在`uv`/`bun`，那加上软链即可。

# Solution

```bash
mkdir -p ~/.cherrystudio/bin
cd ~/.cherrystudio/bin

ln -s /path/to/npx npx
ln -s /opt/homebrew/bin/uv uv
ln -s /opt/homebrew/bin/bun bun
ln -s /opt/homebrew/bin/bunx bunx
ln -s /opt/homebrew/bin/uvx uvx
```
