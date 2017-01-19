# 部署说明

## 步骤

1. 将本库clone到本地，出现hulong.github.io.git文件夹。
2. 将该文件夹更名为blog，方便区分。
3. cd blog/进入文件夹，切换到blog分支（git checkout blog）
4. 执行npm install生成node module文件夹
5. cd blog/themes/，然后将[next](https://github.com/hulog/hexo-theme-next)clone下来。
6. 同样将文件夹更名为next。（注：原空next文件夹需先删除）
7. hexo new "title"发布新文章。
8. hexo clean -> hexo s -g -> hexo d 
