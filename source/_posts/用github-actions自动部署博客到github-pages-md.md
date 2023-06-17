创建.github/workflows/pages.yml

```yml

name: Deploy github pages  # 工作流程的名称

on:  # 触发工作流程的事件
  push:  # 当代码推送到远程仓库时触发
    branches:  # 限定分支
      - source

jobs:  # 工作流程中的作业
  deploy:  # 作业名称
    runs-on: ubuntu-latest  # 作业运行的操作系统环境
    permissions:  # 作业权限设置
      contents: write  # 允许写入操作

    steps:  # 作业中的步骤
      - uses: actions/checkout@v2  # 使用 checkout 操作，从远程仓库检出代码
      - name: Setup Node.js  # 步骤名称
        uses: actions/setup-node@v1  # 使用 setup-node 操作，设置 Node.js 环境
        with:
          node-version: 14.17.0  # 设置 Node.js 版本为 14.17.0
          npm-version: 6.14.13  # 设置 npm 版本为 6.14.13
      - name: Cache node modules  # 步骤名称
        uses: actions/cache@v2  # 使用 cache 操作，缓存 node_modules 目录
        with:
          path: node_modules  # 缓存的目录路径
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}  # 缓存的键
          restore-keys: |
            ${{ runner.os }}-node-  # 恢复缓存的键
      - name: Install dependencies  # 步骤名称
        run: npm install  # 运行命令，安装依赖
      - name: Build  # 步骤名称
        run: npm run build  # 运行命令，构建项目
      - name: Deploy  # 步骤名称
        uses: peaceiris/actions-gh-pages@v3  # 使用 peaceiris/actions-gh-pages 操作，部署到 GitHub Pages
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}  # GitHub Token，用于进行部署的身份验证 自动生成 无需手动创建
          publish_dir: ./public  # 要发布到 GitHub Pages 的目录


```

这个工作流会把 hexo 生成的 public 目录下的文件 push 到 gh-pages 分支
actions 运行结果如下图

![image-20230617192254695](https://raw.githubusercontent.com/johnson329/picture_bed/master/image-20230617192254695.png)
