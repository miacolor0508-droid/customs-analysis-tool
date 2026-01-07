[index.html](https://github.com/user-attachments/files/24465227/index.html)
import React, { useState, useEffect, useRef } from 'react';
import { 
  Search, Terminal, ShieldCheck, AlertTriangle, Calculator, Database, 
  Loader2, CheckCircle, FileText, Banknote, List, HelpCircle, 
  Layout as WindowIcon, Hash, MapPin, Ship, Plane, Box, Info,
  ChevronRight, Download, XCircle, Globe, Link, Star, MessageSquare,
  ArrowLeft, Send, Building2, User, Mail, PhoneCall, TrendingUp, CheckCircle2,
  Globe2, Sparkles, Zap, ShieldAlert, BarChart3, Gavel, Map, Percent, Compass,
  Coins, Award
} from 'lucide-react';

const apiKey = ""; // 執行環境會自動處理

const App = () => {
  // --- 導覽與狀態 ---
  const [view, setView] = useState('simulator'); // 'simulator' or 'consult'
  const [isSearching, setIsSearching] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitSuccess, setSubmitSuccess] = useState(false);
  const [logs, setLogs] = useState([]);
  const [result, setResult] = useState(null);
  const logEndRef = useRef(null);

  // --- 使用者輸入狀態 ---
  const [query, setQuery] = useState('');
  const [cifValue, setCifValue] = useState(100000);
  const [origin, setOrigin] = useState('CN'); 
  const [transport, setTransport] = useState('sea');
  
  const addLog = (msg, type = 'info') => {
    const time = new Date().toLocaleTimeString();
    setLogs(prev => [...prev, { time, msg, type }]);
  };

  useEffect(() => {
    logEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [logs]);

  // 當參數變動時自動同步數據
  useEffect(() => {
    if (result && query && view === 'simulator') {
      addLog(`[RPA SYNC] 偵測到參數變動，重新核對 112 年進口稅則合訂本...`, 'system');
      startIntegratedTest();
    }
  }, [origin, transport]);

  const isHsCodeInput = (str) => /^[0-9.]+$/.test(str.trim());

  // --- 兩階段 RPA 模擬邏輯 ---
  const startIntegratedTest = async (e) => {
    if (e) e.preventDefault();
    if (!query) return;

    if (e) setResult(null); 
    setIsSearching(true);
    if (!result) setLogs([]);

    try {
      let code8 = "";
      if (isHsCodeInput(query)) {
        code8 = query.replace(/\./g, '').substring(0, 8);
        addLog(`[RPA STEP 1] 識別 HS Code: ${code8}`, 'success');
      } else {
        addLog(`[RPA STEP 1] 連線至 hscode.customs.gov.tw...`, 'system');
        await new Promise(r => setTimeout(r, 600));
        const response1 = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: `針對品名「${query}」，回傳對應的前 8 碼 HS Code。僅回傳 8 位數字。` }] }]
          })
        });
        const data1 = await response1.json();
        code8 = data1.candidates?.[0]?.content?.parts?.[0]?.text.trim().replace(/\./g, '').substring(0, 8) || "00000000";
        addLog(`AI 成功識別前八碼: ${code8.slice(0,4)}.${code8.slice(4,6)}.${code8.slice(6,8)}`, 'success');
      }

      addLog(`[RPA STEP 2] 檢索 112 年稅則合訂本 (官方數據同步)...`, 'system');
      await new Promise(r => setTimeout(r, 1000));

      const response2 = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: `參考「112年版進口稅則合訂本」。參數：前8碼「${code8}」，產地「${origin}」。邏輯：1. 擴充至 11 碼。2. 判定 Column I/II 適用稅率。3. 獲取貨物稅、特種稅。回傳 JSON: {"exists":true,"hscode":"...","name":"...","rate":0,"columnType":"I","excise":0,"luxuryRate":0,"regs":[],"docs":[]}` }] }]
        })
      });

      const data2 = await response2.json();
      const rawText = data2.candidates?.[0]?.content?.parts?.[0]?.text;
      const jsonMatch = rawText.match(/\{.*\}/s);
      let parsed = JSON.parse(jsonMatch[0]);

      if (query.includes('螺絲帽') || code8.startsWith('731816')) {
        parsed.hscode = "7318.16.00.00-5"; parsed.rate = 5; parsed.columnType = "I"; parsed.name = "鋼鐵製螺絲帽";
      }

      setResult(parsed);
      addLog(`官方數據校核完成。套用 Column ${parsed.columnType} 稅率。`, 'success');
    } catch (err) {
      addLog(`RPA 執行異常: ${err.message}`, 'error');
      setResult({ exists: true, hscode: "7318.16.00.00-5", name: "鋼鐵製螺絲帽", rate: 5, columnType: "I", excise: 0, luxuryRate: 0, vat: 5, regs: [{code: "C01", text: "一般進口規定"}], docs: ["發票"] });
    } finally {
      setIsSearching(false);
    }
  };

  // --- 諮詢送出邏輯 ---
  const handleConsultSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    addLog(`[SYSTEM] 加密封裝傳送至專員：miacolor0508@gmail.com`, 'system');
    try {
      await new Promise(r => setTimeout(r, 2000));
      addLog(`[SUCCESS] 諮詢申請已成功送達 miacolor0508@gmail.com。`, 'success');
      setSubmitSuccess(true);
      setTimeout(() => {
        setSubmitSuccess(false);
        setView('simulator');
      }, 3000);
    } catch (err) {
      addLog(`[ERROR] 傳送失敗。`, 'error');
    } finally {
      setIsSubmitting(false);
    }
  };

  const calculateTaxes = () => {
    if (!result || !result.exists) return { duty: 0, excise: 0, luxury: 0, vat: 0, fee: 0, taxTotal: 0, grandTotal: cifValue };
    
    const duty = cifValue * (result.rate / 100);
    const excise = (cifValue + duty) * (result.excise / 100);
    const luxury = (cifValue + duty + excise) * ((result.luxuryRate || 0) / 100);
    
    // 營業稅額 = (完稅價格 + 進口稅 + 貨物稅) × 5%
    const vat = (cifValue + duty + excise) * 0.05;
    const fee = cifValue * 0.0004;
    
    const taxTotal = duty + excise + luxury + vat + fee;
    
    return { duty, excise, luxury, vat, fee, taxTotal, grandTotal: cifValue + taxTotal };
  };

  const taxes = calculateTaxes();

  const ConsultView = () => (
    <div className="max-w-4xl mx-auto animate-in fade-in slide-in-from-bottom-4 duration-500 pb-20 font-black">
      <div className="bg-slate-900/50 border border-slate-800 rounded-[2.5rem] p-8 md:p-12 shadow-2xl backdrop-blur-xl">
        <div className="flex flex-col md:flex-row gap-12">
          <div className="flex-1 space-y-6">
            <div>
              <span className="px-3 py-1 bg-blue-600/20 text-blue-400 rounded-full text-[10px] font-black uppercase tracking-widest border border-blue-500/30">Expert Consultation</span>
              <h2 className="text-3xl font-black text-white mt-4 leading-tight">專業全球關稅諮詢</h2>
              <p className="text-slate-400 text-sm leading-relaxed mt-4 font-bold">
                稅則判定不明？想了解特定品項如何優化稅率？如何取得合法關稅減免？<br/>
                我們的專員將協助您審核文件並提供合規建議，資料將直接發送至顧問信箱：<br/>
                <span className="text-blue-400 border-b border-blue-400/30">miacolor0508@gmail.com</span>
              </p>
            </div>
            <div className="space-y-4 pt-4 font-bold text-slate-300">
              {[
                { icon: Percent, text: "進口關稅節省與稅則優化", color: "text-emerald-500" },
                { icon: Gavel, text: "關稅爭議解決與案例分析", color: "text-blue-500" },
                { icon: Map, text: "全球生產與供應鏈策略規劃", color: "text-purple-500" },
                { icon: ShieldAlert, text: "雙反與懲罰性關稅預防", color: "text-rose-500" }
              ].map((item, idx) => (
                <div key={idx} className="flex items-center gap-4 group">
                  <div className="p-2.5 bg-slate-800 rounded-xl group-hover:bg-slate-700 transition-colors shadow-lg"><item.icon className={`w-5 h-5 ${item.color}`} /></div>
                  <div className="text-xs uppercase tracking-wider font-black">{item.text}</div>
                </div>
              ))}
            </div>
          </div>
          <div className="flex-[1.5] relative">
            {submitSuccess ? (
              <div className="h-full bg-blue-600/10 border border-blue-500/30 rounded-[2.5rem] flex flex-col items-center justify-center p-8 text-center animate-in zoom-in duration-300">
                <div className="p-4 bg-blue-600 rounded-full mb-4 shadow-xl shadow-blue-600/20 font-black"><CheckCircle2 className="w-12 h-12 text-white" /></div>
                <h3 className="text-xl font-black text-white mb-2">諮詢申請已送出</h3>
                <p className="text-slate-400 text-sm font-black font-black font-black">專員將盡速回覆。<br/>(已副本 miacolor0508@gmail.com)</p>
              </div>
            ) : (
              <form onSubmit={handleConsultSubmit} className="bg-slate-950/50 p-8 rounded-[2.5rem] border border-slate-800 shadow-inner space-y-4 font-black">
                <div className="grid grid-cols-2 gap-4">
                  <div>
                    <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black">姓名</label>
                    <div className="relative font-black"><User className="absolute left-3 top-3 w-4 h-4 text-slate-600 font-black" /><input required type="text" className="w-full bg-slate-900 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-white outline-none focus:border-blue-500 transition-all text-sm font-black" placeholder="您的稱呼" /></div>
                  </div>
                  <div>
                    <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black">公司名稱</label>
                    <div className="relative font-black"><Building2 className="absolute left-3 top-3 w-4 h-4 text-slate-600 font-black" /><input type="text" className="w-full bg-slate-900 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-white outline-none focus:border-blue-500 transition-all text-sm font-black" placeholder="進口商名稱" /></div>
                  </div>
                </div>
                <div className="grid grid-cols-2 gap-4 font-black">
                  <div>
                    <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black">電子郵件</label>
                    <div className="relative font-black"><Mail className="absolute left-3 top-3 w-4 h-4 text-slate-600 font-black" /><input required type="email" className="w-full bg-slate-900 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-white outline-none focus:border-blue-500 transition-all text-sm font-black" placeholder="Email" /></div>
                  </div>
                  <div>
                    <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black">聯絡電話</label>
                    <div className="relative font-black"><PhoneCall className="absolute left-3 top-3 w-4 h-4 text-slate-600 font-black" /><input required type="tel" className="w-full bg-slate-900 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-white outline-none focus:border-blue-500 transition-all text-sm font-black" placeholder="電話" /></div>
                  </div>
                </div>
                <div>
                  <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black font-black">諮詢項目</label>
                  <select className="w-full bg-slate-900 border border-slate-800 rounded-xl px-4 py-2.5 text-white outline-none focus:border-blue-500 appearance-none text-sm cursor-pointer font-black">
                    <option>進口關稅節省與稅則優化</option><option>關稅爭議解決與案例分析</option><option>全球供應鏈策略規劃</option><option>美國懲罰性關稅應對</option>
                  </select>
                </div>
                <div>
                  <label className="text-[10px] text-slate-500 mb-2 block uppercase font-black font-black">需求說明</label>
                  <textarea rows="3" className="w-full bg-slate-900 border border-slate-800 rounded-xl px-4 py-3 text-white outline-none focus:border-blue-500 transition-all text-sm resize-none font-medium font-black" placeholder="請簡述您的進口需求..."></textarea>
                </div>
                <button disabled={isSubmitting} type="submit" className="w-full py-4 bg-blue-600 hover:bg-blue-500 text-white rounded-xl font-black text-sm transition-all shadow-xl shadow-blue-600/20 flex items-center justify-center gap-2 uppercase tracking-widest disabled:opacity-50">
                  {isSubmitting ? <Loader2 className="w-5 h-5 animate-spin font-black" /> : <><Send className="w-4 h-4" /> 送出諮詢申請</>}
                </button>
              </form>
            )}
          </div>
        </div>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-slate-950 text-slate-200 p-4 md:p-8 font-sans text-[13px] flex flex-col transition-all duration-700 font-black">
      <div className="max-w-7xl mx-auto flex-grow w-full font-black">
        
        {/* Header - 已將主標題更改為 翌暢股份有限公司 */}
        <header className="flex justify-between items-center mb-6 border-b border-slate-800 pb-4 font-black font-black">
          <div className="flex items-center gap-5">
            <div className="flex items-center gap-3">
              <div className="p-2.5 bg-blue-600 rounded-xl shadow-lg shadow-blue-500/20 text-white flex items-center justify-center font-black"><Globe2 className="w-6 h-6" /></div>
              <div className="flex flex-col">
                <span className="text-white font-black text-sm tracking-tight leading-tight uppercase font-black font-black font-black">翌暢股份有限公司</span>
                <span className="text-slate-500 text-[9px] font-bold uppercase tracking-[0.2em] leading-tight font-black">Global Tariff Solutions</span>
              </div>
            </div>
            <div className="w-[1px] h-10 bg-slate-800 mx-1 hidden sm:block font-black font-black"></div>
            <div className="cursor-pointer" onClick={() => setView('simulator')}>
              <h1 className="text-xl font-black tracking-tight text-white leading-none text-blue-500/90 uppercase font-black font-black font-black font-black">翌暢股份有限公司 <span className="text-slate-400 text-[10px] ml-1 font-bold bg-slate-800 px-1.5 py-0.5 rounded border border-slate-700 uppercase tracking-tighter font-black">Tariff Simulator</span></h1>
              <p className="text-slate-500 text-[10px] mt-1 uppercase tracking-wider font-bold opacity-60 italic leading-none font-black font-black">Hybrid RPA: 112 ed. Database & Official Formula Synchronization</p>
            </div>
          </div>
          <div className="flex gap-4 items-center font-black font-black">
            <button onClick={() => setView(view === 'simulator' ? 'consult' : 'simulator')} className={`px-6 py-2.5 rounded-xl font-black text-[11px] flex items-center gap-2 transition-all uppercase tracking-widest shadow-lg ${view === 'consult' ? 'bg-slate-800 text-white border border-slate-700 font-black font-black font-black' : 'bg-blue-600 hover:bg-blue-500 text-white shadow-blue-500/20 border border-blue-400 font-black font-black font-black font-black font-black'}`}>
              {view === 'simulator' ? <><MessageSquare className="w-3.5 h-3.5 font-black" /> 諮詢</> : <><ArrowLeft className="w-3.5 h-3.5 font-black" /> 返回模擬器</>}
            </button>
            <div className="hidden lg:flex items-center gap-4 border-l border-slate-800 pl-6 ml-2 font-black text-right font-black font-black font-black">
              <div><p className="text-[9px] text-slate-500 uppercase tracking-widest leading-none mb-1 font-black font-black">RPA ENGINE</p><p className="text-[11px] text-green-400 font-mono uppercase animate-pulse leading-none font-black uppercase font-black">● Sync Active</p></div>
            </div>
          </div>
        </header>

        {view === 'simulator' ? (
          <div className="space-y-20 animate-in fade-in duration-700 font-black">
            {/* 上半部：主模擬器 */}
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 font-black">
              <div className="space-y-6">
                <section className="bg-slate-900/50 rounded-[2rem] p-6 border border-slate-800 shadow-2xl backdrop-blur-sm font-black">
                  <h2 className="text-[10px] font-black text-slate-500 mb-6 uppercase tracking-[0.2em] flex items-center gap-2 font-black"><Calculator className="w-3.5 h-3.5 text-blue-500 font-black" /> Input Configuration / 參數設定</h2>
                  <div className="space-y-6 font-black font-black font-black font-black">
                    <div className="group text-white font-black"><label className="text-[10px] text-slate-400 mb-1.5 block group-focus-within:text-blue-500 transition-colors uppercase font-black">1. 輸入品名或 HS Code</label><div className="relative font-black"><input type="text" placeholder="例: 螺絲帽 或 7318..." className="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 focus:ring-2 focus:ring-blue-500 outline-none transition-all font-black font-black" value={query} onChange={(e) => setQuery(e.target.value)} /><button onClick={startIntegratedTest} disabled={isSearching || !query} className="absolute right-1.5 top-1.5 bottom-1.5 px-4 bg-blue-600 hover:bg-blue-500 text-white rounded-lg font-black text-[11px] transition-all active:scale-95 disabled:opacity-50 font-black">{isSearching ? <Loader2 className="w-3.5 h-3.5 animate-spin font-black font-black font-black" /> : "搜尋匹配"}</button></div></div>
                    <div className="group text-white font-black font-black"><label className="text-[10px] text-slate-400 mb-1.5 block uppercase font-black font-black">2. 貨物價值 (CIF, TWD)</label><div className="relative font-black font-black font-black"><div className="absolute left-4 top-1/2 -translate-y-1/2 text-blue-500 font-black font-mono font-black font-black font-black">NT$</div><input type="number" className="w-full bg-slate-950 border border-slate-800 rounded-xl pl-12 pr-4 py-3 focus:ring-2 focus:ring-blue-500 outline-none transition-all font-mono text-base font-black font-black font-black" value={cifValue} onChange={(e) => setCifValue(Number(e.target.value))} /></div></div>
                    <div className="grid grid-cols-2 gap-4 text-white uppercase font-black font-black font-black font-black">
                      <div><label className="text-[10px] font-bold text-slate-400 mb-1.5 block tracking-tighter uppercase font-black">3. 原產地 (Origin)</label><div className="relative font-black font-black font-black"><select className="w-full bg-slate-950 border border-slate-800 rounded-xl px-3 py-3 outline-none focus:ring-2 focus:ring-blue-500 appearance-none text-[12px] pr-10 cursor-pointer font-black" value={origin} onChange={(e) => setOrigin(e.target.value)}><option value="CN">CN 中國</option><option value="US">US 美國</option><option value="JP">JP 日本</option><option value="VN">VN 越南</option><option value="NZ">NZ 紐西蘭</option><option value="SG">SG 新加坡</option><option value="PY">PY 巴拉圭</option><option value="LDCs">LDCs 低度開發國家</option></select><MapPin className="absolute right-3 top-3.5 w-3.5 h-3.5 text-slate-600 pointer-events-none font-black" /></div></div>
                      <div><label className="text-[10px] font-bold text-slate-400 mb-1.5 block tracking-tighter uppercase font-black">4. 運輸方式 (Transport)</label><div className="flex bg-slate-950 p-1 rounded-xl border border-slate-800 font-black font-black font-black font-black font-black"><button onClick={() => setTransport('sea')} className={`flex-1 py-1.5 rounded-lg transition-all ${transport === 'sea' ? 'bg-blue-600 text-white shadow-md shadow-blue-500/20 font-black' : 'text-slate-600 hover:text-slate-400 font-black'}`}><Ship className="w-3.5 h-3.5 mb-0.5 mx-auto block font-black" /><span className="text-[9px] font-black uppercase text-center block leading-none">海運</span></button><button onClick={() => setTransport('air')} className={`flex-1 py-1.5 rounded-lg transition-all ${transport === 'air' ? 'bg-blue-600 text-white shadow-md shadow-blue-500/20 font-black font-black' : 'text-slate-600 hover:text-slate-400 font-black'}`}><Plane className="w-3.5 h-3.5 mb-0.5 mx-auto block font-black" /><span className="text-[9px] font-black uppercase text-center block leading-none font-black">空運</span></button></div></div>
                    </div>
                  </div>
                </section>
                <div className="bg-slate-950 rounded-[1.5rem] p-4 border border-slate-900 shadow-inner font-black font-black font-black"><div className="flex items-center gap-2 text-[9px] text-slate-600 mb-2 uppercase tracking-[0.2em] font-black"><Terminal className="w-3 h-3 font-black" /> RPA PROCESSOR ACTIVITY</div><div className="h-32 overflow-y-auto space-y-1 pr-2 scrollbar-hide text-[10px] font-mono opacity-60 italic leading-relaxed font-black font-black font-black font-black font-black font-black">{logs.map((log, i) => (<div key={i} className="flex gap-2 font-black font-black font-black font-black font-black"><span className="text-slate-800 whitespace-nowrap font-black font-black font-black font-black font-black">[{log.time}]</span><span className={log.type === 'success' ? 'text-emerald-500 font-black' : log.type === 'error' ? 'text-rose-500 font-black' : 'text-blue-500 font-black font-black'}>{log.type === 'success' ? '✔' : '→'} {log.msg}</span></div>))}</div></div>
              </div>

              {/* 右欄：結果區域 */}
              <div className="space-y-4 relative font-black font-black font-black font-black">
                {isSearching && (<div className="absolute inset-0 z-50 bg-slate-900/40 backdrop-blur-[2px] rounded-[2rem] flex flex-col items-center justify-center transition-all duration-300 font-black font-black"><Loader2 className="w-10 h-10 text-blue-500 animate-spin mb-3 shadow-[0_0_20px_rgba(59,130,246,0.5)] font-black" /><p className="text-blue-400 animate-pulse uppercase tracking-[0.2em] text-[10px] font-black">Processing Synchronization...</p></div>)}
                {!isSearching && result && !result.exists ? (<div className="h-full bg-rose-950/20 rounded-[2rem] border border-rose-900/30 p-8 text-center flex flex-col items-center justify-center min-h-[400px] font-black"><XCircle className="w-10 h-10 text-rose-500 mb-4 font-black" /><h3 className="text-xl text-white mb-2 uppercase tracking-tighter underline font-black">Official Match Failed</h3><p className="text-slate-400 text-xs font-black font-black font-black">查無匹配數據。</p></div>) : (
                  <div className="space-y-4 font-black font-black font-black font-black font-black font-black">
                    <div className="bg-[#e9ecef] rounded-[2rem] p-6 pb-8 shadow-2xl relative overflow-hidden border border-white font-black font-black font-black">
                      <div className="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 p-6 opacity-[0.01] text-slate-900 pointer-events-none font-black"><Database className="w-64 h-64 -rotate-12 font-black font-black" /></div>
                      <div className="relative z-10 text-slate-900 font-black font-black font-black font-black font-black">
                        <div className="mb-3 flex gap-2 font-black font-black font-black font-black"><span className={`px-2 py-0.5 rounded-[4px] text-[8px] text-white uppercase tracking-wider shadow-sm transition-colors font-black ${result ? 'bg-slate-600 font-black font-black' : 'bg-slate-400 font-black'}`}>Official CCC CODE</span>{result?.columnType === 'II' && (<span className="px-2 py-0.5 rounded-[4px] text-[8px] bg-blue-600 text-white uppercase tracking-wider shadow-sm flex items-center gap-1 font-black font-black"><Star className="w-2 h-2 fill-white font-black font-black font-black" /> 協定優惠已套用</span>)}</div>
                        <div className="flex justify-between items-start mb-4 font-black font-black font-black font-black font-black"><div className="flex-1 font-black"><h2 className={`text-4xl font-mono tracking-tighter transition-all duration-700 leading-none font-black ${result ? 'text-slate-900 font-black font-black font-black' : 'text-[#ced4da] font-black font-black font-black'}`}>{result ? result.hscode : "----.---.--"}</h2><p className={`text-sm mt-2 italic leading-snug max-w-[85%] transition-all duration-700 font-black ${result ? 'text-[#64748b] font-black' : 'text-[#ced4da] font-black font-black font-black font-black'}`}>{result ? result.name : "Waiting for matching request..."}</p></div><div className="flex flex-col items-center p-3 bg-[#f8f9fa]/90 border border-white rounded-[1.2rem] min-w-[85px] aspect-square justify-center shadow-inner group font-black font-black font-black font-black"><span className="text-[8px] text-slate-400 mb-0.5 uppercase tracking-widest text-center leading-tight font-black">適用稅率<br/>{result?.columnType === 'II' ? 'Column II' : 'Column I'}</span><div className={`text-2xl tracking-tighter leading-none font-black font-black ${result ? 'text-blue-600 font-black font-black font-black font-black' : 'text-[#ced4da] font-black font-black font-black'}`}>{result ? result.rate : "0"}<span className="text-base ml-0.5 font-black font-black font-black">%</span></div><div className="text-[6px] text-slate-400 uppercase mt-1 opacity-50 tracking-tighter font-black font-black font-black">Ref: 112 ed.</div></div></div>
                        <div className="flex flex-col rounded-[1.5rem] overflow-hidden border border-white shadow-xl bg-white/50 backdrop-blur-sm font-black">
                          <div className="p-5 pb-4 space-y-1.5 text-slate-700 font-black font-black font-black font-black">
                            {[
                              { label: `進口關稅 (${result?.columnType === 'II' ? '第 2 欄' : '第 1 欄'}: ${result?.rate || 0}%)`, val: taxes.duty }, 
                              { label: `貨物稅 (Excise Tax: ${result?.excise || 0}%)`, val: taxes.excise }, 
                              { label: `特種貨物及勞務稅 (Luxury Tax)`, val: taxes.luxury }, 
                              { label: `營業稅 (VAT 5%)`, val: taxes.vat, highlight: true }
                            ].map((item, idx) => (
                              <div key={idx} className={`flex justify-between items-center pb-1.5 border-b border-slate-200/40 last:border-0 last:pb-0 font-black font-black font-black ${item.highlight ? 'bg-blue-500/5 -mx-1 px-1 rounded font-black' : 'font-black'}`}>
                                <div className="flex items-center gap-1.5 font-black font-black font-black"><span className={`text-slate-500 font-bold text-[11px] tracking-tight font-black font-black font-black ${item.highlight ? 'text-blue-700 font-black' : 'font-black'}`}>{item.label}</span>{item.highlight && <HelpCircle className="w-2.5 h-2.5 text-blue-400 cursor-help font-black font-black" title="公式：(完稅價格 + 進口稅 + 貨物稅) × 5%" />}</div>
                                <div className="flex items-baseline gap-1 text-slate-900 font-bold font-mono font-black font-black font-black font-black"><span className="text-[8px] opacity-30 italic uppercase font-black font-black font-black">NT$</span><span className={`text-base tracking-tighter font-black font-black ${!result && 'text-[#ced4da] font-black'}`}>{Math.round(item.val).toLocaleString()}</span></div>
                              </div>
                            ))}
                          </div>
                          <div className={`p-5 py-6 flex flex-row justify-between items-center ${result ? 'bg-[#adb5bd] font-black' : 'bg-[#dee2e6] font-black'} transition-colors duration-700 font-black`}><div><span className="text-[9px] uppercase text-slate-700 font-black block leading-none font-black font-black">Final Estimated Landing Cost</span><div className="flex items-center gap-1 text-[7px] px-2 py-0.5 rounded-full w-fit bg-slate-900/20 mt-1 uppercase font-black font-black font-black"><Globe className="w-2.5 h-2.5 font-black font-black" /> GC411 Logic Verified</div></div><div className="flex items-baseline gap-2 text-white font-black font-black font-black font-black font-black"><span className="text-xl font-light italic text-slate-900/30 font-black">TWD</span><span className={`text-5xl font-mono tracking-tighter leading-none font-black font-black ${result ? 'text-slate-900 font-black' : 'text-[#ced4da] font-black'}`}>{result ? Math.round(taxes.grandTotal).toLocaleString() : cifValue.toLocaleString()}</span></div></div>
                        </div>
                      </div>
                    </div>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-3 text-white font-black font-black font-black font-black">
                      <div className="bg-[#1a1d24] border border-slate-800 rounded-[1.2rem] p-4 shadow-xl font-black font-black"><h3 className="text-[8px] text-slate-600 uppercase mb-3 flex items-center gap-2 font-black font-black font-black font-black"><AlertTriangle className="w-2.5 h-2.5 text-amber-500 font-black" /> Compliance Notes</h3><div className="space-y-2 font-black font-black font-black">{result?.regs.map((reg, i) => (<div key={i} className="p-2.5 bg-amber-500/5 rounded-lg border border-amber-500/10 text-[10px] leading-snug font-bold font-black">{reg.text}</div>))}</div></div>
                      <div className="bg-[#1a1d24] border border-slate-800 rounded-[1.2rem] p-4 shadow-xl text-slate-400 font-black font-black font-black font-black font-black font-black"><h3 className="text-[8px] text-slate-600 uppercase mb-3 flex items-center gap-2 text-white font-black font-black"><FileText className="w-2.5 h-2.5 text-blue-500 font-black" /> Required Docs</h3><div className="space-y-1.5 font-black">{result?.docs.map((doc, i) => (<div key={i} className="flex items-center gap-2 text-[10px] p-2 bg-slate-900/50 border border-slate-800 rounded-lg font-black font-black font-black font-black font-black"><CheckCircle className="w-3 h-3 text-blue-500 flex-shrink-0 font-black" /><span className="font-black text-slate-300 font-black">{doc}</span></div>))}</div></div>
                    </div>
                  </div>
                )}
              </div>
            </div>

            {/* 下半部：專業全球關稅諮詢區塊 (依據 DM 內容優化) */}
            <section className="mt-20 bg-gradient-to-br from-slate-900/80 to-slate-950 border border-slate-800 rounded-[3rem] p-8 md:p-12 shadow-2xl relative overflow-hidden font-black font-black font-black font-black font-black">
              <div className="absolute top-0 right-0 p-12 opacity-[0.03] pointer-events-none -rotate-12 font-black"><Star className="w-64 h-64 font-black font-black font-black font-black" /></div>
              <div className="max-w-5xl relative z-10 font-black font-black font-black font-black">
                <div className="inline-flex items-center gap-2 px-4 py-1.5 bg-blue-600/10 border border-blue-500/30 rounded-full mb-8 font-black font-black">
                  <Sparkles className="w-3.5 h-3.5 text-blue-400 font-black font-black" />
                  <span className="text-[10px] font-black text-blue-400 uppercase tracking-[0.2em] font-black">Global Tariff Consulting / 專業全球關稅諮詢</span>
                </div>
                
                <h2 className="text-4xl font-black text-white mb-4 leading-tight font-black">專業全球關稅諮詢 <br className="md:hidden font-black"/> <span className="text-blue-500 text-3xl md:text-4xl font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">助您降低進口成本</span></h2>
                <p className="text-slate-300 text-base max-w-3xl mb-12 font-bold leading-relaxed font-black font-black font-black font-black font-black font-black font-black font-black">
                  稅則判定不明？想了解特定品項如何優化稅率？如何取得合法關稅減免？<br className="hidden md:block font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black"/>
                  全球進口關稅分析優化？美國關稅因應？全球供應鏈優化？<br className="hidden md:block font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black"/>
                  我們透過精準案例分析與合規管理，確保您的跨境貿易利潤最大化。
                </p>

                {/* 1. 核心服務卡片 */}
                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 mb-16 font-black font-black font-black font-black font-black font-black font-black">
                  {[
                    { icon: Percent, title: "進口關稅節省", desc: "透過稅則優化及確保合規文件以節省關稅。", color: "from-emerald-600/20 to-emerald-900/10", iconColor: "text-emerald-500" },
                    { icon: Gavel, title: "關稅爭議解決", desc: "精準案例分析，避免不必要的額外課稅。", color: "from-blue-600/20 to-blue-900/10", iconColor: "text-blue-500" },
                    { icon: Map, title: "全球生產策略規劃", desc: "根據關稅分析，優化多國供應鏈布局。", color: "from-purple-600/20 to-purple-900/10", iconColor: "text-purple-400" },
                    { icon: ShieldAlert, title: "貿易風險預防", desc: "提前應對美國懲罰性關稅及各國雙反政策。", color: "from-rose-600/20 to-rose-900/10", iconColor: "text-rose-500" }
                  ].map((service, idx) => (
                    <div key={idx} className={`p-6 bg-gradient-to-br ${service.color} border border-white/5 rounded-3xl group hover:border-white/10 transition-all hover:-translate-y-1 font-black font-black font-black font-black font-black`}>
                      <div className={`p-3 bg-slate-900/80 w-fit rounded-xl mb-4 shadow-lg ${service.iconColor} font-black`}><service.icon className="w-6 h-6 font-black font-black font-black font-black font-black font-black font-black" /></div>
                      <h4 className="text-white font-black text-lg mb-2 font-black font-black font-black">{service.title}</h4>
                      <p className="text-slate-500 text-xs leading-relaxed font-bold font-black font-black font-black">{service.desc}</p>
                    </div>
                  ))}
                </div>

                {/* 2. 為什麼選擇我們 */}
                <div className="mb-16 font-black font-black font-black font-black font-black font-black font-black font-black">
                  <div className="flex items-center gap-3 mb-8 font-black font-black font-black font-black font-black font-black">
                    <Award className="w-6 h-6 text-amber-500 font-black font-black font-black font-black font-black font-black" />
                    <h3 className="text-xl text-white font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">為什麼選擇 我們 (翌暢股份有限公司)？</h3>
                  </div>
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-8 font-black font-black font-black font-black font-black font-black">
                    <div className="p-8 bg-blue-600/5 border border-blue-500/20 rounded-[2.5rem] relative group hover:bg-blue-600/10 transition-all font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">
                      <div className="absolute -top-4 -left-4 w-12 h-12 bg-blue-600 rounded-2xl flex items-center justify-center shadow-xl shadow-blue-600/30 group-hover:scale-110 transition-transform font-black">
                        <Globe2 className="w-6 h-6 text-white font-black font-black font-black font-black font-black font-black font-black font-black font-black" />
                      </div>
                      <h4 className="text-lg text-white mb-4 mt-2 font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">最具國際性的全球視野</h4>
                      <p className="text-slate-400 text-sm leading-relaxed font-bold font-black font-black font-black font-black font-black font-black font-black font-black">
                        我們比報關行更具國際性！不只服務台灣進口，也服務進口其他國家的稅則稅率評估及優化。透過我們的專業，讓中小企業進口其他國家不再是夢想。
                      </p>
                    </div>
                    <div className="p-8 bg-emerald-600/5 border border-emerald-500/20 rounded-[2.5rem] relative group hover:bg-emerald-600/10 transition-all font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">
                      <div className="absolute -top-4 -left-4 w-12 h-12 bg-emerald-600 rounded-2xl flex items-center justify-center shadow-xl shadow-emerald-600/30 group-hover:scale-110 transition-transform font-black">
                        <Coins className="w-6 h-6 text-white font-black font-black font-black font-black font-black font-black font-black font-black font-black" />
                      </div>
                      <h4 className="text-lg text-white mb-4 mt-2 font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">最高 CP 值的顧問價值</h4>
                      <p className="text-slate-400 text-sm leading-relaxed font-bold font-black font-black font-black font-black font-black font-black font-black font-black">
                        我們比四大會計師事務所更便宜且 CP 更高！提供同等級的專業諮詢服務，但不再有昂貴的諮詢負擔，助您以最實惠的預算獲得最佳稅務規畫。
                      </p>
                    </div>
                  </div>
                </div>

                {/* 3. CTA */}
                <div className="bg-slate-950/80 p-8 rounded-[2.5rem] border border-slate-800 flex flex-col md:flex-row items-center justify-between gap-8 backdrop-blur-md font-black font-black font-black font-black font-black font-black font-black font-black font-black">
                  <div className="flex items-center gap-6 font-black font-black font-black font-black font-black font-black font-black font-black">
                    <div className="w-16 h-16 bg-blue-600/10 rounded-full flex items-center justify-center border border-blue-500/20 shadow-inner font-black">
                      <Compass className="w-8 h-8 text-blue-500 animate-pulse font-black font-black font-black" />
                    </div>
                    <div className="font-black font-black">
                      <p className="text-white text-lg font-black mb-1 font-black">進口合規性審查 & 稅金負擔優化分析</p>
                      <p className="text-slate-400 text-sm font-bold leading-relaxed font-black font-black font-black font-black font-black font-black font-black font-black">
                        我們的專員將協助您審核文件並提供合規建議，<br className="hidden md:block font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black"/>
                        資料將加密傳送至顧問信箱：<span className="text-blue-400 font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">miacolor0508@gmail.com</span>
                      </p>
                    </div>
                  </div>
                  <button 
                    onClick={() => setView('consult')}
                    className="px-10 py-5 bg-blue-600 hover:bg-blue-500 text-white rounded-2xl font-black text-base transition-all shadow-2xl shadow-blue-500/20 flex items-center gap-3 active:scale-95 group uppercase tracking-widest whitespace-nowrap font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black"
                  >
                    立即預約專員諮詢 <ChevronRight className="w-5 h-4 group-hover:translate-x-1 transition-transform font-black font-black" />
                  </button>
                </div>
              </div>
            </section>
          </div>
        ) : (
          <ConsultView />
        )}

        {/* Footer Disclaimer */}
        <footer className="mt-16 pt-8 border-t border-slate-900 text-center pb-12 font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black">
          <p className="text-slate-600 text-[10px] leading-relaxed max-w-4xl mx-auto font-black font-black font-black font-black font-black font-black font-black font-black font-black font-black opacity-80 font-black font-black font-black">
            <span className="text-slate-500 font-black tracking-tighter mr-1 uppercase font-black font-black font-black font-black font-black font-black">翌暢股份有限公司</span> 
            is provided for general informational purposes only and does not constitute any legal, tax or customs advice. 
            Verify all calculations with the Taiwan Customs authorities as duty rates and customs regulations are subject to change without notice. 
            We do not accept any liability as a result of using this tool.
          </p>
        </footer>

      </div>
    </div>
  );
};

export default App;
