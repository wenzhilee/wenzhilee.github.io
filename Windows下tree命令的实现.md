# Windows下tree命令的实现

## Problem
最近，使用git-for-windows的时候想使用tree命令，但是发现git bush无法直接使用Windows内置的tree命令。其实是输入`tree.com`才可以使用，不过，如果不改变gitbush的字符编码(`utf-8`)为系统默认编码(`gbk`)，会出现乱码。
如图所示：

![git bush下没有tree命令](http://imglf1.nosdn.127.net/img/U1hJTFBDakFzRmtJZGdyK09QdWtKNFJsd1lpWHpkZ3N6NHAwb2I4MnBVV1VWWCtNbmJBdUxBPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

![tree.com在gitbush下乱码](http://imglf2.nosdn.127.net/img/U1hJTFBDakFzRmtJZGdyK09QdWtKK3FiWmRkVmdzTy9KLytHSVNyOGQrZXNGVDRRalZLREtBPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

## Solution
既然git bush下没有tree命令，我们就自己实现一个吧！

### 1. 目录树的构建
现在假设我们有如下目录：
```
|__bin
|  |__Debug
|     |__tree.exe
|__main.c
|__obj
   |__Debug
   |  |__main.o
   |__folder1
   |  |__file1.log
   |  |__file2.log
   |__folder2
      |__file3.log
```
如果我们把它画成树形结构，如图所示：

![目录树](http://imglf0.nosdn.127.net/img/U1hJTFBDakFzRmtCei9rcGlHQ2RKUmxmeFFLZWVpQy9oZUhKSlFJRlB6M2wrSC9IMEtScmZRPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

为了更好的处理这个树，我们将上述目录树转化为二叉树。

转化成二叉树的步骤如下：

**1. 所有兄弟节点（即同一级）的节点连成一线；**

**2. 对树中的每个结点，只保留它与第一个孩子结点之间的连线，删除它与其它孩子结点之间的连线；**

**3. 调整并整理树的位置，让它好看点。**

例如：对于根节点 `./`， 将 `bin/`，`main.c`， `obj/` 用线连起来，之后删除`./`和`main.c`， `obj/`之间的连线，对于其它子节点使用同样的方法可得如图所示的二叉树：

![转化为二叉树的目录树](http://imglf2.nosdn.127.net/img/U1hJTFBDakFzRmtCei9rcGlHQ2RKUy9UdVhoVU1tQUttcVdFbU5hb3pucE51Ukt1Q3dUTEVBPT0.png?imageView&thumbnail=500x0&quality=96&stripmeta=0&type=jpg)

**在这里，我们把文件夹和文件（也就是不是文件夹的文件）都看作是文件，我们的左子树是当前文件（如果是文件夹）下的子文件，右子树是当前文件的兄弟节点。**

保存一个二叉树的节点，我们需要一个如下的数据结构：
```C
typedef DWORD atrr;
struct tree_node_ {
    char name[MAX_PATH];//文件名
    attr type;//文件属性，指明是文件夹还是其他文件
    struct tree_node_ * parent;//父节点，不使用
    struct tree_node_ * sibling;//兄弟节点
    struct tree_node_ * child;//子节点
}
typedef tree_node_ tree_node;
```

首先，我们需要一个初始化一个节点的函数，以防止每次都写一大堆重复代码：
```C
//初始化一个节点
tree_node* tree_init_a_node(WIN32_FIND_DATA* data)
{
	tree_node* node = NULL;

	node = (tree_node*)malloc(sizeof(tree_node));

	if (node == NULL)
	{
		printf("内存分配失败\n");
		return NULL;
	}
	//将指针域置为空
	node->parent = node->sibling = node->child = NULL;
    //根据data，设置节点属性
	//文件名
	memset(node->name, 0, MAX_PATH);
	memmove(node->name, data->cFileName, strlen(data->cFileName));
	//文件类型
	node->type = data->dwFileAttributes;

	return node;
}
```

接下来，我们来定义一个插入二叉树节点的函数：

```C
//插入一个节点，这里使用了指针的指针，如果不使用**,容易出错，画个图就知道了。
void tree_insert_node(tree_node** node, WIN32_FIND_DATA* data)
{
	if (*node != NULL)
	{
        //这里我们只需要考虑兄弟节点，子节点child会在其他地方就行判断并插入
		tree_insert_node(&((*node)->sibling), data);
	}
	else
	{
        //将初始化的节点出入到二叉树上
		*node = tree_init_a_node(data);
	}
}
```

将初始化的节点出入到二叉树上
`*node = tree_init_a_node(data);`


### 2. 搜索文件
既然是在Windows下，我们使用Windows API来查找当前目录下的每一个文件。先介绍下面的几个函数：
```C
//查找目录下的第一个文件
HANDLEWINAPI FindFirstFile(
_In_ LPCTSTRlpFileName, // 待查目录
_Out_ LPWIN32_FIND_DATA lpFindFileData // 找到的文件(输出参数)
);
//获取当前目录
DWORD WINAPI GetCurrentDirectory(
  _In_   DWORD nBufferLength,   // 保存目录的变量大小
  _Out_  LPTSTR lpBuffer        // 保存目录的变量
);
//获取下一个文件
BOOL WINAPI FindNextFile(
  _In_   HANDLE hFindFile,      // 相对文件句柄
  _Out_  LPWIN32_FIND_DATA lpFindFileData   // 获取文件信息(输出参数)
);
```
由于我喜欢用小写字母加下划线的方式编程，所以我们将上述函数封装一下：

```C
HANDLE find_first_file(char* filename, WIN32_FIND_DATA* findfiledata)
{
    return FindFirstFile(filename, findfiledata);
}

DWORD get_current_dir(char* buff, int bufflen)
{
    return GetCurrentDirectory(bufflen, buff);
}

BOOL find_next_file(HANDLE h_find_file, WIN32_FIND_DATA* findfiledata)
{
    return FindNextFile(h_find_file, findfiledata);
}
```

现在我们来搜索当前目录及其子目录下的所有文件：
```C
tree_node* tree_search(tree_node** node, const char* current_dir)
{
	tree_node** temp = NULL;
	char wildcard[MAX_PATH] = { 0 };//搜索文件时使用的通配符
	char sub_dir[MAX_PATH] = { 0 };//如果文件是一个文件夹，我们用当前目录和该文件夹组成路径进行递归搜索
	HANDLE hfile;//当前文件句柄
	WIN32_FIND_DATA file_data;//保存找到文件的信息的结构体
    //构成文件通配符
	sprintf(wildcard, "%s\\*", current_dir);
    //搜索文件文件，找到第一个文件，
	hfile = FindFirstFile(wildcard, &file_data);
    //无效句柄，搜索失败或没有子文件
	if (hfile == INVALID_HANDLE_VALUE)
	{
		printf("搜索文件失败%ld\n", GetLastError());
		return NULL;
	}
    //如果找到第一个文件
	else
	{
        //找到当前目录下的所有文件
		do {
			//过滤 "." 和 ".."
			if (!(strcmp(file_data.cFileName, ".")) || !(strcmp(file_data.cFileName, "..")))
			{
				continue;
			}
            //插入一个节点
			tree_insert_node(node, &file_data);
            //如果是一个文件夹，我们就进入这个文件夹，递归调用tree_search
			if (file_data.dwFileAttributes == FILE_ATTRIBUTE_DIRECTORY)
			{
				sprintf(sub_dir, "%s\\%s", current_dir, file_data.cFileName);
				temp = node;
				while ((*temp)->sibling != NULL)
				{
					temp = &(*temp)->sibling;
				}
				tree_search(&((*temp)->child), sub_dir);
			}
			temp = NULL;
		} while (FindNextFile(hfile, &file_data));
	}
	return *node;
}
```
需要说明的是，`.`和`..`代表当前目录和当前目录的父目录，所以我们要过滤掉这两个文件。还有就是如果一个文件是文件夹，我们需要进入这个文件夹，要做到这一点，只要构建好路径，进行递归调用就行。**注意，传入的节点应该是child节点`&((*temp)->child)`，因为接下来找到的文件就这个文件夹的左子树。见二叉树的图。**

## 3. 打印二叉树
因为我们的二叉树中左节点是一个文件（如果是文件夹才有）下的子文件，右节点是同级文件，所以我们只要使用前序遍历就可以进行打印二叉树了。仍然是用我们的递归，代码如下：
```C
/*
node:树的根节点
blk:缩进次数
sharp:节点间连线的形状
*/
void tree_print(const tree_node* node, int blk, char* sharp)
{
	int i;
	if (node != NULL)
	{
		for (i = 0; i < blk; i++)
		{
			printf("   ");
		}
        //先序遍历
        //打印根节点
		printf("%s%s\n", sharp, node->name);
        //打印左子树
		tree_print(node->child, blk + 1, sharp);
        //打印右子树
		tree_print(node->sibling, blk, sharp);
	}
}
```
按照先序遍历的步骤打印二叉树即可。在`tree_print`中，我们用`blk`指明缩进次数，`sharp`指明节点间连线的形状。需要注意的是，如果一个节点有子节点，在打印其子节点时，我们需要增加一次缩进`blk + 1`，而对于兄弟节点，则不需要。

最后，我们来释放为二叉树分配的内存：
```C
void tree_free(tree_node* node)
{
	if (node != NULL)
	{
        //先释放当前节点的sibling节点和它的child节点
		tree_free(node->sibling);
		tree_free(node->child);
        //最后在释放当前节点
		free(node);
	}
}
```
释放节点的时候要注意，先释放当前节点的sibling节点和它的child节点，最后在释放当前节点。如果先释放当前节点，会使当前节点为空，无法找到其子节点和兄弟节点。

## 附完整代码

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <windows.h>

typedef DWORD attr;

typedef struct tree_node_ {
	char name[MAX_PATH];
	attr type;
	struct tree_node_* parent;
	struct tree_node_* sibling;
	struct tree_node_* child;
}tree_node;

DWORD get_current_dir(char* buff, int bufflen);
tree_node* tree_init_a_node(WIN32_FIND_DATA* data);
void tree_insert_node(tree_node** node, WIN32_FIND_DATA* data);
tree_node* tree_search(tree_node** node, const char* current_dir);
void tree_print(const tree_node* node, int blk, char* sharp);
void tree_free(tree_node* node);


int main(int agrc, char* argv[])
{
	tree_node* root = NULL;
	char current_dir[MAX_PATH] = { 0 };

	root = (tree_node*)malloc(sizeof(tree_node));
	root->sibling = root->child = NULL;
	memset(root->name, 0, MAX_PATH);
	get_current_dir(current_dir, MAX_PATH);
	memmove(root->name, current_dir, strlen(current_dir));
	root->type = FILE_ATTRIBUTE_DIRECTORY;

	tree_search(&root, current_dir);
	tree_print(root, 0, "|__");
	system("PAUSE");
	return 0;
}

DWORD get_current_dir(char* buff, int bufflen)
{
	return GetCurrentDirectory(bufflen, buff);
}

tree_node* tree_init_a_node(WIN32_FIND_DATA* data)
{
	tree_node* node = NULL;

	node = (tree_node*)malloc(sizeof(tree_node));

	if (node == NULL)
	{
		printf("内存分配失败\n");
		return NULL;
	}
	//指针指向空
	node->parent = node->sibling = node->child = NULL;
	//name属性
	memset(node->name, 0, MAX_PATH);
	memmove(node->name, data->cFileName, strlen(data->cFileName));
	//type属性
	node->type = data->dwFileAttributes;

	return node;
}

//插入一个节点，这里使用了指针的指针，如果不使用**,容易出错，画个图就知道了。
void tree_insert_node(tree_node** node, WIN32_FIND_DATA* data)
{
	if (*node != NULL)
	{
        //这里我们只需要考虑兄弟节点，子节点child会在其他地方就行判断并插入
		tree_insert_node(&((*node)->sibling), data);
	}
	else
	{
        //将初始化的节点出入到二叉树上
		*node = tree_init_a_node(data);
	}
}

tree_node* tree_search(tree_node** node, const char* current_dir)
{
	tree_node** temp = NULL;
	char wildcard[MAX_PATH] = { 0 };//搜索文件时使用的通配符
	char sub_dir[MAX_PATH] = { 0 };//如果文件是一个文件夹，我们用当前目录和该文件夹组成路径进行递归搜索
	HANDLE hfile;//当前文件句柄
	WIN32_FIND_DATA file_data;//保存找到文件的信息的结构体
    //构成文件通配符
	sprintf(wildcard, "%s\\*", current_dir);
    //搜索文件文件，找到第一个文件，
	hfile = FindFirstFile(wildcard, &file_data);
    //无效句柄，搜索失败或没有子文件
	if (hfile == INVALID_HANDLE_VALUE)
	{
		printf("搜索文件失败%ld\n", GetLastError());
		return NULL;
	}
    //如果找到第一个文件
	else
	{
        //找到当前目录下的所有文件
		do {
			//过滤 "." 和 ".."
			if (!(strcmp(file_data.cFileName, ".")) || !(strcmp(file_data.cFileName, "..")))
			{
				continue;
			}
            //插入一个节点
			tree_insert_node(node, &file_data);
            //如果是一个文件夹，我们就进入这个文件夹，递归调用tree_search
			if (file_data.dwFileAttributes == FILE_ATTRIBUTE_DIRECTORY)
			{
				sprintf(sub_dir, "%s\\%s", current_dir, file_data.cFileName);
				temp = node;
				while ((*temp)->sibling != NULL)
				{
					temp = &(*temp)->sibling;
				}
				tree_search(&((*temp)->child), sub_dir);
			}
			temp = NULL;
		} while (FindNextFile(hfile, &file_data));
	}
	return *node;
}

/*
node:树的根节点
blk:缩进次数
sharp:节点间连线的形状
*/
void tree_print(const tree_node* node, int blk, char* sharp)
{
	int i;
	if (node != NULL)
	{
		for (i = 0; i < blk; i++)
		{
			printf("   ");
		}
        //先序遍历
        //打印根节点
		printf("%s%s\n", sharp, node->name);
        //打印左子树
		tree_print(node->child, blk + 1, sharp);
        //打印右子树
		tree_print(node->sibling, blk, sharp);
	}
}

void tree_free(tree_node* node)
{
	if (node != NULL)
	{
        //先释放当前节点的sibling节点和它的child节点
		tree_free(node->sibling);
		tree_free(node->child);
        //最后在释放当前节点，否则无法找到其子节点和兄弟节点
		free(node);
	}
}
```
