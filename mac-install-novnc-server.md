macOS安裝noVNC Server
使用時連線到 \<IP\>:6080
輸入使用者名稱和密碼

安裝noVNC Server
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

使用noVNC Server：設定登入後打開
開機後需要使用自動登入或是手動登入以解密檔案保險箱以啟用此功能
把這些存成一個sh
<pre>
#!/bin/bash
# 跳到 noVNC 資料夾
cd ~/noVNC

# 把 log 存起來方便 debug
LOGFILE=~/novnc.log

# 先檢查 websockify 是否已啟動
if ! pgrep -f "websockify.*6080" > /dev/null; then
  echo "$(date) Starting noVNC..." >> $LOGFILE
  nohup websockify --web . 6080 localhost:5900 >> $LOGFILE 2>&1 &
else
  echo "$(date) noVNC already running." >> $LOGFILE
fi
  </pre>
然後給執行權限
<pre>
chmod +x ~/start-novnc.sh
</pre>
之後加入「在登入時打開」
(設定➡️一般➡️登入項目與延伸功能)
