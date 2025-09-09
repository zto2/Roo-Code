# Ask 模式执行流程模拟

## 场景：用户询问项目结构

假设用户输入的问题是："请分析这个项目的整体架构和主要组件"

### 阶段 1: 用户输入处理

```typescript
// 用户在 VSCode 中输入问题
const userInput = "请分析这个项目的整体架构和主要组件"

// VSCode 扩展接收输入
vscode.commands.executeCommand("roo-code.newTask", {
	prompt: userInput,
	mode: "ask", // 或者从当前状态获取
})
```

### 阶段 2: 任务初始化

```typescript
// ClineProvider.handleNewTask 被调用
const handleNewTask = async (params: { prompt?: string } | null | undefined) => {
	let prompt = params?.prompt

	if (!prompt) {
		prompt = await vscode.window.showInputBox({
			prompt: "请输入任务描述",
			placeHolder: "描述您想要完成的任务...",
		})
	}

	const task = await this.createTask(prompt, undefined, undefined, {
		mode: "ask", // 明确指定 ask 模式
	})

	// 创建 Task 实例并初始化模式
	await task.initializeTaskMode(this)
}
```

### 阶段 3: 系统提示生成

```typescript
// SYSTEM_PROMPT 函数生成完整的系统提示
const systemPrompt = await SYSTEM_PROMPT(
	this.context,
	this.cwd,
	this.supportsComputerUse,
	this.mcpHub,
	this.diffStrategy,
	this.browserViewportSize,
	"ask", // mode
	this.customModePrompts,
	this.customModes,
	this.globalCustomInstructions,
	this.diffEnabled,
	this.experiments,
	this.enableMcpServerCreation,
	this.language,
	this.rooIgnoreInstructions,
	this.partialReadsEnabled,
	this.settings,
	this.todoList,
	this.api.getModel().id,
)

// 生成的系统提示包含：
// - 角色定义："You are Roo, a knowledgeable technical assistant..."
// - 可用工具：read, browser, mcp
// - 规则和限制
// - 环境信息
```

### 阶段 4: API 请求构建

```typescript
// Task.initiateTaskLoop 开始执行
async function initiateTaskLoop(userContent: Anthropic.Messages.ContentBlockParam[]) {
	while (!this.abort) {
		// 构建用户消息内容
		const parsedUserContent = await processUserContentMentions({
			userContent: userContent,
			cwd: this.cwd,
			urlContentFetcher: this.urlContentFetcher,
			fileContextTracker: this.fileContextTracker,
			rooIgnoreController: this.rooIgnoreController,
			showRooIgnoredFiles: false,
			includeDiagnosticMessages: true,
			maxDiagnosticMessages: 50,
			maxReadFileLine: -1,
		})

		// 获取环境详情
		const environmentDetails = await getEnvironmentDetails(this, true)

		// 构建最终用户内容
		const finalUserContent = [...parsedUserContent, { type: "text", text: environmentDetails }]

		// 发送 API 请求
		const stream = this.attemptApiRequest()
	}
}
```

### 阶段 5: AI 模型响应处理

```typescript
// 流式接收 AI 响应
const iterator = stream[Symbol.asyncIterator]()
let item = await iterator.next()

while (!item.done) {
	const chunk = item.value

	switch (chunk.type) {
		case "text":
			// 处理文本内容
			this.assistantMessageContent = this.assistantMessageParser.processChunk(chunk.text)
			presentAssistantMessage(this)
			break

		case "tool_use":
			// 处理工具调用
			this.assistantMessageContent.push({
				type: "tool_use",
				name: chunk.name,
				params: chunk.params,
			})
			presentAssistantMessage(this)
			break
	}

	item = await iterator.next()
}
```

### 阶段 6: 工具调用处理

```typescript
// presentAssistantMessage 处理工具调用
async function presentAssistantMessage(cline: Task) {
	const block = cline.assistantMessageContent[cline.currentStreamingContentIndex]

	if (block.type === "tool_use") {
		// 验证工具权限
		const { mode, customModes } = await cline.providerRef.deref()?.getState()

		try {
			validateToolUse(
				block.name,
				mode,
				customModes,
				{
					apply_diff: cline.diffEnabled,
				},
				block.params,
			)

			// 执行相应工具
			switch (block.name) {
				case "list_files":
					await listFilesTool(cline, block, askApproval, handleError, pushToolResult, removeClosingTag)
					break

				case "read_file":
					await readFileTool(cline, block, askApproval, handleError, pushToolResult, removeClosingTag)
					break

				case "search_files":
					await searchFilesTool(cline, block, askApproval, handleError, pushToolResult, removeClosingTag)
					break
			}
		} catch (error) {
			// 工具验证失败
			pushToolResult(formatResponse.toolError(error.message))
		}
	}
}
```

### 阶段 7: 工具执行示例

```typescript
// listFilesTool 执行示例
async function listFilesTool(cline: Task, block: ToolUse, askApproval, handleError, pushToolResult, removeClosingTag) {
	const { path } = block.params

	// 请求用户批准
	const approved = await askApproval("tool", `List files in: ${path}`)

	if (approved) {
		try {
			// 执行文件列出操作
			const files = await fs.readdir(path, { withFileTypes: true })

			const result = files
				.map((file) => {
					const type = file.isDirectory() ? "[DIR]" : "[FILE]"
					return `${type} ${file.name}`
				})
				.join("\\n")

			// 返回结果
			pushToolResult(`Files in ${path}:\\n${result}`)
		} catch (error) {
			handleError("listing files", error)
		}
	}
}
```

### 阶段 8: 任务完成处理

```typescript
// attemptCompletionTool 执行
async function attemptCompletionTool(
	cline: Task,
	block: ToolUse,
	askApproval,
	handleError,
	pushToolResult,
	removeClosingTag,
) {
	const result = block.params.result

	// 检查未完成 todos
	const hasIncompleteTodos = cline.todoList && cline.todoList.some((todo) => todo.status !== "completed")

	if (hasIncompleteTodos) {
		// 如果有未完成的任务，返回错误
		cline.consecutiveMistakeCount++
		cline.recordToolError("attempt_completion")

		pushToolResult(
			formatResponse.toolError(
				"Cannot complete task while there are incomplete todos. Please finish all todos before attempting completion.",
			),
		)
		return
	}

	// 发送完成结果
	await cline.say("completion_result", result, undefined, false)

	// 触发任务完成事件
	cline.emit(RooCodeEventName.TaskCompleted, cline.taskId, cline.getTokenUsage(), cline.toolUsage)

	// 检查是否为子任务
	if (cline.parentTask) {
		// 子任务：询问用户是否批准完成
		const didApprove = await askFinishSubTaskApproval()

		if (!didApprove) {
			return
		}

		// 完成子任务并返回父任务
		await cline.providerRef.deref()?.finishSubTask(result)
		return
	}

	// 主任务：询问用户是否满意结果
	const { response, text, images } = await cline.ask("completion_result", "", false)

	if (response === "yesButtonClicked") {
		// 用户满意，任务完成
		pushToolResult("")
		return
	}

	// 用户不满意，添加反馈并继续对话
	await cline.say("user_feedback", text ?? "", images)
	const toolResults = []

	toolResults.push({
		type: "text",
		text: `The user has provided feedback on the results. Consider their input to continue the task, and then attempt completion again.\\n<feedback>\\n${text}\\n</feedback>`,
	})

	toolResults.push(...formatResponse.imageBlocks(images))
	cline.userMessageContent.push({ type: "text", text: `${toolDescription()} Result:` })
	cline.userMessageContent.push(...toolResults)

	return
}
```

## 完整流程总结

1. **用户输入** → VSCode 扩展接收
2. **任务创建** → 初始化为 ask 模式
3. **提示生成** → 构建系统提示
4. **API 请求** → 发送到 AI 模型
5. **流式响应** → 处理 AI 回复
6. **工具调用** → 验证并执行工具
7. **结果处理** → 整合工具结果
8. **任务完成** → 用户确认后结束

## 关键修正点

1. **函数签名**: 使用正确的 `handleNewTask` 函数签名
2. **变量名**: 修正 `taskInstance` 为正确的变量引用
3. **事件名称**: 使用 `RooCodeEventName.TaskCompleted` 而不是字符串
4. **任务完成逻辑**: 区分主任务和子任务的不同处理逻辑
5. **错误处理**: 添加适当的错误计数和记录

这个模拟展示了 ask 模式如何安全有效地处理技术问题，同时维护代码的安全性和用户体验。
