# 🔹Lab 架構與角色

```
[ 筆電 ]
IP: 10.1.1.100
   |
   v
[ FortiGate 60E ]  ← 邊界防火牆 (VIP/NAT)
 Port2 (WAN): 10.1.1.101
 Port5 (DMZ): 172.16.1.100
 VIP: 10.1.1.101:443 → 172.16.1.99:443
   |
   v
[ FortiGate 50E ]  ← 內部防火牆 (內網 routing)
 Port2: 172.16.1.99
 Default Route: 0.0.0.0/0 → 172.16.1.100
   |
   v
[ 內部 LAN/Server ]
```

---

# 🔹實驗目的

1. 筆電 (外部 Client) 存取 60E 公開 IP
2. 60E 透過 **VIP** 把流量轉到 50E (內部防火牆)
3. 測試 **有/無 NAT、Default Route** 的差異
4. 學會逐步排錯（ping / GUI / CLI）

---

# 🔹硬體準備

* **設備**

  * FortiGate 60E ×1（邊界防火牆）
  * FortiGate 50E ×1（內部防火牆）
  * 筆電 ×1（管理 + 模擬外部 Client）

* **線材**

  * Console 線（USB-RJ45，管理用）
  * 網路線 (RJ45) ×3

    * 筆電 ↔ 60E (Inetrnal2)
    * 60E (Inetrnal5) ↔ 50E (lan2)
    * 筆電 ↔ 60E (MGMT/Internal1)（可選，用來初始管理）

* **電源**

  * 兩台 FortiGate 電源
  * 延長線

---

# 🔹接線拓樸

```
   [ 筆電 ]
   (Console 線) ───────>  [60E Console port]
   (RJ45) ───────────>  [60E Inetrnal2 (WAN)]
                           |
                     [60E Inetrnal5 (DMZ)]
                           |
                     (RJ45)
                           |
                     [50E lan2]
```

---

# 🔹檢查 COM Port (Windows)

在用 Console 線連 FortiGate 前，先確認電腦分配到的 COM Port：

1. `Win + R` → 輸入 `devmgmt.msc` → Enter
2. 開啟「裝置管理員」
3. 展開 **連接埠 (COM & LPT)**
4. 找到 `USB Serial Port (COMx)`

   * 記下 COM 號碼 (例如 COM3)
   * 在 XShell/Putty 設定時使用

⚙️ 連線參數：

```
Baud rate: 9600
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

---
好，我幫你把 FortiGate 的 **初始化與還原方式** 條理清楚整理成一份「一眼懂的筆記」。

---

# 🔹FortiGate 初始化 / 還原方法整理

## **方法一：`execute factoryreset`**

* **功能**：清除設定檔，恢復出廠 IP = `192.168.1.99/24`
* **操作**：

  1. 登入 Web GUI / SSH / Console
  2. CLI 輸入：

     ```bash
     execute factoryreset
     ```
  3. 確認後自動重開
* **適用**：知道密碼、只是要清掉設定
* **缺點**：不能處理「忘記密碼」

---

## **方法二：Reset 按鈕**

* **功能**：恢復出廠設定
* **操作**：

  1. 找到機器背後 / 側邊「RESET」小孔
  2. 迴紋針長按 10 秒
  3. 指示燈閃爍 → 自動重置
* **適用**：小型機種 (40/60/80 常見)
* **缺點**：不是每台都有

---

## **方法三：Console → Boot Menu (清除設定)**

* **功能**：透過 Console 進入開機選單，清掉設定
* **操作**：

  1. Console 線接好 → 開機
  2. 出現提示：

     ```
     Press any key to display configuration menu...
     ```

     立即按鍵
  3. 選擇：

     ```
     I  → Format boot device
     E  → Erase all data
     Y  → Confirm
     ```
  4. 自動重啟 → 回到出廠狀態
* **適用**：有 Console 線，但忘記密碼
* **缺點**：需要電腦 + Console 線

---

## **方法四：Maintainer 帳號（舊機型專屬）**

* **功能**：忘記密碼時用「序號」登入，重設 admin 密碼
* **帳號**：`maintainer`
* **密碼格式**：`bcpb<序號>`

  * 例：序號 `FGT60E1234567890` → 密碼 `bcpbFGT60E1234567890`
* **限制**：

  * 只能在開機後 **30 秒內 Console 登入**
  * **新版 (FortiOS 6.x+) 幾乎都關掉了**
* **適用**：舊機型、忘記密碼
* **缺點**：新版機種用不到

---

## **方法五：Format Boot Device + 重灌 Firmware**

* **功能**：整顆 Flash 格式化 → 需要重新安裝 FortiOS
* **操作**：

  1. Console → Boot Menu
  2. 選擇 `F = Format boot device`
  3. FortiGate 只剩 Boot Loader
  4. 用 **TFTP Server** 下載 Firmware (`.out`)
* **需要準備**：

  * **FortiOS Firmware 檔**（官方 Support Portal 下載）
  * **TFTP Server 軟體**（例如 tftpd64）
  * Console 線
* **適用**：

  * Firmware 損壞
  * 升級失敗無法開機
  * 要「乾淨重灌」
* **缺點**：麻煩、要有 Firmware 檔

---

# 🔹Firmware 取得方式

* **唯一合法管道**：[Fortinet Customer Service & Support](https://support.fortinet.com/)
* 登入需求：

  * Fortinet 帳號
  * 註冊設備序號（有時不需要合約，也能下載部分版本）
  * 有效 FortiCare / FortiGuard 合約可下載最新版本

檔案格式：`.out`
用途：

* Web GUI 升級
* Console + TFTP 重灌

---

# 🔹比較表

| 方法                      | 功能範圍         | 適用情境            | 缺點                  |
| ----------------------- | ------------ | --------------- | ------------------- |
| `execute factoryreset`  | 清除設定         | 知道密碼，要回預設狀態     | 無法解決忘記密碼            |
| Reset 按鈕                | 清除設定         | 小型機種、方便快速       | 不是每台都有              |
| Console Boot Menu (E/I) | 清除設定         | 有 Console，忘記密碼  | 需要 Console 線        |
| Maintainer 帳號           | 重設 admin 密碼  | 舊機型、忘記密碼        | 新版已禁用               |
| Format + TFTP           | 全部清空 + 重灌 OS | Firmware 損壞、要重灌 | 需 Firmware 檔 + TFTP |

---

# 🔹初始化 FortiGate

1. Console 線接好 → XShell → Serial (9600 baud)

2. 登入：admin / (空白)

3. 恢復出廠：

   ```bash
   execute factoryreset
   ```

   → 預設管理 IP = `192.168.1.99/24`

4. 筆電設：

   ```
   IP: 192.168.1.10
   Mask: 255.255.255.0
   GW: 192.168.1.99
   ```

   → 瀏覽器開 [https://192.168.1.99](https://192.168.1.99) 登入 GUI

---

# 🔹設定 FortiGate 60E（邊界防火牆）

電腦接網路線到 60E 的 Port1
GUI → Network → Interface
預設的 Internal 只保留 1，其他 2~7 拿掉

1. **Internal2 (WAN)**

   ```
   IP: 10.1.1.101/24
   Role: WAN
   Allow Access: PING, HTTPS, SSH
   ```
   
   筆電設：

   ```
   IP: 10.1.1.100
   Mask: 255.255.255.0
   ```

   驗證：在筆電 CMD → `ping 10.1.1.101`
   驗證完改回自動取得 IP 位址

2. **Internal5 (DMZ)**

   ```
   IP: 172.16.1.100/24
   Role: DMZ
   Allow Access: PING
   ```

---

# 🔹設定 FortiGate 50E（內部防火牆）

電腦接網路線到 50E 的 Port1
GUI → Network → Interface
預設的 lan 只保留 1，其他 2~5 拿掉

1. **lan2**

   ```
   IP: 172.16.1.99/24
   Role: LAN / WAN 都可（只要方便）
   Allow Access: HTTPS, PING
   ```

2. **Default Route**

GUI → Network → Static Routes → Create New

   ```
   Destination: 0.0.0.0/0
   Gateway: 172.16.1.100
   Device: lan2
   ```

驗證：
* 拿另一條網路線接 60E 的 Port5 和 50E 的 Port2
* 電腦接網路線到 50E 的 Port1，在 50E CLI → `execute ping 172.16.1.100` (能通 60E Internal5)
* 電腦接網路線到 60E 的 Port1，在 60E CLI → `execute ping 172.16.1.99` (能通 50E lan2)

---

# 🔹建立 VIP（60E）

GUI → Policy & Objects → Virtual IPs → Create New

```
Name: VIP-HTTPS
Interface: Internal2
External IP: 10.1.1.101
Mapped IP: 172.16.1.99
```

---

# 🔹建立 Policy（60E）

1. **WAN → DMZ (HTTPS)**

   ```
   Incoming: Internal2
   Outgoing: Internal5
   Source: all
   Destination: VIP-HTTPS
   Service: HTTPS
   Action: ACCEPT
   NAT: Enable
   ```
---

# 🔹驗證流程

1. 筆電設：

   ```
   IP: 10.1.1.100
   Mask: 255.255.255.0
   GW: 10.1.1.101
   DNS: 8.8.8.8
   ```

2. 測試：

   ```
   ping 10.1.1.101   ← 能通 60E
   ping 172.16.1.99  ← 要看 60E Policy 是否允許 ICMP
   ```

3. 測 VIP：

   * 瀏覽器 → `https://10.1.1.101`
   * 流量應轉到 50E (172.16.1.99:443)

---

# 🔹實務對應

* **VIP**：公司對外公開服務 (Web, Mail) → 轉到內部伺服器
* **NAT**：確保回覆封包能回來
* **Default Route**：伺服器知道怎麼回外部
* **雙防火牆**：銀行、醫療、政府常見「邊界 + 內部分區」架構

---

# 名詞說明

### 🔹DMZ (Demilitarized Zone)

DMZ 全名是 **Demilitarized Zone**（非軍事區），在網路架構裡是一個 **介於外部網路 (Internet) 與內部 LAN 之間的「中立區」**。

它的主要功能：

* 把 **需要對外提供服務的伺服器**（Web、Mail、DNS…）放在 DMZ
* 即使被駭客入侵，也不會直接進入公司內部 LAN
* 強化分層安全（Segmentation），降低風險

### 架構比喻

你可以把它想像成「公司的大門口接待大廳」：

* **外部訪客 (Internet)**：只能進到大廳（DMZ）
* **公司內部 (LAN)**：還有一道門（內部防火牆）保護，不能直接被外部訪客碰到
* **保全 (防火牆)**：控制誰能進大廳、誰能從大廳再進內部

### 典型 DMZ 架構

```
[ Internet ]
     |
 [ 邊界防火牆 ]
     |
    DMZ  ← 放公開服務 (Web/Mail/DNS/FTP)
     |
 [ 內部防火牆 ]
     |
   [ LAN ]
```

* **DMZ**：只放公開伺服器
* **LAN**：只給內部員工使用
* **防火牆 Policy**：嚴格限制 DMZ ↔ LAN 之間的通道

### 在這個 LAB 的角色定位

* **角色**：在 60E 的 `Internal5` → 用來放置 50E（模擬內部伺服器）。
* **目的**：隔離外網 (WAN) 與內網 (LAN)。
* **實驗意義**：筆電打到 60E 的公開 IP (10.1.1.101)，再透過 VIP 導到 DMZ 內的 50E (172.16.1.99)。
* 60E 的 **Internal5** 被設定成 DMZ

  * IP: `172.16.1.100`
  * 負責跟內部 50E 溝通
* 50E 的 **lan2** 相當於「內部伺服器」的入口 (`172.16.1.99`)

所以當筆電 (外部 Client) 要存取 50E：

1. 先打到 60E (WAN: `10.1.1.101`)
2. 60E 透過 **VIP** 把流量丟到 DMZ (`172.16.1.99`)
3. 50E 再處理內部服務

---

### 🔹STP （Spanning Tree Protocol）

* **用途**：避免二層網路（Switch Layer 2）出現迴圈。

  * 例如 Switch A ↔ Switch B ↔ Switch C ↔ Switch A，會造成 **廣播風暴 (Broadcast Storm)**。
* **防火牆**：

  * **通常沒有啟用 STP**，因為防火牆是三層設備（L3 Routing），不是 L2 交換器。
  * Port-to-Port 轉送走的是 **IP Routing**，而不是 Switch 的 MAC Flooding。
* ✅ 所以你看到「STP 可以關掉」，就是因為防火牆用不到它。

### 在這個 LAB 的角色定位

* **角色**：在這個 LAB 完全 **沒有用到**。
* **原因**：你用的是防火牆，傳遞走的是 L3 IP，不會有 L2 迴圈問題。
* **實驗意義**：你可以理解「STP 關掉也沒影響」，因為這裡沒有多顆交換器。

---

### 🔹NTP （Network Time Protocol）

* **功能**：讓所有設備的時間保持一致（時鐘同步）。
* **為什麼重要**：

  * 日誌 (Log) 對時：60E 跟 50E 的 Log 如果沒同步，排錯會很亂。
  * SSL / VPN / 憑證：時間錯誤會導致驗證失敗。
* **常用 NTP Server**：`time.google.com`、`time.windows.com`、`pool.ntp.org`

### 在這個 LAB 的角色定位

* **角色**：對於 **封包測試**沒影響，但對 **日誌紀錄**有意義。
* **實驗意義**：如果要觀察 60E/50E 的 policy log，建議把時間先同步，否則 Log 看起來會不一致。

---

### 🔹MGMT

* **全名**：Management Port
* **用途**：專門給管理人員連 GUI/CLI，不參與生產網路流量。
* **好處**：

  * 管理流量與用戶流量分開，避免互相干擾。
* 在 FortiGate 上 → `MGMT Port` 預設 IP 通常是 `192.168.1.99`。
### 在這個 LAB 的角色定位

* **角色**：管理用的 Port，預設 IP 是 192.168.1.99。
* **在這個 LAB**：你一開始用它登入 GUI，設定好後就可以不用它，轉而用 port2/port5 來做測試。

---

### 🔹LLDP （Link Layer Discovery Protocol）

* **用途**：網路設備互相「自我介紹」。

  * 會廣播「我是誰、我是什麼裝置、我的 IP/MAC、廠牌型號」。
* **場景**：

  * 拓樸偵測（像 Cisco CDP，Fortinet 也支援 LLDP）。
  * NMS（網管系統）會自動繪製網路拓樸。
* ✅ 安全性建議：內網用可以開，外網建議關閉，避免洩漏設備資訊。

### 在這個 LAB 的角色定位

* **角色**：設備之間的「自我介紹」。
* **在這個 LAB**：幾乎沒用，你不需要靠 LLDP 去發現鄰居設備。
* **實驗意義**：如果開啟，60E 與 50E 可以互相報告「我是 FortiGate」。但這只是資訊，不影響封包轉送。
---

### 🔹TTL （Time To Live）

* **用途**：IP 封包的「生命週期」。
* **運作方式**：每經過一台 Router/Firewall，TTL 減 1，直到 0 → 封包被丟棄。
* **好處**：防止封包在網路裡「無限迴圈」。
* **範例**：

  * Windows 預設 TTL=128
  * Linux/Mac 預設 TTL=64

### 在這個 LAB 的角色定位

* **角色**：控制封包壽命。
* **在這個 LAB**：封包從筆電 → 60E → 50E → 回筆電，最多經過 2 hops，TTL 幾乎不會歸零。
* **實驗意義**：你不需要調整 TTL，但透過 `tracert` 可以看到封包經過的路徑 (60E、50E)。

---

### 🔹NAT （Network Address Translation）

* **用途**：把內部私有 IP 換成公有 IP。
* **有沒有開 NAT → 影響很大**

  * **開 NAT**：

    * 內部 Client → 外部 Server（只看到防火牆 IP，不知道內部 IP）。
    * 適合上網、保護內部架構。
  * **關 NAT**：

    * 外部 Client 必須能直接路由回內部 IP。
    * 適合伺服器對外服務（常搭配 VIP）。

---

### 在這個 LAB 的角色定位

* **角色**：最重要的功能，決定回覆能不能回來。
* **在這個 LAB**：

  * 開 NAT → 50E 只會看到 60E (172.16.1.100) 當來源，回覆一定能回來。
  * 關 NAT → 50E 會看到筆電的 10.1.1.100 當來源，若 50E 沒有回 10.1.1.0/24 的路由，封包會丟掉。
---

# 🔹實驗延伸：GW 與 NAT 組合差異

| 筆電 GW             | NAT 狀態  | 結果                                                                                     |
| ----------------- | ------- | -------------------------------------------------------------------------------------- |
| 設 GW=10.1.1.101 | 開 NAT | 筆電能 ping VIP (10.1.1.101)，也能透過 VIP 連到 50E，回覆封包順利。                                      |
| 設 GW=10.1.1.101 | 關 NAT | 筆電 ping 10.1.1.101 成功，但打到 172.16.1.99 會失敗（50E 不知道怎麼回 10.1.1.100）。 unless 50E 有靜態路由指回去。 |
| 不設 GW           | 開 NAT | 筆電只能通 10.1.1.101 (60E)，VIP 測試會失敗（因為沒有預設路徑回應）。                                          |
| 不設 GW           | 關 NAT | 什麼都不通，因為筆電連外都沒有路徑。                                                                     |

**重點：**

* **有無設 GW** → 決定「筆電能不能往外發」。
* **有無 NAT** → 決定「回覆能不能回來」。
* 兩個缺一不可，否則 Lab 就會失敗。

---
