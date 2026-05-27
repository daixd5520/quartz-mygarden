# 词元无限-ai算法面经

1. 项目基本没问，就让我自己讲了讲

2. 手撕 -- LC 78 数组所有子集

   回溯，模版题

   在本地写，自定义输入（我本地C++环境坏了，面的时候用的python；下来又力扣上写了一遍）

   ```C++
   class Solution {
   public:
       vector<vector<int>> res;
       vector<vector<int>> subsets(vector<int>& nums) {
           vector<int> path ={};
           dfs(nums,0,path);
           return res;
       }
       void dfs(vector<int>& nums,int start,vector<int>& path){
           res.push_back(path);//递归树访问该节点的时候存一下当前的路径
           for(int i = start;i<nums.size();i++){//同一层（字符数相等）
               path.push_back(nums[i]);
               dfs(nums,i+1,path);
               path.pop_back();
           }
       } 
   };
   ```

3. 进阶手撕--LC90 有重复元素的子集

   其实应该先排序，由于输入是排好序的就没写

   ```c++
   class Solution {
   public:
       vector<vector<int>> res;
       vector<vector<int>> subsetsWithDup(vector<int>& nums) {
           vector<int> path ={};
           dfs(nums,0,path);
           return res;
       }
       void dfs(vector<int>& nums,int start,vector<int>& path){
           res.push_back(path);
           for(int i = start;i<nums.size();i++){//同一层（字符数相等）
               //-------------------new
               //以这个字符开头的子集全被前面相同字符找过了
               if(i>start && nums[i]==nums[i-1]) continue;
               //-------------------/new
               path.push_back(nums[i]);
               dfs(nums,i+1,path);
               path.pop_back();
           }
       } 
   };
   ```

   