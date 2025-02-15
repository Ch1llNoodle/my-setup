# 任务栏图标不显示

Win+R，在打开的运行窗口中输入`%localappdata%`，回车。
勾选“查看”->“显示”->“隐藏的项目”，展示隐藏文件。
删除`IconCache.db`文件。
Ctrl+Shift+Esc，在进程中找到“Windows资源管理器”，选择“重新启动”，重建图标缓存。