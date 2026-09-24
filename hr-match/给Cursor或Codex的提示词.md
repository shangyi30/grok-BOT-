请严格同目录文件《HR人岗评价系统-Cursor完整施工方案.md》+《UX说明.md》落地实现整站。

要求：
1. 不要改技术栈；不要做成文档 RAG；不要微调
2. 按方案第 10 节分步实施；每步结束列出改动文件；自测再继续
3. 页面交互以《UX说明.md》为准（优先：导入映射、近邻对照、跑批并排）；路由与契约以施工方案为准
4. 先用 MOCK_LLM 跑通主路径，再接真 LLM
5. 最终按 Compose + Caddy/HTTPS + 登录闭环，并按第 14 节 Definition of Done 自检

开始前先搭好 monorepo 骨架，再按 Prisma → Auth → Cases → 近邻 → Inference → Eval → 前端九页 → 部署 推进。
