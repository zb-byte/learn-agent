# Agent Runtime Infographic Prompt

Selected template: `infographic-engine`

Useful examples: `case 334`, `case 1`, `case 8`

```text
Use case: infographic-diagram
Asset type: 16:9 presentation infographic for an engineering talk
Primary request: Create a polished Chinese technical infographic explaining "Claude Code Agent Runtime" as a sustainable agent runtime, not a simple model caller.

Subject and task:
Design one clear architecture infographic that shows how an Agent Runtime organizes user input, queryLoop, model decisions, tool execution, safety, extensions, memory, compression, and recovery into a closed loop.

Composition and layout:
- Landscape 16:9 slide, clean presentation-ready infographic.
- Use a central circular flow titled "Agent Loop / queryLoop".
- Around the central loop, place four grouped side panels:
  1. "入口与会话" with labels: "Terminal / Pipe / SDK / Resume", "CLI 模式适配", "processUserInput", "QueryEngine".
  2. "上下文与记忆" with labels: "Context Builder", "Memory", "Compression", "Prompt Cache", "Transcript / Resume".
  3. "工具与安全" with labels: "tool_use", "Tool Runtime", "Schema 校验", "Permission", "Hooks", "Sandbox", "tool_result".
  4. "扩展与可靠性" with labels: "Skill", "Plugin", "MCP", "Multi-Agent", "Fallback", "Recovery".
- The central loop must show this exact interaction path with arrows:
  "整理上下文" -> "模型请求" -> "Assistant Stream" -> "是否有 tool_use?"
  Branch A: "有 tool_use" -> "工具执行" -> "tool_result 回填" -> back to "整理上下文".
  Branch B: "无 tool_use" -> "Finalization Checks" -> "恢复 / Hook / Budget / 成功判定" -> "completed 或继续".
- Make the "无 tool_use 不等于直接完成" idea visually explicit with a small warning callout near Finalization Checks.
- Use arrows that clearly show feedback loops and not just stacked layers.

Visual style and materials:
- Premium flat engineering infographic, crisp vector-like raster rendering, clean grids, thin connector lines, subtle shadows, high legibility.
- Style should feel like a modern technical conference slide: calm, precise, high-signal.
- Use restrained multi-color grouping: blue for entry/session, teal for context/memory, amber for agent loop, red/coral for tool/safety, purple for extension/recovery.
- White or very light warm-gray background, no dark theme, no decorative blobs.
- Use small simple icons where helpful: terminal, database, shield, wrench, branching arrows, memory chip.

Text and label requirements:
- Main title text, exactly: "Claude Code Agent Runtime"
- Subtitle text, exactly: "模型负责判断下一步，运行时负责让每一步可执行、可约束、可恢复、可解释"
- Keep all labels short and readable.
- Use Chinese labels as written above; keep technical tokens exactly as `queryLoop`, `tool_use`, `tool_result`, `SDK`, `MCP`.
- Avoid long paragraphs inside the image.

Aspect ratio and output format:
- 16:9 landscape infographic, suitable for presentation slides.
- High-resolution, clean edges, readable text.

Constraints and negative details:
- Do not render it as a simple vertical stack of layers.
- Do not omit the feedback loop from `tool_result` back to context/model.
- Do not show `no tool_use` as direct completion; include Finalization Checks.
- Do not use tiny unreadable text, lorem ipsum, fake code blocks, watermarks, logos, QR codes, random brand marks, or decorative gradient orbs.
- Avoid clutter: maximum five major visual groups plus the central loop.
```

