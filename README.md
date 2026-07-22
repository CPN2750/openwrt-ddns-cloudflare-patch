README.md
# OpenWRT ddns-scripts Cloudflare Detect Registered IP Patch  

## 📖 Project Purpose 專案目的
+ English:
	OpenWrt’s ddns-scripts originally detect registered IP via DNS query. When using Cloudflare with Proxy (orange cloud) enabled, DNS queries return Cloudflare edge node IPs instead of the backend registered IP.
	This causes ddns-scripts to misjudge and force update every cycle.
	This patch modifies get_registered_ip() in dynamic_dns_functions.sh to directly call the Cloudflare API, retrieving the registered IP from Cloudflare backend records.
+ 中文:
	OpenWrt 的 `ddns-scripts` 原本透過 DNS 查詢來判斷已註冊的 IP。但在使用 **Cloudflare** 並開啟 Proxy（橘色雲）時，DNS 查詢會回傳 Cloudflare 邊緣節點 IP，而不是 Cloudflare 後台登記的紀錄。  
	這會導致 ddns-scripts 誤判，認為已註冊 IP 與 WAN IP 不一致，結果就是 **每次檢查都需要強制更新 (force update)**。  
	此修補程式在 `dynamic_dns_functions.sh` 中的 `get_registered_ip()` 功能函數中添加一段功能，直接呼叫 **Cloudflare API**，從「Cloudflare 後台紀錄」取得已登記的IP。

---

## 📂 Project Structure 專案結構
```plaintext
openwrt-ddns-cloudflare-patch/
├── README.md
├── get_cf_registered_ip
└── patch/
    └── dynamic_dns_functions.sh.patch
```

---

## ⚙️ Principle 原理
1. Detect if `service_name` is `cloudflare.com-v4` or `cloudflare.com-v6`.
2. Use `curl` to call Cloudflare API:
   - Get Zone ID
   - Get DNS Record (A/AAAA)
3. Parse JSON "content" → extract IP.
4. Update `REGISTERED_IP` and `/var/run/ddns/$SECTION.ip`.
5. Success → `return 0`, Failure → `return 127`.

---

## 🔄 Flow Comparison 流程比較圖 (ASCII Style)

```
Original ddns-scripts Flow (原始流程)
-----------------------------------
get_registered_ip()
        |
        v
 DNS Query (nslookup/drill/host)
        |
        v
 Result: Cloudflare Proxy IP
        |
        v
 Compare Proxy IP vs Local WAN IP
        |
        v
 Always mismatch --> Force update every cycle


Cloudflare Patch Flow (修補後流程)
---------------------------------
get_registered_ip()
        |
        v
 Detect service_name = cloudflare.com-v4/v6
        |
        v
 Call Cloudflare API (curl)
        |
        v
   +--> Get Zone ID
   |
   +--> Get DNS Record (A/AAAA)
        |
        v
 Parse JSON "content" --> Extract IP
        |
        v
 Compare Real Record IP vs Local WAN IP
        |
        v
 Correct detection --> Update only when needed
```

---

## 📜 Log Example 實際日誌範例

**Before Patch 修補前**
```
2026-07-22 21:40:01 INFO : Registered IP '104.16.123.96' detected
2026-07-22 21:40:01 INFO : Local WAN IP '203.0.113.45' detected
2026-07-22 21:40:01 WARN : IP mismatch, forcing update
2026-07-22 21:40:02 INFO : Update successful
...
(repeats every cycle 每次檢查都重複)
```

➡ 因為 DNS 查詢回傳的是 Cloudflare Proxy IP (`104.16.123.96`)，ddns-scripts 認為 IP 不一致，導致 **每次檢查都強制更新**。

---

**After Patch 修補後**
```
2026-07-22 21:50:01 INFO : Cloudflare Registered IP '203.0.113.45' detected
2026-07-22 21:50:01 INFO : Local WAN IP '203.0.113.45' detected
2026-07-22 21:50:01 INFO : IPs match, no update required
...
(next cycles show same 後續檢查皆一致，只有 WAN IP 改變時才更新)
```

➡ 使用 Cloudflare API 後，ddns-scripts 正確抓到 **後台紀錄 IP**，只有在 WAN IP 真正改變時才會更新。

---

## 📜 `get_cf_registered_ip` 程式碼
```sh
    # Get registered ip from Cloudflare patch start
    if [ "$service_name" = "cloudflare.com-v4" ] || [ "$service_name" = "cloudflare.com-v6" ]; then
        local __DATFILE __ERRFILE __ZONEID __DATA __TYPE __HOST __DOMAIN CF_IP

        __HOST=$(printf %s "$domain" | cut -d@ -f1)
        __DOMAIN=$(printf %s "$domain" | cut -d@ -f2)
        [ -z "$__HOST" ] && __HOST=$__DOMAIN
        [ "$__HOST" != "$__DOMAIN" ] && __HOST="${__HOST}.${__DOMAIN}"

        [ $use_ipv6 -eq 0 ] && __TYPE="A" || __TYPE="AAAA"

        __DATFILE="/var/run/ddns/${SECTION}_cf.dat"
        __ERRFILE="/var/run/ddns/${SECTION}_cf.err"

        __PRGBASE="$CURL -s --stderr $__ERRFILE -o $__DATFILE"
        if [ "$username" = "Bearer" ]; then
            __PRGBASE="$__PRGBASE --header 'Authorization: Bearer $password'"
        else
            __PRGBASE="$__PRGBASE --header 'X-Auth-Email: $username' --header 'X-Auth-Key: $password'"
        fi
        __PRGBASE="$__PRGBASE --header 'Content-Type: application/json'"

        # Get zone id
        eval "$__PRGBASE 'https://api.cloudflare.com/client/v4/zones?name=$__DOMAIN'"
        __ZONEID=$(grep -o '"id":\s*"[^"]*' $__DATFILE | grep -o '[^"]*$' | head -1)

        # Get DNS Record
        eval "$__PRGBASE 'https://api.cloudflare.com/client/v4/zones/$__ZONEID/dns_records?name=$__HOST&type=$__TYPE'"
        __DATA=$(grep -o '"content":\s*"[^"]*' $__DATFILE | grep -o '[^"]*$' | head -1)

        if [ $use_ipv6 -eq 0 ]; then
            CF_IP=$(printf "%s" "$__DATA" | grep -m 1 -o "$IPV4_REGEX")
        else
            CF_IP=$(printf "%s" "$__DATA" | grep -m 1 -o "$IPV6_REGEX")
        fi

        if [ -n "$CF_IP" ]; then
            eval "$1=\"$CF_IP\""
            [ -z "$IPFILE" ] || echo "$CF_IP" > "$IPFILE"
            write_log 7 "Cloudflare Registered IP '$CF_IP' detected"
            return 0
        else
            [ -z "$IPFILE" ] || echo "" > "$IPFILE"
            write_log 4 "Cloudflare Registered IP not found"
            return 127
        fi
    fi
    # Patch end
```

## 🔧 Installation 安裝方式

**Steps 步驟**
```bash
# 1. Backup original file 備份原始檔案
cp /usr/lib/ddns/dynamic_dns_functions.sh /usr/lib/ddns/dynamic_dns_functions.sh.bak

# 2. Edit the file manually 手動編輯檔案
vi /usr/lib/ddns/dynamic_dns_functions.sh

# 3. Locate function 找到函數定義
get_registered_ip() {
    Insert Cloudflare patch code here 在這裡插入 Cloudflare 修補程式程式碼
    # Original code continues below 原本程式碼繼續在下面
}

# 4. Save and exit 儲存並離開
:wq

# 5. Restart ddns service 重啟 ddns 服務
/etc/init.d/ddns restart
```

---

## 📦 Requirements
+ OpenWrt 21.02+
+ 已安裝 ddns-scripts 與 curl
+ Cloudflare API Token (建議使用 Token 而非 Global API Key)
  + 權限需求：Zone:Read, DNS:Edit

---

