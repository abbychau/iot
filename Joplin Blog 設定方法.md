---  
title: Joplin Blog 設定方法  
updated: 2022-10-31 14:36:40Z  
created: 2022-10-31 13:55:24Z  
share: true  
---  
  
Abby 的Joplin Blog 設定方法  
  
主題和Blog 內容只限Abby 使用，和讓我自己日後參考，以下路徑請按個人需求修改。  
  
1. https://github.com/settings/tokens , 建立一個public_repo(這個就夠) 的token  
2. 打開 C:\Users\abbyc\.config\joplin-desktop\plugin-data  
3. git clone https://github.com/abbychau/joplin-plugin-page-publisher-theme ylc395.pagesPublisher  
4. ctrl+shift+p -> 選 publisher  
5. 按Generate -> http://localhost:3000 檢查有沒有錯誤  
  
完  
  
註: 不知何故 `_assets/main.css` 沒法引入，暫時完全inline。