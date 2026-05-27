# Leetcode记录

## Map

### 新建

`Map<Integer,Integer> m=new HashMap<>()`

### 插入

`m.put(key,v)`

### 查找

`if(m.containsKey(key))`

## **层序遍历二叉树(记录高度)**

```c++
deque <TreeNode*> dq;
dq.push_back(root);
intr h = 0;
//特判root是null，不写了
while(!dq.empty()){
    int layersize = dq.size();//如果无需size的话，直接写循环内部的代码
    for(int i = 0;i<layersize;++i){
        cur = dq.front();
        dq.pop_front();
        /*同时跟着pop进行对当前节点的操作*/
        //todo xxx：比如交换左右节点
        
        if(cur->left!=nullptr) dq.push_back(cur->left);
        if(cur->right!=nullptr) dq.push_back(cur->right);
        
    }
    ++h;//遍历完一层计数
}
return h;
```

## 对称二叉树

需要注意递归的时候不仅ab的左右节点需要判，ab本身的val也要相等

迭代法是一次判断一对（ab）；一对入4个（a左b右a右b左）

```c++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    bool isSymmetric(TreeNode* root) {
        // if(root == nullptr) return true;
        // return ismirror(root->left,root->right);
        if(root==nullptr) return true;
        deque<TreeNode*> dq;
        dq.push_back(root->left);
        dq.push_back(root->right);
        while(!dq.empty()){
            TreeNode* a = dq.front();
            dq.pop_front();
            TreeNode* b = dq.front();
            dq.pop_front();

            if(a==nullptr && b==nullptr) continue;
            if(a==nullptr || b==nullptr) return false;
            if(a->val!=b->val) return false;
            dq.push_back(a->left);
            dq.push_back(b->right);
            dq.push_back(a->right);
            dq.push_back(b->left);
        }
        return true;

    }
    // bool ismirror(TreeNode* a,TreeNode* b){
    //     if(a!=nullptr&&b==nullptr) return false;
    //     if(a==nullptr&&b==nullptr) return true;
    //     if(a==nullptr&&b!=nullptr) return false;
    //     return (ismirror(a->right,b->left) &&ismirror(a->left,b->right) && a->val==b->val);
    // }
};
```

## 二叉树展开为链表

```c++
class Solution {
public:
    void flatten(TreeNode* root) {
        //两个指针，pre,cur
        //pre用来找cur的左子树的最右，找到后指向cur的right
        //然后cur断开原本right,cur的right更新为cur的left
        //然后cur更新下一个（cur->right）
        TreeNode* cur = root;
        TreeNode* pre;
        while(cur!=nullptr){
            if(cur->left!=nullptr){//判左子树
                pre = cur->left;//开始找左子树最右
                while(pre->right!=nullptr){
                    pre = pre->right;
                }
                if(pre->right!=cur){pre->right = cur->right;}
                cur->right = cur->left;//cur解绑
                cur->left = nullptr;
            }
            cur = cur->right;
        }
    }
};
```

## 从前序中序建立二叉树

```c++
class Solution {
public:
    map<int,int> hm;
    int preind = 0;
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        int size = inorder.size();
        for(int i = 0;i<size;++i){//建中序哈希表
            // hm.insert(inorder[i],i);//没这个用法
            hm[inorder[i]] = i;
        }
        return build(preorder,0,size);
    }
    TreeNode* build(vector<int>& pre,int l,int r){
        if(l==r) return nullptr;//递归出口

        int curval = pre[preind++];
        TreeNode* cur = new TreeNode(curval);
        
        /*进行查哈希表；其实不用这么麻烦*/
        // auto iter = hm.find(curval);
        // int ind = iter->second;

        int ind = hm[curval];

        cur->left = build(pre,l,ind);
        cur->right = build(pre,ind+1,r);
        return cur;
    }
};
```

## LCA

```c++
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        //递归。如果当前节点左右分别有p,q，则当前节点就是p和q的LCA
        //如果一个查到了，另一个没查到，就哪边有战果汇报哪边；向上汇报
        if(root==nullptr || root==p || root==q){
            return root;
        }
        TreeNode* left = lowestCommonAncestor(root->left,p,q);
        TreeNode* right = lowestCommonAncestor(root->right,p,q);
        if(left!=nullptr&&right!=nullptr) return root;
        if(left!=nullptr) return left;///哪边有战果汇报哪边
        if(right!=nullptr) return right;
    }
};
```



