# 分类学习代码

## string

### 1 string的全排列（有无重复）

我们求整个字符串的排列，可以看成两步，首先求所有可能出现在第一个位置的字符，即把第一个字符和后面所有的字符交换，第二部固定第一个字符，求后面所有字符的排列

这个时候，我们仍然把后面所有的字符分成两部分，后面字符的第一个字符，以及这个字符之后的所有字符

~~~ C++
#include<iostream>
using namespace std;
#include<assert.h>
 
void Permutation(char* pStr, char* pBegin)
{
	assert(pStr && pBegin);
 
	if(*pBegin == '\0')
		printf("%s\n",pStr);
	else
	{
		for(char* pCh = pBegin; *pCh != '\0'; pCh++)
		{
			swap(*pBegin,*pCh);
			Permutation(pStr, pBegin+1);
			swap(*pBegin,*pCh);
		}
	}
}
 
int main(void)
{
	char str[] = "abc";
	Permutation(str,str);
	return 0;
}
--------------------- 
版权声明：本文为CSDN博主「hackbuteer1」的原创文章，遵循CC 4.0 by-sa版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/hackbuteer1/article/details/7462447
~~~



**如果string中有重复的字符怎么办？**

换种思维，对122，第一个数1与第二个数2交换得到212，然后考虑第一个数1与第三个数2交换，此时由于第三个数等于第二个数，所以第一个数不再与第三个数交换，全排列中去掉重复的规则——去重的全排列就是从第一个数字起每个数分别与它后面非重复出现的数字交换。

~~~ C++

#include<iostream>
using namespace std;
#include<assert.h>
 
//在[nBegin,nEnd)区间中是否有字符与下标为pEnd的字符相等
bool IsSwap(char* pBegin , char* pEnd)
{
	char *p;
	for(p = pBegin ; p < pEnd ; p++)
	{
		if(*p == *pEnd)
			return false;
	}
	return true;
}
void Permutation(char* pStr , char *pBegin)
{
	assert(pStr);
 
	if(*pBegin == '\0')
	{
		static int num = 1;  //局部静态变量，用来统计全排列的个数
		printf("第%d个排列\t%s\n",num++,pStr);
	}
	else
	{i
		for(char *pCh = pBegin; *pCh != '\0'; pCh++)   //第pBegin个数分别与它后面的数字交换就能得到新的排列   
		{
			if(IsSwap(pBegin , pCh))
			{
				swap(*pBegin , *pCh);
				Permutation(pStr , pBegin + 1);
				swap(*pBegin , *pCh);
			}
		}
	}
}
 
int main(void)
{
	char str[] = "baa";
	Permutation(str , str);
	return 0;
}

原文链接：https://blog.csdn.net/hackbuteer1/article/details/7462447


vector的话代码如下：
class Solution {
public:
    vector<vector<int>> result;
    vector<vector<int> > permuteUnique(vector<int> &num) {
        //sort(num.begin(),num.end());
        //这样的if判断语句，和引用传参，无需传参也是ok哒
        int len = num.size();
        recursion(num,0,len);
        return result;
    }
    
    void recursion(vector<int>& num,int i,int len){
        if(i==len-1){
            result.push_back(num);
        }else{
            for(int k=i;k<len;++k){
                //如果从i开始的后两个相邻的元素相同，则不需要交换并进行后续的递归调用
                //避免产生重复的元组
                //if(i!=k && num[i]==num[k] || (i!=k && num[k]==num[k-1]))
                    //continue;
                if(isSwap(i,k,num)){
                    swap(num[i],num[k]);
                    recursion(num,i+1,len);
                    swap(num[i],num[k]);
                }
                
                
            }
        }
    }
    
    bool isSwap(int begin,int end,vector<int> num){
        for(int i=begin;i<end;++i){
            if(num[i]==num[end])
                return false;
        }
        return true;
    }
};
~~~



**如果不是以引用或者指针的方式进行传递，则代码改动如下：**

> note:这道题的精华在于sort，也就是说要求，进行递归的数组都是有序的才可以
>
> 那么此时传递改为引用的话，就有可能将后续递归的数组变为无序的，这样就会出错
>
> 示例如下：
>
> 1234
>
> 在同一层swap后是2134，那么2134传入后面的递归也是2134中的134
>
> 在同一层swap后是3124，那么3124传入后面的递归也是3124中的124
>
> 在同一层swap后是4123，那么4123传入后面的递归也是4123中的123
>
> 这样相当于是循环字符串，那么一定可以保证参与递归的是有序的数组
>
> 嗯！理解非常nice了

~~~ C++
class Solution {
public:
    vector<vector<int>> result;
    vector<vector<int> > permuteUnique(vector<int> &num) {
        sort(num.begin(),num.end());
        int len = num.size();
        recursion(num,0,len);
        return result;
    }
    
    void recursion(vector<int>& num,int i,int len){
        if(i==len-1){
            result.push_back(num);
        }else{
            for(int k=i;k<len;++k){
                //如果从i开始的后两个相邻的元素相同，则不需要交换并进行后续的递归调用
                //避免产生重复的元组
                if(i!=k && num[i]==num[k])
                    continue;
                swap(num[i],num[k]);
                recursion(num,i+1,len);
            }
        }
    }
};
~~~



### 2 string的全组合（有无重复）

**解题思路：**

可以考虑求长度为n的字符串中m个字符的组合，设为C(n,m)。原问题的解即为C(n, 1), C(n, 2),...C(n, n)的总和。对于求C(n, m)，从第一个字符开始扫描，每个字符有两种情况，要么被选中，要么不被选中，如果被选中，递归求解C(n-1, m-1)。如果未被选中，递归求解C(n-1, m)。不管哪种方式，n的值都会减少，递归的终止条件n=0或m=0。

~~~ C++
#include <iostream>
#include <cstring>
#include <vector>

using namespace std;

//从一个字符串中选 m 个元素 
void Combination_m(char* str, int m, vector<char> &result) 
{
    //字符串为空，或者长度达不到 m
    if (str == NULL || (*str == '\0' && m != 0))
        return;
    
    //出口，输出组合
    if(m == 0)
    {
        static int count = 0;
        cout << ++count << ":";
        for (int i = 0; i < result.size(); i++)
            cout << result[i];
        cout << endl;
        return;
    } 
    
    result.push_back(*str);
    Combination_m(str+1, m-1, result);
    result.pop_back();
    Combination_m(str+1, m, result);
}

//求一个字符串的组合
void Combination(char* str)
{
    if (str == NULL || *str == '\0')
        return;
    int num = strlen(str);
    for (int i = 1; i <= num; i++)
    {
        vector<char> result;
        Combination_m(str, i, result);
    }
} 

int main()
{
    char str[20];
    cin >> str;
    Combination(str);
    return 0;
}
~~~

**分析：**在Combination_m()函数中加入判断，判断当前组合是否已经存在（即打印）。

reference：https://www.jianshu.com/p/8e9ef061a79b



### 3 大数相加（不含负数）



