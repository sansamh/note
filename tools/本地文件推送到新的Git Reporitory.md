1. **先在Git上创建新的项目**

2. **初始化本地项目**

   ```properties
   加入git提交忽略的文件.gitignore文件
   
   .idea 忽略以.idea文件
   logs/ 忽略logs文件夹
   *.iml 忽略以iml结尾的文件
   target/ 忽略target文件夹
   ```

   

3. **进入文件夹主目录 将文件夹加入git管理 - 执行 git init**

4. **将项目添加到git本地仓库 - 执行 git add .**

5. **提交文件到本地仓库 - git commit -m "提交的备注"**

6. **连接远程仓库 - git remote add origin https://github.com/xxx/xxx.git**

7. **先从远程仓库更新 - git pull orgin master**

8. **本地内容推送到远程仓库 - git push -u origin master (-f强制推送)** 

9. 执行完后，如果没有异常，等待执行完就上传成功了，中间可能会让你输入Username和Password，你只要输入github的账号和密码就行了

   

   ```shell
   # 解决无法pull分支，强制merge
   fatal: refusing to merge unrelated histories 
   git pull origin master --allow-unrelated-histories
   
   # 解决无法推送
   error: failed to push some refs to
   git pull --rebase origin master
   
   git rebase --skip
    
   ```

   

   