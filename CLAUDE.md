（交流可以用英文，本文档中文，保留这句）

# 项目说明

## 项目目标
编写一份本 repo 实现细节的教程markdown，要包含细致的多角度分析
第一部分是按模块切分为多个 chapter
第二部分是按功能和场景（跨模块）切分为多个 chapter
整体组织为 index.md + chapter1.md + ...
repo 本身的概述见 README.md

## 章节结构要求

每个章节应包含：
1. **开篇段落**：简要介绍本章所讨论模块或功能
2. **本章小结**：总结关键概念

## Source map
.
├── CLAUDE.md
├── CONTRIBUTING.md
├── Dockerfile
├── GEMINI.md
├── LICENSE
├── Makefile
├── README.md
├── ROADMAP.md
├── esbuild.config.js
├── eslint.config.js
├── index.md
├── integration-tests
│   ├── file-system.test.js
│   ├── google_web_search.test.js
│   ├── list_directory.test.js
│   ├── read_many_files.test.js
│   ├── replace.test.js
│   ├── run-tests.js
│   ├── run_shell_command.test.js
│   ├── save_memory.test.js
│   ├── simple-mcp-server.test.js
│   ├── test-helper.js
│   └── write_file.test.js
├── package-lock.json
├── package.json
├── packages
│   ├── cli
│   │   ├── index.ts
│   │   ├── package.json
│   │   ├── src
│   │   │   ├── acp
│   │   │   │   ├── acp.ts
│   │   │   │   └── acpPeer.ts
│   │   │   ├── config
│   │   │   │   ├── auth.test.ts
│   │   │   │   ├── auth.ts
│   │   │   │   ├── config.integration.test.ts
│   │   │   │   ├── config.test.ts
│   │   │   │   ├── config.ts
│   │   │   │   ├── extension.test.ts
│   │   │   │   ├── extension.ts
│   │   │   │   ├── sandboxConfig.ts
│   │   │   │   ├── settings.test.ts
│   │   │   │   └── settings.ts
│   │   │   ├── gemini.test.tsx
│   │   │   ├── gemini.tsx
│   │   │   ├── nonInteractiveCli.test.ts
│   │   │   ├── nonInteractiveCli.ts
│   │   │   ├── patches
│   │   │   │   └── is-in-ci.ts
│   │   │   ├── services
│   │   │   │   ├── BuiltinCommandLoader.test.ts
│   │   │   │   ├── BuiltinCommandLoader.ts
│   │   │   │   ├── CommandService.test.ts
│   │   │   │   ├── CommandService.ts
│   │   │   │   ├── FileCommandLoader.test.ts
│   │   │   │   ├── FileCommandLoader.ts
│   │   │   │   ├── McpPromptLoader.ts
│   │   │   │   ├── prompt-processors
│   │   │   │   │   ├── argumentProcessor.test.ts
│   │   │   │   │   ├── argumentProcessor.ts
│   │   │   │   │   ├── shellProcessor.test.ts
│   │   │   │   │   ├── shellProcessor.ts
│   │   │   │   │   └── types.ts
│   │   │   │   └── types.ts
│   │   │   ├── test-utils
│   │   │   │   ├── mockCommandContext.test.ts
│   │   │   │   └── mockCommandContext.ts
│   │   │   ├── ui
│   │   │   │   ├── App.test.tsx
│   │   │   │   ├── App.tsx
│   │   │   │   ├── __snapshots__
│   │   │   │   │   └── App.test.tsx.snap
│   │   │   │   ├── colors.ts
│   │   │   │   ├── commands
│   │   │   │   │   ├── aboutCommand.test.ts
│   │   │   │   │   ├── aboutCommand.ts
│   │   │   │   │   ├── authCommand.test.ts
│   │   │   │   │   ├── authCommand.ts
│   │   │   │   │   ├── bugCommand.test.ts
│   │   │   │   │   ├── bugCommand.ts
│   │   │   │   │   ├── chatCommand.test.ts
│   │   │   │   │   ├── chatCommand.ts
│   │   │   │   │   ├── clearCommand.test.ts
│   │   │   │   │   ├── clearCommand.ts
│   │   │   │   │   ├── compressCommand.test.ts
│   │   │   │   │   ├── compressCommand.ts
│   │   │   │   │   ├── copyCommand.test.ts
│   │   │   │   │   ├── copyCommand.ts
│   │   │   │   │   ├── corgiCommand.test.ts
│   │   │   │   │   ├── corgiCommand.ts
│   │   │   │   │   ├── docsCommand.test.ts
│   │   │   │   │   ├── docsCommand.ts
│   │   │   │   │   ├── editorCommand.test.ts
│   │   │   │   │   ├── editorCommand.ts
│   │   │   │   │   ├── extensionsCommand.test.ts
│   │   │   │   │   ├── extensionsCommand.ts
│   │   │   │   │   ├── helpCommand.test.ts
│   │   │   │   │   ├── helpCommand.ts
│   │   │   │   │   ├── ideCommand.test.ts
│   │   │   │   │   ├── ideCommand.ts
│   │   │   │   │   ├── initCommand.test.ts
│   │   │   │   │   ├── initCommand.ts
│   │   │   │   │   ├── mcpCommand.test.ts
│   │   │   │   │   ├── mcpCommand.ts
│   │   │   │   │   ├── memoryCommand.test.ts
│   │   │   │   │   ├── memoryCommand.ts
│   │   │   │   │   ├── privacyCommand.test.ts
│   │   │   │   │   ├── privacyCommand.ts
│   │   │   │   │   ├── quitCommand.test.ts
│   │   │   │   │   ├── quitCommand.ts
│   │   │   │   │   ├── restoreCommand.test.ts
│   │   │   │   │   ├── restoreCommand.ts
│   │   │   │   │   ├── statsCommand.test.ts
│   │   │   │   │   ├── statsCommand.ts
│   │   │   │   │   ├── themeCommand.test.ts
│   │   │   │   │   ├── themeCommand.ts
│   │   │   │   │   ├── toolsCommand.test.ts
│   │   │   │   │   ├── toolsCommand.ts
│   │   │   │   │   ├── types.ts
│   │   │   │   │   └── vimCommand.ts
│   │   │   │   ├── components
│   │   │   │   │   ├── AboutBox.tsx
│   │   │   │   │   ├── AsciiArt.ts
│   │   │   │   │   ├── AuthDialog.test.tsx
│   │   │   │   │   ├── AuthDialog.tsx
│   │   │   │   │   ├── AuthInProgress.tsx
│   │   │   │   │   ├── AutoAcceptIndicator.tsx
│   │   │   │   │   ├── ConsoleSummaryDisplay.tsx
│   │   │   │   │   ├── ContextSummaryDisplay.tsx
│   │   │   │   │   ├── DetailedMessagesDisplay.tsx
│   │   │   │   │   ├── EditorSettingsDialog.tsx
│   │   │   │   │   ├── Footer.tsx
│   │   │   │   │   ├── GeminiRespondingSpinner.tsx
│   │   │   │   │   ├── Header.tsx
│   │   │   │   │   ├── Help.tsx
│   │   │   │   │   ├── HistoryItemDisplay.test.tsx
│   │   │   │   │   ├── HistoryItemDisplay.tsx
│   │   │   │   │   ├── IDEContextDetailDisplay.tsx
│   │   │   │   │   ├── InputPrompt.test.tsx
│   │   │   │   │   ├── InputPrompt.tsx
│   │   │   │   │   ├── LoadingIndicator.test.tsx
│   │   │   │   │   ├── LoadingIndicator.tsx
│   │   │   │   │   ├── MemoryUsageDisplay.tsx
│   │   │   │   │   ├── ModelStatsDisplay.test.tsx
│   │   │   │   │   ├── ModelStatsDisplay.tsx
│   │   │   │   │   ├── SessionSummaryDisplay.test.tsx
│   │   │   │   │   ├── SessionSummaryDisplay.tsx
│   │   │   │   │   ├── ShellConfirmationDialog.test.tsx
│   │   │   │   │   ├── ShellConfirmationDialog.tsx
│   │   │   │   │   ├── ShellModeIndicator.tsx
│   │   │   │   │   ├── ShowMoreLines.tsx
│   │   │   │   │   ├── StatsDisplay.test.tsx
│   │   │   │   │   ├── StatsDisplay.tsx
│   │   │   │   │   ├── SuggestionsDisplay.tsx
│   │   │   │   │   ├── ThemeDialog.tsx
│   │   │   │   │   ├── Tips.tsx
│   │   │   │   │   ├── ToolStatsDisplay.test.tsx
│   │   │   │   │   ├── ToolStatsDisplay.tsx
│   │   │   │   │   ├── UpdateNotification.tsx
│   │   │   │   │   ├── __snapshots__
│   │   │   │   │   │   ├── ModelStatsDisplay.test.tsx.snap
│   │   │   │   │   │   ├── SessionSummaryDisplay.test.tsx.snap
│   │   │   │   │   │   ├── ShellConfirmationDialog.test.tsx.snap
│   │   │   │   │   │   ├── StatsDisplay.test.tsx.snap
│   │   │   │   │   │   └── ToolStatsDisplay.test.tsx.snap
│   │   │   │   │   ├── messages
│   │   │   │   │   │   ├── CompressionMessage.tsx
│   │   │   │   │   │   ├── DiffRenderer.test.tsx
│   │   │   │   │   │   ├── DiffRenderer.tsx
│   │   │   │   │   │   ├── ErrorMessage.tsx
│   │   │   │   │   │   ├── GeminiMessage.tsx
│   │   │   │   │   │   ├── GeminiMessageContent.tsx
│   │   │   │   │   │   ├── InfoMessage.tsx
│   │   │   │   │   │   ├── ToolConfirmationMessage.test.tsx
│   │   │   │   │   │   ├── ToolConfirmationMessage.tsx
│   │   │   │   │   │   ├── ToolGroupMessage.tsx
│   │   │   │   │   │   ├── ToolMessage.test.tsx
│   │   │   │   │   │   ├── ToolMessage.tsx
│   │   │   │   │   │   ├── UserMessage.tsx
│   │   │   │   │   │   └── UserShellMessage.tsx
│   │   │   │   │   └── shared
│   │   │   │   │       ├── MaxSizedBox.test.tsx
│   │   │   │   │       ├── MaxSizedBox.tsx
│   │   │   │   │       ├── RadioButtonSelect.test.tsx
│   │   │   │   │       ├── RadioButtonSelect.tsx
│   │   │   │   │       ├── __snapshots__
│   │   │   │   │       │   └── RadioButtonSelect.test.tsx.snap
│   │   │   │   │       ├── text-buffer.test.ts
│   │   │   │   │       ├── text-buffer.ts
│   │   │   │   │       ├── vim-buffer-actions.test.ts
│   │   │   │   │       └── vim-buffer-actions.ts
│   │   │   │   ├── constants.ts
│   │   │   │   ├── contexts
│   │   │   │   │   ├── OverflowContext.tsx
│   │   │   │   │   ├── SessionContext.test.tsx
│   │   │   │   │   ├── SessionContext.tsx
│   │   │   │   │   ├── StreamingContext.tsx
│   │   │   │   │   └── VimModeContext.tsx
│   │   │   │   ├── editors
│   │   │   │   │   └── editorSettingsManager.ts
│   │   │   │   ├── hooks
│   │   │   │   │   ├── atCommandProcessor.test.ts
│   │   │   │   │   ├── atCommandProcessor.ts
│   │   │   │   │   ├── shellCommandProcessor.test.ts
│   │   │   │   │   ├── shellCommandProcessor.ts
│   │   │   │   │   ├── slashCommandProcessor.test.ts
│   │   │   │   │   ├── slashCommandProcessor.ts
│   │   │   │   │   ├── useAuthCommand.ts
│   │   │   │   │   ├── useAutoAcceptIndicator.test.ts
│   │   │   │   │   ├── useAutoAcceptIndicator.ts
│   │   │   │   │   ├── useBracketedPaste.ts
│   │   │   │   │   ├── useCompletion.test.ts
│   │   │   │   │   ├── useCompletion.ts
│   │   │   │   │   ├── useConsoleMessages.test.ts
│   │   │   │   │   ├── useConsoleMessages.ts
│   │   │   │   │   ├── useEditorSettings.test.ts
│   │   │   │   │   ├── useEditorSettings.ts
│   │   │   │   │   ├── useFocus.test.ts
│   │   │   │   │   ├── useFocus.ts
│   │   │   │   │   ├── useGeminiStream.test.tsx
│   │   │   │   │   ├── useGeminiStream.ts
│   │   │   │   │   ├── useGitBranchName.test.ts
│   │   │   │   │   ├── useGitBranchName.ts
│   │   │   │   │   ├── useHistoryManager.test.ts
│   │   │   │   │   ├── useHistoryManager.ts
│   │   │   │   │   ├── useInputHistory.test.ts
│   │   │   │   │   ├── useInputHistory.ts
│   │   │   │   │   ├── useKeypress.test.ts
│   │   │   │   │   ├── useKeypress.ts
│   │   │   │   │   ├── useLoadingIndicator.test.ts
│   │   │   │   │   ├── useLoadingIndicator.ts
│   │   │   │   │   ├── useLogger.ts
│   │   │   │   │   ├── usePhraseCycler.test.ts
│   │   │   │   │   ├── usePhraseCycler.ts
│   │   │   │   │   ├── usePrivacySettings.ts
│   │   │   │   │   ├── useReactToolScheduler.ts
│   │   │   │   │   ├── useRefreshMemoryCommand.ts
│   │   │   │   │   ├── useShellHistory.test.ts
│   │   │   │   │   ├── useShellHistory.ts
│   │   │   │   │   ├── useShowMemoryCommand.ts
│   │   │   │   │   ├── useStateAndRef.ts
│   │   │   │   │   ├── useTerminalSize.ts
│   │   │   │   │   ├── useThemeCommand.ts
│   │   │   │   │   ├── useTimer.test.ts
│   │   │   │   │   ├── useTimer.ts
│   │   │   │   │   ├── useToolScheduler.test.ts
│   │   │   │   │   ├── vim.test.ts
│   │   │   │   │   └── vim.ts
│   │   │   │   ├── privacy
│   │   │   │   │   ├── CloudFreePrivacyNotice.tsx
│   │   │   │   │   ├── CloudPaidPrivacyNotice.tsx
│   │   │   │   │   ├── GeminiPrivacyNotice.tsx
│   │   │   │   │   └── PrivacyNotice.tsx
│   │   │   │   ├── themes
│   │   │   │   │   ├── ansi-light.ts
│   │   │   │   │   ├── ansi.ts
│   │   │   │   │   ├── atom-one-dark.ts
│   │   │   │   │   ├── ayu-light.ts
│   │   │   │   │   ├── ayu.ts
│   │   │   │   │   ├── color-utils.test.ts
│   │   │   │   │   ├── color-utils.ts
│   │   │   │   │   ├── default-light.ts
│   │   │   │   │   ├── default.ts
│   │   │   │   │   ├── dracula.ts
│   │   │   │   │   ├── github-dark.ts
│   │   │   │   │   ├── github-light.ts
│   │   │   │   │   ├── googlecode.ts
│   │   │   │   │   ├── no-color.ts
│   │   │   │   │   ├── shades-of-purple.ts
│   │   │   │   │   ├── theme-manager.test.ts
│   │   │   │   │   ├── theme-manager.ts
│   │   │   │   │   ├── theme.test.ts
│   │   │   │   │   ├── theme.ts
│   │   │   │   │   └── xcode.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── utils
│   │   │   │       ├── CodeColorizer.tsx
│   │   │   │       ├── ConsolePatcher.ts
│   │   │   │       ├── InlineMarkdownRenderer.tsx
│   │   │   │       ├── MarkdownDisplay.test.tsx
│   │   │   │       ├── MarkdownDisplay.tsx
│   │   │   │       ├── TableRenderer.tsx
│   │   │   │       ├── __snapshots__
│   │   │   │       │   └── MarkdownDisplay.test.tsx.snap
│   │   │   │       ├── clipboardUtils.test.ts
│   │   │   │       ├── clipboardUtils.ts
│   │   │   │       ├── commandUtils.test.ts
│   │   │   │       ├── commandUtils.ts
│   │   │   │       ├── computeStats.test.ts
│   │   │   │       ├── computeStats.ts
│   │   │   │       ├── displayUtils.test.ts
│   │   │   │       ├── displayUtils.ts
│   │   │   │       ├── errorParsing.test.ts
│   │   │   │       ├── errorParsing.ts
│   │   │   │       ├── formatters.test.ts
│   │   │   │       ├── formatters.ts
│   │   │   │       ├── markdownUtilities.test.ts
│   │   │   │       ├── markdownUtilities.ts
│   │   │   │       ├── textUtils.ts
│   │   │   │       ├── updateCheck.test.ts
│   │   │   │       └── updateCheck.ts
│   │   │   ├── utils
│   │   │   │   ├── cleanup.ts
│   │   │   │   ├── events.ts
│   │   │   │   ├── handleAutoUpdate.test.ts
│   │   │   │   ├── handleAutoUpdate.ts
│   │   │   │   ├── installationInfo.test.ts
│   │   │   │   ├── installationInfo.ts
│   │   │   │   ├── package.ts
│   │   │   │   ├── readStdin.ts
│   │   │   │   ├── sandbox-macos-permissive-closed.sb
│   │   │   │   ├── sandbox-macos-permissive-open.sb
│   │   │   │   ├── sandbox-macos-permissive-proxied.sb
│   │   │   │   ├── sandbox-macos-restrictive-closed.sb
│   │   │   │   ├── sandbox-macos-restrictive-open.sb
│   │   │   │   ├── sandbox-macos-restrictive-proxied.sb
│   │   │   │   ├── sandbox.ts
│   │   │   │   ├── startupWarnings.test.ts
│   │   │   │   ├── startupWarnings.ts
│   │   │   │   ├── updateEventEmitter.ts
│   │   │   │   ├── userStartupWarnings.test.ts
│   │   │   │   ├── userStartupWarnings.ts
│   │   │   │   └── version.ts
│   │   │   ├── validateNonInterActiveAuth.test.ts
│   │   │   └── validateNonInterActiveAuth.ts
│   │   ├── tsconfig.json
│   │   └── vitest.config.ts
│   ├── core
│   │   ├── index.ts
│   │   ├── package-lock.json
│   │   ├── package.json
│   │   ├── src
│   │   │   ├── __mocks__
│   │   │   │   └── fs
│   │   │   │       └── promises.ts
│   │   │   ├── code_assist
│   │   │   │   ├── codeAssist.ts
│   │   │   │   ├── converter.test.ts
│   │   │   │   ├── converter.ts
│   │   │   │   ├── oauth2.test.ts
│   │   │   │   ├── oauth2.ts
│   │   │   │   ├── server.test.ts
│   │   │   │   ├── server.ts
│   │   │   │   ├── setup.test.ts
│   │   │   │   ├── setup.ts
│   │   │   │   └── types.ts
│   │   │   ├── config
│   │   │   │   ├── config.test.ts
│   │   │   │   ├── config.ts
│   │   │   │   ├── flashFallback.test.ts
│   │   │   │   └── models.ts
│   │   │   ├── core
│   │   │   │   ├── __snapshots__
│   │   │   │   │   └── prompts.test.ts.snap
│   │   │   │   ├── client.test.ts
│   │   │   │   ├── client.ts
│   │   │   │   ├── contentGenerator.test.ts
│   │   │   │   ├── contentGenerator.ts
│   │   │   │   ├── coreToolScheduler.test.ts
│   │   │   │   ├── coreToolScheduler.ts
│   │   │   │   ├── geminiChat.test.ts
│   │   │   │   ├── geminiChat.ts
│   │   │   │   ├── geminiRequest.ts
│   │   │   │   ├── logger.test.ts
│   │   │   │   ├── logger.ts
│   │   │   │   ├── modelCheck.ts
│   │   │   │   ├── nonInteractiveToolExecutor.test.ts
│   │   │   │   ├── nonInteractiveToolExecutor.ts
│   │   │   │   ├── prompts.test.ts
│   │   │   │   ├── prompts.ts
│   │   │   │   ├── tokenLimits.ts
│   │   │   │   ├── turn.test.ts
│   │   │   │   └── turn.ts
│   │   │   ├── ide
│   │   │   │   ├── ide-client.ts
│   │   │   │   ├── ideContext.test.ts
│   │   │   │   └── ideContext.ts
│   │   │   ├── index.test.ts
│   │   │   ├── index.ts
│   │   │   ├── mcp
│   │   │   │   ├── google-auth-provider.test.ts
│   │   │   │   ├── google-auth-provider.ts
│   │   │   │   ├── oauth-provider.test.ts
│   │   │   │   ├── oauth-provider.ts
│   │   │   │   ├── oauth-token-storage.test.ts
│   │   │   │   ├── oauth-token-storage.ts
│   │   │   │   ├── oauth-utils.test.ts
│   │   │   │   └── oauth-utils.ts
│   │   │   ├── prompts
│   │   │   │   ├── mcp-prompts.ts
│   │   │   │   └── prompt-registry.ts
│   │   │   ├── services
│   │   │   │   ├── fileDiscoveryService.test.ts
│   │   │   │   ├── fileDiscoveryService.ts
│   │   │   │   ├── gitService.test.ts
│   │   │   │   ├── gitService.ts
│   │   │   │   ├── loopDetectionService.test.ts
│   │   │   │   ├── loopDetectionService.ts
│   │   │   │   ├── shellExecutionService.test.ts
│   │   │   │   └── shellExecutionService.ts
│   │   │   ├── telemetry
│   │   │   │   ├── clearcut-logger
│   │   │   │   │   ├── clearcut-logger.ts
│   │   │   │   │   └── event-metadata-key.ts
│   │   │   │   ├── constants.ts
│   │   │   │   ├── file-exporters.ts
│   │   │   │   ├── index.ts
│   │   │   │   ├── integration.test.circular.ts
│   │   │   │   ├── loggers.test.circular.ts
│   │   │   │   ├── loggers.test.ts
│   │   │   │   ├── loggers.ts
│   │   │   │   ├── metrics.test.ts
│   │   │   │   ├── metrics.ts
│   │   │   │   ├── sdk.ts
│   │   │   │   ├── telemetry.test.ts
│   │   │   │   ├── types.ts
│   │   │   │   ├── uiTelemetry.test.ts
│   │   │   │   └── uiTelemetry.ts
│   │   │   ├── tools
│   │   │   │   ├── diffOptions.ts
│   │   │   │   ├── edit.test.ts
│   │   │   │   ├── edit.ts
│   │   │   │   ├── glob.test.ts
│   │   │   │   ├── glob.ts
│   │   │   │   ├── grep.test.ts
│   │   │   │   ├── grep.ts
│   │   │   │   ├── ls.ts
│   │   │   │   ├── mcp-client.test.ts
│   │   │   │   ├── mcp-client.ts
│   │   │   │   ├── mcp-tool.test.ts
│   │   │   │   ├── mcp-tool.ts
│   │   │   │   ├── memoryTool.test.ts
│   │   │   │   ├── memoryTool.ts
│   │   │   │   ├── modifiable-tool.test.ts
│   │   │   │   ├── modifiable-tool.ts
│   │   │   │   ├── read-file.test.ts
│   │   │   │   ├── read-file.ts
│   │   │   │   ├── read-many-files.test.ts
│   │   │   │   ├── read-many-files.ts
│   │   │   │   ├── shell.test.ts
│   │   │   │   ├── shell.ts
│   │   │   │   ├── tool-registry.test.ts
│   │   │   │   ├── tool-registry.ts
│   │   │   │   ├── tools.ts
│   │   │   │   ├── web-fetch.test.ts
│   │   │   │   ├── web-fetch.ts
│   │   │   │   ├── web-search.ts
│   │   │   │   ├── write-file.test.ts
│   │   │   │   └── write-file.ts
│   │   │   └── utils
│   │   │       ├── LruCache.ts
│   │   │       ├── bfsFileSearch.test.ts
│   │   │       ├── bfsFileSearch.ts
│   │   │       ├── browser.ts
│   │   │       ├── editCorrector.test.ts
│   │   │       ├── editCorrector.ts
│   │   │       ├── editor.test.ts
│   │   │       ├── editor.ts
│   │   │       ├── errorReporting.test.ts
│   │   │       ├── errorReporting.ts
│   │   │       ├── errors.ts
│   │   │       ├── fetch.ts
│   │   │       ├── fileUtils.test.ts
│   │   │       ├── fileUtils.ts
│   │   │       ├── flashFallback.integration.test.ts
│   │   │       ├── formatters.ts
│   │   │       ├── generateContentResponseUtilities.test.ts
│   │   │       ├── generateContentResponseUtilities.ts
│   │   │       ├── getFolderStructure.test.ts
│   │   │       ├── getFolderStructure.ts
│   │   │       ├── gitIgnoreParser.test.ts
│   │   │       ├── gitIgnoreParser.ts
│   │   │       ├── gitUtils.ts
│   │   │       ├── memoryDiscovery.test.ts
│   │   │       ├── memoryDiscovery.ts
│   │   │       ├── memoryImportProcessor.test.ts
│   │   │       ├── memoryImportProcessor.ts
│   │   │       ├── messageInspectors.ts
│   │   │       ├── nextSpeakerChecker.test.ts
│   │   │       ├── nextSpeakerChecker.ts
│   │   │       ├── partUtils.test.ts
│   │   │       ├── partUtils.ts
│   │   │       ├── paths.ts
│   │   │       ├── quotaErrorDetection.ts
│   │   │       ├── retry.test.ts
│   │   │       ├── retry.ts
│   │   │       ├── safeJsonStringify.test.ts
│   │   │       ├── safeJsonStringify.ts
│   │   │       ├── schemaValidator.ts
│   │   │       ├── session.ts
│   │   │       ├── shell-utils.test.ts
│   │   │       ├── shell-utils.ts
│   │   │       ├── summarizer.test.ts
│   │   │       ├── summarizer.ts
│   │   │       ├── systemEncoding.test.ts
│   │   │       ├── systemEncoding.ts
│   │   │       ├── testUtils.ts
│   │   │       ├── textUtils.ts
│   │   │       ├── user_account.test.ts
│   │   │       ├── user_account.ts
│   │   │       ├── user_id.test.ts
│   │   │       └── user_id.ts
│   │   ├── test-setup.ts
│   │   ├── tsconfig.json
│   │   └── vitest.config.ts
│   └── vscode-ide-companion
│       ├── LICENSE
│       ├── README.md
│       ├── assets
│       │   └── icon.png
│       ├── esbuild.js
│       ├── eslint.config.mjs
│       ├── package-lock.json
│       ├── package.json
│       ├── src
│       │   ├── extension.ts
│       │   ├── ide-server.ts
│       │   ├── open-files-manager.test.ts
│       │   ├── open-files-manager.ts
│       │   └── utils
│       │       └── logger.ts
│       └── tsconfig.json
├── scripts
│   ├── build.js
│   ├── build_package.js
│   ├── build_sandbox.js
│   ├── build_vscode_companion.js
│   ├── check-build-status.js
│   ├── clean.js
│   ├── copy_bundle_assets.js
│   ├── copy_files.js
│   ├── create_alias.sh
│   ├── generate-git-commit-info.js
│   ├── get-release-version.js
│   ├── local_telemetry.js
│   ├── prepare-package.js
│   ├── sandbox_command.js
│   ├── start.js
│   ├── telemetry.js
│   ├── telemetry_gcp.js
│   ├── telemetry_utils.js
│   ├── tests
│   │   ├── get-release-version.test.js
│   │   ├── test-setup.ts
│   │   └── vitest.config.ts
│   └── version.js
└── tsconfig.json

49 directories, 500 files
