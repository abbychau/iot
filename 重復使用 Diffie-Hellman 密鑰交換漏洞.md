---  
title: 重復使用 Diffie-Hellman 密鑰交換漏洞  
updated: 2023-05-05 07:29:17Z  
created: 2023-05-05 06:57:37Z  
tags:  
  - '#computing'  
share: true  
---  
  
  
![](https://i3.ytimg.com/vi/gVtjsd00fWo/maxresdefault.jpg)  
link: https://www.youtube.com/watch?v=gVtjsd00fWo  
  
好了，我其實不關心有沒有網站開始強制禁止 TLS 1.3 downgrade。下面是看完之後給還是不懂的人寫的一點小筆記。  
  
我把這個漏洞的原理命名為「重復使用Diffie-Hellman密鑰交換」好了  
  
概括來說：  
利用這個漏洞有三步  
  
1. 攻擊者先得知一些使用相同固定參數的Diffie-Hellman密鑰交換協商過程，記錄其中的通信數據和參數。這些參數通常由Web服務器預先設定，並在與客戶端進行安全協商時使用。參數下面再寫。  
2. 攻擊者使用這些記錄下來的通信數據和參數，利用離線破解的方法計算出協商的對稱密鑰。  
3. 利用對Negotiated Symetric Key (下稱NGK或代數s) 解密並記錄  
  
計算原理如下：  
1. 假設攻擊者攔截了使用固定參數的Diffie-Hellman密鑰交換協商過程，其中Web Server使用了預設的參數：  
  
		質數 or p = 23  
		底 or g = 5  
		Web服務器選擇的私鑰為a = 6  
		而現實上，p 是Oakley Group 2。(1024長度，768長度則是Group 1)  
  
2. 那麼攻擊者可以先計算 A = 5^6 mod 23 = 8  
  
3. client 和server 執行基於Diffie-Hellman 的 public key exchange  
  
4. 假設client private key, b = 15, 好好記住這個代數b, 這是攻擊者的目標 , 那麼public key, `B = 5^15 mod 23 = 19`  
  
5. Server 和 Client 各自使用對方的public key 和自己的private key 計算出NGK :  
在這個例子中：  
  
	web: s = B^a mod p = 19^6 mod 23 = 2  
	client: A^b mod p = 8^15 mod 23 = 2  
  
6. 這步是關鍵的一步  
攻擊者希望得到b，也就是client 的private key  
也就是`b = logg(s) mod p = log5(2) mod 23 = 15`  
  
～～～～～～～～  
第六步是這樣的：  
  
攻擊者可以先離線破解的部分在於：  
`g^b = s (mod p)` 是一個離散對數問題  
g和p 是固定的, s 是公開的  
  
那麼要在短時間內, 基於某個s 找出g^b，離散對數問題可以用BSGS 或者Pollard-rho 來解。time-complexity 是 sqrt(p)。而且parallelize well。  
  
那麼破解的重點在於把`g^b = s mod p` 裡面計算次序按p 為優先, 把所有可以給(p) -> (g , b) 制成一個離散的對數表。  
  
那麼就可以在破解時利用它來減低 time-complexity。  
計算所需的cpu time 在片的最後有說要多麼的苛刻就不重複了。  
  
我在Paper 中的 takeaway 是, 在這麼time-complexity reduction 下，增加key 長度的對破解時間需求的增加倍率不大。  
  
所以它終究來說不能成為長久可以依賴的算法。