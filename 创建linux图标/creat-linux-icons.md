# 一、Linux下桌面图标位置
Linux下桌面图标储存路径为： /usr/share/applications

# 二、桌面图标格式
所有桌面图标格式为desktop，例 xxx.desktop

# 三、编辑内容
// 文件头（必须）

[Desktop Entry]
 
// 编码方式（可选）

Encoding=UTF-8
 
//程序名（必须）

Name = XXX
 
//图标（可选）

Icon=图标文件名（全称包含路径）
 
//执行脚本（若为应用程序桌面图标则必选），与在terminal执行命令一致，Type为Application时，Exec才有效

Exec=脚本文件路径
 
// 分类

Type=Application
 
// 鼠标经过上面时的提示名称

Comment=comment 