先判断3 5能否整除（flag），初始化str为'',整除append对应str，flag=false则append str(num)
```python
class Solution:
    def fizzBuzz(self, n: int) -> List[str]:
        res = []
        for i in range(1,n+1):
            s = ''
            null_flag = True
            if i % 3 == 0:
                s = s+'Fizz'
                null_flag = False
            if i % 5 == 0:
                s = s+'Buzz'
                null_flag = False
            if  null_flag:
                s = s+str(i)
            res.append(s)
        return res

```
