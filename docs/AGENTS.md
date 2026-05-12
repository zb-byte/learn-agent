<skills_system priority="1">
## Available Skills
<usage>
用户有任何需求时，请检查下面的可用技能是否有助于更有效地完成需求。
如何使用技能：
- 根据用户需求，查看 <available_skills> 中是否有合适的技能
- 可通过文件系统直接读取技能文件：优先读取 `./.haicode/skills/<skill-name>/SKILL.md`，若不存在则尝试从系统获取
使用说明:
- 优先使用 <available_skills> 中列出的技能
- 不要查询已经在你的上下文中加载的技能
- 每个技能调用都是无状态的
</usage>

<available_skills>

<skill>
<name>openspec-propose</name>
<description>在以下请求场景中请始终使用该技能：（1）涉及规划或方案类内容（如方案、规格说明、变更、计划等相关表述）；（2）介绍新增能力、破坏性变更、架构调整，或大型性能优化、安全相关工作；（3）表述含糊不清，需要在编码前参考权威规格说明。</description>
</skill>

<skill>
<name>openspec-apply-change</name>
<description>实施openspec-propose变更中的任务。当用户想要开始实施、继续实施或逐步完成任务时使用。无需外部 CLI — 所有逻辑已内置。</description>
</skill>

<skill>
<name>openspec-archive-change</name>
<description>归档openspec-apply-change已完成的变更。当用户想要在实施完成后最终归档变更时使用。无需外部 CLI — 所有逻辑已内置。</description>
</skill>

<!-- PROJECT_SKILLS_START -->
<skill>
<name>frontend-guidelines</name>
<description>用于前端项目的工程规范与开发约束</description>
<location>project</location>
</skill>
<!-- PROJECT_SKILLS_END -->

</available_skills>

</skills_system>
