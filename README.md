# nvim-lsp-ts-utils

Utilities to improve the TypeScript development experience for Neovim's
built-in LSP client.

## Motivation

VS Code and [coc-tsserver](https://github.com/neoclide/coc-tsserver) are great
for TypeScript, so great that other LSP implementations don't give TypeScript a
lot of love. This is an attempt to rectify that, bit by bit.

## Features

- Organize imports (exposed as `:TSLspOrganize`)

  Async by default, but a sync variant is available and exposed as
  `:TSLspOrganizeSync` (useful for running on save).

- Fix current (exposed as `:TSLspFixCurrent`)

  A simple way to apply the first available code action to the current line
  without confirmation.

- Rename file and update imports (exposed as `:TSLspRenameFile`)

  One of my most missed features from VS Code / coc.nvim. Enter a new path
  (based on the current file's path) and watch the magic happen.

- Import all missing imports (exposed as `:TSLspImportAll`)

  Gets all code actions, then matches against the action's title to
  (imperfectly) determine whether it's an import action. Also organizes imports
  afterwards to merge imports from the same source.

  If you have [plenary.nvim](https://github.com/nvim-lua/plenary.nvim)
  installed, the function will run asynchronously, which provides a big
  performance and reliability boost. If not, it'll run slower and may time out.

- Import on completion

  Adds missing imports on completion confirm (`<C-y>`) when using the built-in
  LSP `omnifunc` (which is itself enabled by setting
  `vim.bo.omnifunc = "v:lua.vim.lsp.omnifunc"` somewhere in your LSP config).

  Enable by setting `enable_import_on_completion` to `true` inside `setup` (see
  below).

  Binding `.` in insert mode to `.<C-x><C-o>` can trigger imports twice. The
  plugin sets a timeout to avoid importing the same item twice in a short span
  of time, which you can change by setting `import_on_completion_timeout` in
  your setup function (`0` disables this behavior).

- ESLint code actions (experimental)

  Parses ESLint JSON output for the current file and converts possible fixes
  into code actions, like [the VS Code
  plugin](https://github.com/Microsoft/vscode-eslint). Experimental! Feedback
  and contributions greatly appreciated.

  Works with Neovim's built-in code action handler as well as plugins like
  [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) and
  [lspsaga.nvim](https://github.com/glepnir/lspsaga.nvim). See the setup section
  below for instructions.

  Supports the following settings:

  - `eslint_binary`: sets the binary used to get ESLint output. Uses `eslint` by
    default for compatibility, but I highly, highly recommend using
    [eslint_d](https://github.com/mantoni/eslint_d.js).

  - `eslint_enable_disable_comments`: enables ESLint code actions to disable the
    violated rule for the current line / file. Set to `true` by default.

## Setup

Install using your favorite plugin manager and add to your
[nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) `tsserver.setup` function.

An example showing the available settings and their defaults:

```lua
local nvim_lsp = require("lspconfig")

nvim_lsp.tsserver.setup {
    on_attach = function(_, bufnr)
        local ts_utils = require("nvim-lsp-ts-utils")

        require("nvim-lsp-ts-utils").setup {
            -- defaults
            disable_commands = false,
            enable_import_on_completion = false,
            import_on_completion_timeout = 5000,
            eslint_bin = "eslint", -- use eslint_d if possible!
            eslint_enable_disable_comments = true,
            request_handlers = {}
        }

        -- no default maps, so you may want to define some here
        vim.api.nvim_buf_set_keymap(bufnr, "n", "gs", ":TSLspOrganize<CR>", {silent = true})
        vim.api.nvim_buf_set_keymap(bufnr, "n", "qq", ":TSLspFixCurrent<CR>", {silent = true})
        vim.api.nvim_buf_set_keymap(bufnr, "n", "gr", ":TSLspRenameFile<CR>", {silent = true})
        vim.api.nvim_buf_set_keymap(bufnr, "n", "gi", ":TSLspImportAll<CR>", {silent = true})
    end
}
```

To enable ESLint code actions, use the following settings:

```lua
ts_utils.setup {
    -- pass alongside other settings
    request_handlers = {vim.lsp.buf_request, vim.lsp.buf_request_sync}
}

vim.lsp.buf_request_sync = ts_utils.buf_request_sync
vim.lsp.buf_request = ts_utils.buf_request
```

By default, the plugin will define Vim commands for convenience. You can
disable this by passing `disable_commands = true` into `setup` and then calling
the Lua functions directly:

- Organize imports: `:lua require'nvim-lsp-ts-utils'.organize_imports()`
- Fix current: `:lua require'nvim-lsp-ts-utils'.fix_current()`
- Rename file: `:lua require'nvim-lsp-ts-utils'.rename_file()`
- Import all: `:lua require'nvim-lsp-ts-utils'.import_all()`

## Tests

I've covered most of the current functions with LSP integration tests using
[plenary.nvim](https://github.com/nvim-lua/plenary.nvim), which you can run by
running `./test.sh`.

## Goals

- [ ] ESLint code action feature parity with [vscode-eslint](https://github.com/microsoft/vscode-eslint)

  The VS Code plugin supports 3 more code actions: `applySameFixes`,
  `applyAllFixes`, and `openRuleDoc`. Implementing them shouldn't be too hard
  (though `openRuleDoc` should be opt-in, since it requires ESLint to use the
  heavier `json-with-metadata` format).

- [ ] TSLint / stylelint code action support?

  I'm not familiar with these at all, but since they both support CLI JSON
  output, the plugin should be able to handle them in the same way it handles
  ESLint. I'm a little concerned about speed and handling output from more than
  one linter, so I'd appreciate input from users of these linters.

- [ ] Watch project files and update imports on change.

  I've prototyped this using plenary jobs and
  [Watchman](https://facebook.github.io/watchman/) and am waiting for Plenary to
  merge async await jobs to make working with Watchman less painful.
