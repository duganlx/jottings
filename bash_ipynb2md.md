# 将 .ipynb 文件转换成对应的 .md 文件


```bash
if ! which jupyter > /dev/null; then
  echo "未找到 jupyter 命令，请运行命令 pip install jupyter -i https://pypi.tuna.tsinghua.edu.cn/simple"
  exit 1
fi

read -p "是否直接扫描本脚本所在目录，请输入(y/n): " choice

case $choice in
  y)
    path=$(dirname "$0")
    ;;
  n)
    read -p "请输入目录的绝对路径：" path
    ;;
  *)
    echo "输入无效"
    exit 1
    ;;
esac

if [ ! -d "$path" ]; then
  echo "路径不存在或不是一个目录"
  exit 1
fi

ipynb_files=$(ls "$path"/*.ipynb 2>/dev/null)

if [ -z "$ipynb_files" ]; then
  echo "在路径（$path）中不存在ipynb文件"
  exit 0
fi

for file in $ipynb_files; do
  jupyter nbconvert --to markdown $file
  echo "已生成文件（$file）对应的md文件"
done

echo "md生成完毕 ^_^"
```