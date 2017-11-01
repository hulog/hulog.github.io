# 部署说明

## 步骤

1. 上node[官网](https://nodejs.org/en/)下载最新node安装包
2. 配置环境:
 ```shell
 sudo vim /etc/profile
 # set node env
 export NODE_HOME=/home/norman/software/node-v8.9.0-linux-x64
 export PATH=$NODE_HOME/bin:$PATH
 
 source /etc/profile
 node -v
 ```

3. 将本库`clone`到本地，出现**hulong.github.io.git**文件夹。
4. 将该文件夹更名为**blog**，方便区分。
5. `cd blog/` 进入文件夹，切换到`blog`分支（`git checkout blog`）
6. 执行`npm install` 生成**node module**文件夹
7. `hexo new "title"`发布新文章。
8. `hexo clean` -> `hexo s -g` -> `hexo d` 
