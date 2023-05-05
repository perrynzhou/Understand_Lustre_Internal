# Understand_Lustre_Internal


###  协作方式

- task的分配按照issue的方式，task按照"component->feature"的维度细分
- `git checkout -b {your_branch}`,基于自己的分支来完成task，完成task后可以merge到master分支
- 项目的源代码组织形式如下


### 项目组织形式
```
[perrynzhou@perrynzoudeMBP2 ~/Source/Understand_Lustre_Internal]$ tree ./
./
|-- README.md  --说明
|-- internal   --任务完成后的task按照组件和功能放置到internal/下面
|   |-- client
|   |   `-- 1.test
|   |-- mds
|   |   `-- 1.test
|   |-- mgs
|   |   `-- 1.test
|   `-- oss
|       `-- 1.test
|-- perryn  -- 团队成员的自己完成task过程记录
|   `-- 1.test
`-- public -- 这里数据按照book方式组织大家的文章
    `-- 1.test
```