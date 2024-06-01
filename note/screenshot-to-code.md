---
tags:
  - 开源项目
date: 2024-05-31
---
# 开源地址
* [screenshot-to-code](https://github.com/abi/screenshot-to-code)
# 开源构建前端项目
+ 镜像构建容器
	+  `docker run --name screenshot-to-code-backend -d --network host -e OPENAI_API_KEY="你的key" screenshot-to-code-backend:latest poetry run uvicorn main:app --host 0.0.0.0 --port 7001 
	`
	+ ` docker run -p 5173:5173 --name screenshot-to-code-frontend -d \-e VITE_WS_BACKEND_URL="ws://127001:7001" \screenshot-to-code-frontend:latest
	`
# 参考资料
* [Docker入门：使用Dockerfile构建Docker镜像-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2259804)