#  目录结构

```dataviewjs
// 获取所有文件
const files = dv.pages().file;

// 创建一个函数来生成树状结构
function createTreeStructure(files) {
    const root = {};

    files.forEach(file => {
        const parts = file.path.split('/');
        let current = root;

        parts.forEach((part, index) => {
            if (!current[part]) {
                current[part] = index === parts.length - 1 ? null : {};
            }
            current = current[part];
        });
    });

    return root;
}

// 递归函数来构建树状结构字符串
function buildTreeString(node, indent = '', isLast = true) {
    let result = '';
    const keys = Object.keys(node);
    keys.forEach((key, index) => {
        const isLastChild = index === keys.length - 1;
        const connector = isLastChild ? '└── ' : '├── ';
        result += indent + connector + key + '\n';
        if (node[key] !== null) {
            result += buildTreeString(node[key], indent + (isLastChild ? '    ' : '│   '), isLastChild);
        }
    });
    return result;
}

// 创建树状结构字符串
const tree = createTreeStructure(files);
const treeString = buildTreeString(tree);

// 输出树状结构字符串在markdown代码块中
dv.paragraph('```\n' + treeString + '```');
```
