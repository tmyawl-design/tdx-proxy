export default {
  async fetch(request, env, ctx) {
    const corsHeaders = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, HEAD, POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization, X-Requested-With",
      "Access-Control-Max-Age": "86400",
    };

    if (request.method === "OPTIONS") {
      return new Response(null, { status: 204, headers: corsHeaders });
    }

    try {
      // 🚀 請在此處換上您正確的 TDX 金鑰（注意前後不要有空格喔）
      const CLIENT_ID = "您的_TDX_CLIENT_ID";
      const CLIENT_SECRET = "您的_TDX_CLIENT_SECRET";

      // 1. 向 TDX 請求 Token
      const tokenUrl = "https://tdx.transportdata.tw/auth/realms/TDXConnect/protocol/openid-connect/token";
      const tokenRes = await fetch(tokenUrl, {
        method: "POST",
        headers: { "content-type": "application/x-www-form-urlencoded" },
        body: `grant_type=client_credentials&client_id=${CLIENT_ID}&client_secret=${CLIENT_SECRET}`
      });
      
      if (!tokenRes.ok) {
        return new Response(JSON.stringify({ error: "TDX金鑰驗證失敗，請檢查ID與Secret是否打錯" }), { status: 200, headers: corsHeaders });
      }
      
      const tokenData = await tokenRes.json();
      const token = tokenData.access_token;

      // 2. 優先嘗試抓取最穩定的 V2 資料
      let tdxUrl = "https://tdx.transportdata.tw/api/basic/v2/Parking/OffStreet/Availability/City/Taichung?%24format=JSON";
      let dataRes = await fetch(tdxUrl, { headers: { "authorization": `Bearer ${token}` } });
      
      // 3. 備用防線：如果 V2 被政府關閉，自動切換成 V3 特快車
      if (!dataRes.ok) {
        tdxUrl = "https://tdx.transportdata.tw/api/advanced/v3/Parking/OffStreet/Availability/City/Taichung?%24format=JSON";
        dataRes = await fetch(tdxUrl, { headers: { "authorization": `Bearer ${token}` } });
      }

      if (!dataRes.ok) {
        return new Response(JSON.stringify({ error: `政府伺服器拒絕提供車位資料(狀態碼:${dataRes.status})` }), { status: 200, headers: corsHeaders });
      }
      
      const parkingData = await dataRes.json();
      return new Response(JSON.stringify(parkingData), {
        status: 200,
        headers: { "Content-Type": "application/json;charset=UTF-8", ...corsHeaders }
      });

    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
        status: 200,
        headers: { "Content-Type": "application/json", ...corsHeaders }
      });
    }
  }
};
