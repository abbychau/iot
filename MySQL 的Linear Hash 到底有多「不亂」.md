---  
title: MySQL 的Linear Hash 到底有多「不亂」  
updated: 2022-08-17 12:14:02Z  
created: 2022-08-17 12:12:24Z  
tags:  
  - database  
share: true  
---  
  
在MySQL  中，如果資料很多，為了同一個索引的參數可以高速存取，我們都會分區，而分區又有很多的方法。  
  
其中要平均散序時，我們有Hash 和Linear Hash 兩種。  
  
這兩種有甚麼分別呢？  
在我尋找過一陣子之後，好像沒有人解釋Linear Hash 的原理和實際的Trade off。幾乎全部人都滿足於官方文檔中的這個描述。  
  
> The advantage in partitioning by linear hash is that the adding, dropping, merging, and splitting of partitions is made much faster, which can be beneficial when dealing with tables containing extremely large amounts (terabytes) of data. The disadvantage is that data is less likely to be evenly distributed between partitions as compared with the distribution obtained using regular hash partitioning.  
  
中文:  
> 通過線性哈希進行分區的優勢在於，分區的添加，刪除，合併和拆分變得更快，這在處理包含大量數據(兆兆字節)的 table 時可能是有益的。缺點是，與使用常規哈希分區獲得的分佈相比，數據可能在分區之間不太均勻分佈。  
  
算法和說明：https://dev.mysql.com/doc/refman/5.6/en/partitioning-linear-hash.html  
  
  
  
前面的更快兩個字是單調優勢，但在後面的一句 `data is less likely to be evenly distributed between partitions` 就很含糊了。  
  
雖然重組分區的人很少，甚至在實務上都不會把這當成常規操作。可能因為這個原因，我在網上幾乎找不到Linear Hash 的經驗分享。就連它的長處有多長和短處有多短都沒找到。  
這看起來對橫向擴展明明很有益處的。  
  
這個less likely 到底有多less 呢?  
  
相似的疑問在這個SE 的冷門答案有提過。https://stackoverflow.com/questions/59650198/mysql-linear-hash-partitioning  
  
```  
V = POWER(2, CEILING(LOG(2, num)))  
N = F(k) & (V - 1)  
While N >= num  
{ V = V / 2   
  N = N & (V - 1)  
}  
Assign key k to N  
```  
~~是甚麼鬼東西，是人話嗎？~~  
  
  
  
那麼我做個測試吧。  
  
跟據這份文件: https://github.com/mysql/mysql-server/blob/8.0/sql/sql_partition.cc#L1378  
  
我試著重現一個Linear Hash 的結果。  
  
我花了一點時間寫了下面的C++ 代碼:  
  
```c++  
#include <iostream>  
#include <stdint.h>  
  
typedef uint32_t uint32;  
typedef int64_t longlong;  
// typedef uint  
using namespace std;  
  
uint get_linear_hash_mask(uint num_parts) {  
  uint mask;  
  
  for (mask = 1; mask < num_parts; mask <<= 1)  
    ;  
  return mask - 1;  
}  
  
static uint32 get_part_id_from_linear_hash(longlong hash_value, uint mask,  
                                           uint num_parts) {  
  uint32 part_id = (uint32)(hash_value & mask);  
  
  if (part_id >= num_parts) {  
    uint new_mask = ((mask + 1) >> 1) - 1;  
    part_id = (uint32)(hash_value & new_mask);  
  }  
  return part_id;  
}  
int main()  
{  
    int p=10;  
    for(int i =0;i<=200;i++){  
        cout<<get_part_id_from_linear_hash(i,get_linear_hash_mask(p),p)<<",";  
    }  
    cout<<"\n";  
    for(int i =0;i<=2000;i+=10){  
        cout<<get_part_id_from_linear_hash(i,get_linear_hash_mask(p),p)<<",";  
    }  
    cout<<"\n";  
    for(int i =0;i<=2000;i+=4){  
        cout<<get_part_id_from_linear_hash(i,get_linear_hash_mask(6),6)<<",";  
    }  
      
  
    return 0;  
}  
```  
  
```  
0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,9,2,3,4,5,6,7,0,1,2,3,4,5,6,7,8,  
0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,2,4,6,8,2,4,6,0,  
0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,4,0,  
  
```  
  
可見上面是個無迴圈的高速取餘(利用位遮蔽)。這對於一般的HASH 或者自增鍵來說是沒問題的。但是如果數據的值間隔和分區數其中是另一個的倍數或有公因數組合時，分配就會不平了。  
  
希望這文章能幫到一些人吧。