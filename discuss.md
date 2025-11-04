# 討論事項

## 2025/10/28

1. 群聯 上的 nginx 將 /usr/sbin/nginx 指向 port: 3000。
   
2. 未來的 adas_dashboard 和 adas-one-backend 及 ADAS-one-Demo 都由 /usr/sbin/nginx 拖管
   
3. ADAS-one-Demo 後端 logic Response 要調整

4. 研究授權（License）機制


---

## 2025/11/03

1. 登人LOGIN 交接 with 學姐
   
2. Dify研究(SSO，TOOL、MCP關係設計)

3. prompt選單 UI/UX設計

4. 決定 POST: /api/security-analysis-ai 拆出一個微服務，定期每分鐘trigger並將資料寫入redis中。(請Jessie研究規劃以Cloudflare為demo)

---

## 2025/11/04

1. 協助 Jessie 完成 Cloudflare 中 work 的程式碼，加密流程要變更，要在12/12前完成。
   
	- 加密方式由「對稱」式(AES)改為「非對稱」式(RSA)
  
	- 有可能原始碼已移失
  
2. ADAS ONE: 11月主力開發AI頁面功能
	
    - AI Security Analysis (改寫 /api/security-analysis-ai)
	
    - Prompt library 
	
    - AI library (系統選擇AI Model介面)

---