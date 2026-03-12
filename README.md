# Cash-moneyimport { useState, useEffect } from "react";

const formatINR = (amount) => {
  if (amount === "" || amount === null || amount === undefined) return "";
  const num = parseFloat(amount);
  if (isNaN(num)) return "₹0.00";
  return new Intl.NumberFormat("en-IN", {
    style: "currency",
    currency: "INR",
    minimumFractionDigits: 2,
  }).format(num);
};

const parseNum = (val) => parseFloat(val) || 0;

const STORAGE_KEY = "cashbook_data_v1";

export default function CashBook() {
  const [tab, setTab] = useState("cashbook");
  const [days, setDays] = useState([]);
  const [purchases, setPurchases] = useState([]);
  const [calcDisplay, setCalcDisplay] = useState("0");
  const [calcExpression, setCalcExpression] = useState("");
  const [calcHistory, setCalcHistory] = useState([]);
  const [newPurchase, setNewPurchase] = useState({ date: "", item: "", qty: "", rate: "", amount: "" });
  const [editingDay, setEditingDay] = useState(null);
  const [form, setForm] = useState({ date: "", openingBalance: "", received: "", out: "" });

  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) {
        const data = JSON.parse(saved);
        setDays(data.days || []);
        setPurchases(data.purchases || []);
      }
    } catch (e) {}
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify({ days, purchases }));
    } catch (e) {}
  }, [days, purchases]);

  const calcPress = (val) => {
    if (val === "C") { setCalcDisplay("0"); setCalcExpression(""); return; }
    if (val === "⌫") {
      setCalcDisplay(prev => prev.length > 1 ? prev.slice(0, -1) : "0");
      return;
    }
    if (val === "=") {
      try {
        const expr = calcExpression + calcDisplay;
        const result = Function('"use strict"; return (' + expr + ')')();
        const rounded = Math.round(result * 100) / 100;
        setCalcHistory(prev => [...prev.slice(-4), `${expr} = ${rounded}`]);
        setCalcDisplay(String(rounded));
        setCalcExpression("");
      } catch { setCalcDisplay("Err"); setCalcExpression(""); }
      return;
    }
    if (["+", "-", "×", "÷"].includes(val)) {
      const op = val === "×" ? "*" : val === "÷" ? "/" : val;
      setCalcExpression(prev => prev + calcDisplay + op);
      setCalcDisplay("0");
      return;
    }
    if (val === ".") {
      if (!calcDisplay.includes(".")) setCalcDisplay(prev => prev + ".");
      return;
    }
    if (val === "%") {
      setCalcDisplay(prev => String(parseFloat(prev) / 100));
      return;
    }
    setCalcDisplay(prev => prev === "0" ? val : prev + val);
  };

  const getClosingBalance = (day) => {
    return parseNum(day.openingBalance) + parseNum(day.received) - parseNum(day.out);
  };

  const addOrUpdateDay = () => {
    if (!form.date) return;
    const closing = parseNum(form.openingBalance) + parseNum(form.received) - parseNum(form.out);
    if (editingDay !== null) {
      const updated = days.map((d, i) => i === editingDay ? { ...form, closing } : d);
      setDays(updated);
      setEditingDay(null);
    } else {
      const lastClosing = days.length > 0 ? days[days.length - 1].closing : 0;
      setDays(prev => [...prev, { ...form, openingBalance: form.openingBalance || lastClosing, closing }]);
    }
    setForm({ date: "", openingBalance: "", received: "", out: "" });
  };

  const editDay = (i) => {
    setEditingDay(i);
    setForm(days[i]);
    setTab("cashbook");
  };

  const deleteDay = (i) => {
    setDays(prev => prev.filter((_, idx) => idx !== i));
  };

  const addPurchase = () => {
    if (!newPurchase.item) return;
    const amount = newPurchase.qty && newPurchase.rate
      ? parseNum(newPurchase.qty) * parseNum(newPurchase.rate)
      : parseNum(newPurchase.amount);
    setPurchases(prev => [...prev, { ...newPurchase, amount }]);
    setNewPurchase({ date: "", item: "", qty: "", rate: "", amount: "" });
  };

  const deletePurchase = (i) => setPurchases(prev => prev.filter((_, idx) => idx !== i));
  const totalPurchases = purchases.reduce((s, p) => s + parseNum(p.amount), 0);

  const calcButtons = [
    ["%", "C", "⌫", "÷"],
    ["7", "8", "9", "×"],
    ["4", "5", "6", "-"],
    ["1", "2", "3", "+"],
    [".", "0", "00", "="],
  ];

  const tabs = [
    { id: "cashbook", label: "📒 Cash Book" },
    { id: "purchase", label: "🛒 Purchase" },
    { id: "calculator", label: "🔢 Calculator" },
  ];

  return (
    <div style={{ fontFamily: "'Georgia', serif", background: "#1a1a2e", minHeight: "100vh", color: "#e8e0d0", padding: "12px" }}>
      <div style={{ textAlign: "center", marginBottom: "16px" }}>
        <div style={{ fontSize: "22px", fontWeight: "bold", color: "#f5c842", letterSpacing: "2px" }}>📔 CASH BOOK</div>
        <div style={{ fontSize: "11px", color: "#a0a0c0", letterSpacing: "3px" }}>DAILY ACCOUNT REGISTER • ₹</div>
      </div>

      <div style={{ display: "flex", gap: "6px", marginBottom: "16px", background: "#16213e", borderRadius: "10px", padding: "4px" }}>
        {tabs.map(t => (
          <button key={t.id} onClick={() => setTab(t.id)} style={{
            flex: 1, padding: "8px 4px", borderRadius: "8px", border: "none", cursor: "pointer", fontSize: "12px", fontWeight: "bold",
            background: tab === t.id ? "#f5c842" : "transparent",
            color: tab === t.id ? "#1a1a2e" : "#a0a0c0",
            transition: "all 0.2s"
          }}>{t.label}</button>
        ))}
      </div>

      {tab === "cashbook" && (
        <div>
          <div style={{ background: "#16213e", borderRadius: "12px", padding: "14px", marginBottom: "14px", border: "1px solid #2a2a4e" }}>
            <div style={{ fontSize: "13px", fontWeight: "bold", color: "#f5c842", marginBottom: "10px" }}>
              {editingDay !== null ? "✏️ Edit Entry" : "➕ New Entry"}
            </div>
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "8px" }}>
              {[
                { key: "date", label: "Date", type: "date" },
                { key: "openingBalance", label: "Opening Balance ₹", type: "number" },
                { key: "received", label: "Received ₹", type: "number" },
                { key: "out", label: "Out ₹", type: "number" },
              ].map(f => (
                <div key={f.key}>
                  <div style={{ fontSize: "10px", color: "#a0a0c0", marginBottom: "3px" }}>{f.label}</div>
                  <input type={f.type} value={form[f.key]} onChange={e => setForm(p => ({ ...p, [f.key]: e.target.value }))}
                    style={{ width: "100%", padding: "7px", borderRadius: "6px", border: "1px solid #2a2a4e", background: "#0f3460", color: "#e8e0d0", fontSize: "13px", boxSizing: "border-box" }} />
                </div>
              ))}
            </div>
            {form.openingBalance !== "" && (
              <div style={{ marginTop: "8px", padding: "8px", background: "#0f3460", borderRadius: "8px", fontSize: "12px" }}>
                <span style={{ color: "#a0a0c0" }}>Closing Balance: </span>
                <span style={{ color: "#4ade80", fontWeight: "bold" }}>
                  {formatINR(parseNum(form.openingBalance) + parseNum(form.received) - parseNum(form.out))}
                </span>
              </div>
            )}
            <div style={{ display: "flex", gap: "8px", marginTop: "10px" }}>
              <button onClick={addOrUpdateDay} style={{
                flex: 1, padding: "9px", borderRadius: "8px", border: "none", cursor: "pointer",
                background: "#f5c842", color: "#1a1a2e", fontWeight: "bold", fontSize: "13px"
              }}>{editingDay !== null ? "Update" : "Save Entry"}</button>
              {editingDay !== null && (
                <button onClick={() => { setEditingDay(null); setForm({ date: "", openingBalance: "", received: "", out: "" }); }}
                  style={{ padding: "9px 14px", borderRadius: "8px", border: "none", cursor: "pointer", background: "#2a2a4e", color: "#e8e0d0", fontSize: "13px" }}>Cancel</button>
              )}
            </div>
          </div>

          {days.length === 0 ? (
            <div style={{ textAlign: "center", color: "#a0a0c0", padding: "30px", fontSize: "13px" }}>No entries yet. Add your first entry above!</div>
          ) : (
            <div style={{ display: "flex", flexDirection: "column", gap: "8px" }}>
              {days.map((d, i) => (
                <div key={i} style={{ background: "#16213e", borderRadius: "10px", padding: "12px", border: "1px solid #2a2a4e" }}>
                  <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "8px" }}>
                    <div style={{ fontSize: "13px", fontWeight: "bold", color: "#f5c842" }}>📅 {d.date}</div>
                    <div style={{ display: "flex", gap: "6px" }}>
                      <button onClick={() => editDay(i)} style={{ padding: "3px 8px", borderRadius: "5px", border: "none", cursor: "pointer", background: "#0f3460", color: "#a0c4ff", fontSize: "11px" }}>Edit</button>
                      <button onClick={() => deleteDay(i)} style={{ padding: "3px 8px", borderRadius: "5px", border: "none", cursor: "pointer", background: "#3d0000", color: "#ff8080", fontSize: "11px" }}>Del</button>
                    </div>
                  </div>
                  <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "6px", fontSize: "12px" }}>
                    {[
                      { label: "Opening", val: d.openingBalance, color: "#a0c4ff" },
                      { label: "Received", val: d.received, color: "#4ade80" },
                      { label: "Out", val: d.out, color: "#f87171" },
                      { label: "Closing", val: d.closing, color: "#f5c842" },
                    ].map(r => (
                      <div key={r.label} style={{ background: "#0f3460", borderRadius: "6px", padding: "6px" }}>
                        <div style={{ color: "#a0a0c0", fontSize: "10px" }}>{r.label}</div>
                        <div style={{ color: r.color, fontWeight: "bold" }}>{formatINR(r.val)}</div>
                      </div>
                    ))}
                  </div>
                </div>
              ))}
              <div style={{ background: "#0f3460", borderRadius: "10px", padding: "12px", border: "1px solid #f5c842" }}>
                <div style={{ fontSize: "12px", fontWeight: "bold", color: "#f5c842", marginBottom: "8px" }}>📊 Overall Summary</div>
                <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "6px", fontSize: "12px" }}>
                  <div><span style={{ color: "#a0a0c0" }}>Total Received: </span><span style={{ color: "#4ade80", fontWeight: "bold" }}>{formatINR(days.reduce((s, d) => s + parseNum(d.received), 0))}</span></div>
                  <div><span style={{ color: "#a0a0c0" }}>Total Out: </span><span style={{ color: "#f87171", fontWeight: "bold" }}>{formatINR(days.reduce((s, d) => s + parseNum(d.out), 0))}</span></div>
                  <div style={{ gridColumn: "span 2" }}><span style={{ color: "#a0a0c0" }}>Latest Closing: </span><span style={{ color: "#f5c842", fontWeight: "bold" }}>{formatINR(days[days.length - 1]?.closing)}</span></div>
                </div>
              </div>
            </div>
          )}
        </div>
      )}

      {tab === "purchase" && (
        <div>
          <div style={{ background: "#16213e", borderRadius: "12px", padding: "14px", marginBottom: "14px", border: "1px solid #2a2a4e" }}>
            <div style={{ fontSize: "13px", fontWeight: "bold", color: "#f5c842", marginBottom: "10px" }}>➕ Add Purchase</div>
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "8px" }}>
              {[
                { key: "date", label: "Date", type: "date" },
                { key: "item", label: "Item Name", type: "text" },
                { key: "qty", label: "Qty", type: "number" },
                { key: "rate", label: "Rate ₹", type: "number" },
                { key: "amount", label: "Amount ₹ (or auto)", type: "number" },
              ].map(f => (
                <div key={f.key} style={{ gridColumn: f.key === "item" ? "span 2" : "span 1" }}>
                  <div style={{ fontSize: "10px", color: "#a0a0c0", marginBottom: "3px" }}>{f.label}</div>
                  <input type={f.type} value={newPurchase[f.key]}
                    onChange={e => setNewPurchase(p => ({ ...p, [f.key]: e.target.value }))}
                    placeholder={f.key === "amount" ? "Auto if qty×rate" : ""}
                    style={{ width: "100%", padding: "7px", borderRadius: "6px", border: "1px solid #2a2a4e", background: "#0f3460", color: "#e8e0d0", fontSize: "13px", boxSizing: "border-box" }} />
                </div>
              ))}
            </div>
            {newPurchase.qty && newPurchase.rate && (
              <div style={{ marginTop: "8px", fontSize: "12px", color: "#4ade80" }}>
                Auto Amount: {formatINR(parseNum(newPurchase.qty) * parseNum(newPurchase.rate))}
              </div>
            )}
            <button onClick={addPurchase} style={{
              width: "100%", marginTop: "10px", padding: "9px", borderRadius: "8px", border: "none", cursor: "pointer",
              background: "#f5c842", color: "#1a1a2e", fontWeight: "bold", fontSize: "13px"
            }}>Save Purchase</button>
          </div>
          {purchases.length === 0 ? (
            <div style={{ textAlign: "center", color: "#a0a0c0", padding: "30px", fontSize: "13px" }}>No purchases yet!</div>
          ) : (
            <div>
              <div style={{ display: "flex", flexDirection: "column", gap: "8px" }}>
                {purchases.map((p, i) => (
                  <div key={i} style={{ background: "#16213e", borderRadius: "10px", padding: "10px", border: "1px solid #2a2a4e" }}>
                    <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                      <div>
                        <div style={{ fontSize: "13px", fontWeight: "bold", color: "#e8e0d0" }}>{p.item}</div>
                        <div style={{ fontSize: "11px", color: "#a0a0c0" }}>{p.date} {p.qty && p.rate && `| ${p.qty} × ₹${p.rate}`}</div>
                      </div>
                      <div style={{ display: "flex", alignItems: "center", gap: "8px" }}>
                        <div style={{ fontSize: "14px", fontWeight: "bold", color: "#f5c842" }}>{formatINR(p.amount)}</div>
                        <button onClick={() => deletePurchase(i)} style={{ padding: "3px 8px", borderRadius: "5px", border: "none", cursor: "pointer", background: "#3d0000", color: "#ff8080", fontSize: "11px" }}>Del</button>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
              <div style={{ marginTop: "10px", background: "#0f3460", borderRadius: "10px", padding: "12px", border: "1px solid #f5c842", textAlign: "right" }}>
                <span style={{ color: "#a0a0c0", fontSize: "13px" }}>Total Purchases: </span>
                <span style={{ color: "#f5c842", fontWeight: "bold", fontSize: "16px" }}>{formatINR(totalPurchases)}</span>
              </div>
            </div>
          )}
        </div>
      )}

      {tab === "calculator" && (
        <div>
          <div style={{ background: "#16213e", borderRadius: "12px", padding: "14px", border: "1px solid #2a2a4e" }}>
            <div style={{ minHeight: "50px", marginBottom: "8px", padding: "6px", background: "#0f3460", borderRadius: "8px" }}>
              {calcHistory.map((h, i) => (
                <div key={i} style={{ fontSize: "11px", color: "#a0a0c0", textAlign: "right" }}>{h}</div>
              ))}
            </div>
            <div style={{ textAlign: "right", fontSize: "12px", color: "#a0a0c0", minHeight: "18px", marginBottom: "2px" }}>{calcExpression}</div>
            <div style={{ textAlign: "right", fontSize: "28px", fontWeight: "bold", color: "#f5c842", padding: "8px", background: "#0f3460", borderRadius: "8px", marginBottom: "12px" }}>
              {calcDisplay.startsWith("Err") ? "Error" : Number(calcDisplay).toLocaleString("en-IN")}
            </div>
            {calcButtons.map((row, ri) => (
              <div key={ri} style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: "8px", marginBottom: "8px" }}>
                {row.map(btn => (
                  <button key={btn} onClick={() => calcPress(btn)} style={{
                    padding: "14px", borderRadius: "10px", border: "none", cursor: "pointer", fontSize: "16px", fontWeight: "bold",
                    background: btn === "=" ? "#f5c842" : ["+", "-", "×", "÷"].includes(btn) ? "#0f3460" : btn === "C" ? "#3d0000" : "#1e3a5f",
                    color: btn === "=" ? "#1a1a2e" : ["+", "-", "×", "÷"].includes(btn) ? "#f5c842" : btn === "C" ? "#ff8080" : "#e8e0d0",
                  }}>{btn}</button>
                ))}
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
