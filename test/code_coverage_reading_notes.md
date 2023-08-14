# 代码覆盖率（读书笔记）
## 语句覆盖（StatementCoverage）
```cpp
int foo(int a, int b){
    return a / b;
}
// TestCase: a = 10, b = 5, coverage = 100%
```
## 分支覆盖（BranchCoverage）
also known as 判定覆盖（DecisionCoverage）/(All-EdgesCoverage)/基本路径覆盖(BasicPathCoverage)/判定路径覆盖(Decision-Decision-Path)  
它度量程序中每一个判定的分支是否都被测试到了
## 条件覆盖（ConditionCoverage）
```cpp
int foo(int a, int b)
{
    if (a < 10 || b < 10) // 判定
    {
        return 0; // branch1
    }
    else
    {
        return 1; // branch2
    }
}
// BranchCoverage
// TestCase1: a = 5，b = any number 覆盖了branch1
// TestCase2: a = 15，b = 15 覆盖了branch2

// ConditionCoverage
// TestCase1: a = 5，b = 15 true, false  覆盖了branch1
// TestCase2: a = 15，b = 5 false, true  覆盖了branch1
// ConditionCoverage只保证每个表达式的true和false都测试到了就行
```
## 路径覆盖（PathCoverage）
also known as 断言覆盖（PredicateCoverage）  
是否函数的每一个分支都被执行了（需要对所有的分支排列组合）
```cpp
int foo(int a, int b)
{
    int nReturn = 0;
    if (a < 10)
    {
        nReturn += 1; // branch 1
    }
    if (b < 10)
    {
        nReturn += 10; // branch 2
    }
    return nReturn;
}
// StatementCoverage:
// TestCase a = 5, b = 5   nReturn = 11

// BranchCoverage:
// TestCase a = 5, b = 5   nReturn = 11
// TestCase a = 15, b = 15   nReturn = 0

// ConditionCoverage:
// TestCase a = 5, b = 15   nReturn = 1
// TestCase a = 15, b = 5   nReturn = 10

// PathCoverage:
// TestCase1 a = 5,    b = 5     nReturn = 0
// TestCase2 a = 15,  b = 5     nReturn = 1
// TestCase3 a = 5,    b = 15   nReturn = 10
// TestCase4 a = 15,  b = 15   nReturn = 11

// 以上所有的测试用例设计都在该覆盖标准下达到了100%
```
非原创，读书笔记  
Adapted by [CoderZh](https://blog.coderzh.com)
