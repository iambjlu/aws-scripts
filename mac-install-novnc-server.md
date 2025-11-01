# macOS安裝noVNC Server
使用時連線到 <code>https://\<IP\>:6080</code><br>
並輸入使用者名稱和密碼<br>
很重要 一定要https不然有可能會出錯<br><br>


先決條件：<br>
可以存取Macintosh桌面環境<br>
到設定➡️一般➡️共享，開啟螢幕共享總開關<br>
並確認欲使用使用者有權使用<br><br>


## 安裝noVNC Server
終端機執行
<pre>
cd ~
#建立憑證
openssl req -new -x509 -days 365 -nodes \
  -out self.crt \
  -keyout self.key
git clone https://github.com/novnc/noVNC.git
pip install websockify
  </pre>
並且修改 ~/noVNC/app/ui.js
找到「Populate」並把他下面一坨參數覆蓋，不然沒有游標
<pre>
        UI.initSetting('host', '');
        UI.initSetting('port', 0);
        UI.initSetting('encrypt', (window.location.protocol === "https:"));
        UI.initSetting('password');
        UI.initSetting('autoconnect', false);
        UI.initSetting('view_clip', false);
        UI.initSetting('resize', 'scale');
        UI.initSetting('quality', 3);
        UI.initSetting('compression', 8);
        UI.initSetting('shared', true);
        UI.initSetting('bell', 'on');
        UI.initSetting('view_only', false);
        UI.initSetting('show_dot', true);
        UI.initSetting('path', 'websockify');
        UI.initSetting('repeaterID', '');
        UI.initSetting('reconnect', true);
        UI.initSetting('reconnect_delay', 5000);
</pre>

## 使用noVNC Server：設定登入後打開
開機後需要使用自動登入或是手動登入以解密檔案保險箱以啟用此功能
把這些存成一個sh
<pre>
#!/bin/bash
# 跳到 noVNC 資料夾
cd ~/noVNC

# 先檢查 websockify 是否已啟動
if ! pgrep -f "websockify.*6080" > /dev/null; then
  echo "$(date) Starting noVNC..." >/dev/null 2>&1 &
  nohup websockify --web . --cert self.crt --key self.key 6080 localhost:5900 >/dev/null 2>&1 &
  lsof -i :6080
else
  echo "$(date) noVNC already running." >/dev/null 2>&1 &
  lsof -i :6080
fi
  </pre>
然後給執行權限
<pre>
chmod +x ~/noVNC/start-noVNC.sh
</pre>
之後加入「在登入時打開」
(設定➡️一般➡️登入項目與延伸功能)

- 請確保sh檔案預設使用終端機開啟<br>
在Finder對sh檔案按下輔助按鈕<br>
並按下「取得資訊」
<img width="295" height="83" alt="image" src="https://github.com/user-attachments/assets/55fdedcb-6abd-42ce-b5ed-8a2cdbb62139" />

## 如果發生問題要關掉
查PID
<pre>lsof -i :6080</pre>
找到後
<code>kill -9 \<PID\></code>


## 使用本地機器測試可用
<img width="1800" height="1077" alt="image" src="https://github.com/user-attachments/assets/a3e605c9-e7c5-4638-8bb2-296866b1329e" />


