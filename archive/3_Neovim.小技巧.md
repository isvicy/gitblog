# [Neovim 小技巧](https://github.com/isvicy/gitblog/issues/3)

**本文旨在记录自己在使用 Neovim Debug 以及和 AI 结对编程的过程中使用的一些有助于提升效率的小技巧。**

### 1. 在 Neovim 中编译 Go 程序，并且将编译失败的结果同步到 quicklist 窗口
#### 场景
1. 在频繁编辑的项目里，由于可能存在语法错误，Trouble 这类聚合 project 级别错误信息的插件可能会短暂不工作，此时可以使用此功能来实现和 Trouble Diagnosis 一样的效果；
2. Go Build 可以作为实际检验代码是否还有语法错误的最后一环；

#### code piece
```lua
-- Run go build asynchronously and populate errors to quickfix.
-- This should be used as a supplement of trouble diagnostics.
local function async_go_build_quickfix()
  -- Clear the quickfix list first
  vim.fn.setqflist({}, "r")

  -- Show a message that build started
  vim.notify("Go build started...", vim.log.levels.INFO)

  -- Create jobs for async execution
  local stderr_data = {}
  local job = vim.fn.jobstart("go build ./...", {
    on_stderr = function(_, data)
      if data then
        for _, line in ipairs(data) do
          if line ~= "" then
            table.insert(stderr_data, line)
          end
        end
      end
    end,
    on_exit = function(_, exit_code)
      if exit_code == 0 then
        vim.schedule(function()
          vim.notify("Build successful!", vim.log.levels.INFO)
        end)
        return
      end

      -- Parse the error output
      local errors = {}
      for _, line in ipairs(stderr_data) do
        -- Match Go error format: file:line:col: message
        local file, lnum, col, msg = line:match("(.+):(%d+):(%d+): (.*)")
        if file and lnum and col and msg then
          table.insert(errors, {
            filename = file,
            lnum = tonumber(lnum),
            col = tonumber(col),
            text = msg,
            type = "E",
          })
        end
      end

      -- Update quickfix in the main thread
      vim.schedule(function()
        vim.fn.setqflist(errors)
        if #errors > 0 then
          vim.cmd("copen")
          vim.notify("Build failed! Check quickfix list.", vim.log.levels.ERROR)
        end
      end)
    end,
  })

  if job == 0 then
    vim.notify("Failed to start go build", vim.log.levels.ERROR)
  elseif job == -1 then
    vim.notify("Invalid arguments for go build", vim.log.levels.ERROR)
  end
end
-- Create a command to run the build
vim.api.nvim_create_user_command("GoBuild", async_go_build_quickfix, {})
-- Create keymap
vim.keymap.set("n", "<Leader>cgb", ":GoBuild<CR>", { silent = true, noremap = true })
```
#### 使用说明
将此段代码加入到你原本存放 auto cmd 的文件里，之后使用 `<leader>cgb` (cgb -> code golang build) 即可达到标题里的效果。

### 2. 快速复制当前行的 diagnostic 信息
#### 场景
当和浏览器端的 AI 结对编程时，此功能可以快速复制错误提示，方便快速粘贴，提供信息给 AI，供其理解后修复。

```lua
-- Copy diagnostic info to system clipboard.
-- This could be useful when you chat with AI Models to supply context.
local function copy_diagnostic()
  local current_line = vim.fn.line(".") - 1
  local current_buf = vim.api.nvim_get_current_buf()

  local diagnostics = vim.diagnostic.get(current_buf, { lnum = current_line })
  if #diagnostics == 0 then
    vim.notify("No diagnostics", vim.log.levels.WARN, { title = "Copy" })
    return
  end

  local messages = {}
  for _, diagnostic in ipairs(diagnostics) do
    table.insert(messages, diagnostic.message)
  end
  local diagnostic_text = table.concat(messages, "\n")

  -- Copy to system clipboard and unnamed register
  vim.fn.setreg("+", diagnostic_text)
  vim.fn.setreg('"', diagnostic_text)

  -- Use shorter notification without the actual text
  vim.notify("Diagnostic copied", vim.log.levels.INFO, { title = "Copy" })
end

-- Create command with silent option
vim.api.nvim_create_user_command("CopyDiagnostic", copy_diagnostic, {})
-- Add keymapping with silent option
vim.keymap.set("n", "<leader>ccd", function()
  copy_diagnostic()
end, {
  silent = true, -- This ensures no "Press ENTER" prompt
  desc = "Copy diagnostic message",
})
```
#### 使用说明
将此段代码放置到你存放 auto cmd 的文件内，然后使用 `<leader>ccd` (ccd -> code copy diagnostic) 即可复制当前行的错误信息到剪切板。