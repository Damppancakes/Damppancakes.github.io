<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Data Center Planner</title>
<link href="https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=JetBrains+Mono:wght@400;600;700&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.9/babel.min.js"></script>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #08080a; }
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useMemo } = React;

const SERVERS = {
  systemx: { name: "System X", color: "#D4A017", sizes: { "3U": { iops: 5000, cost: 400 }, "7U": { iops: 12000, cost: 1600 } } },
  risc: { name: "RISC", color: "#2255CC", sizes: { "3U": { iops: 5000, cost: 450 }, "7U": { iops: 12000, cost: 1750 } } },
  mainframe: { name: "Mainframe", color: "#CC2299", sizes: { "3U": { iops: 5000, cost: 850 }, "7U": { iops: 12000, cost: 2000 } } },
  gpu: { name: "GPU", color: "#22AA44", sizes: { "3U": { iops: 5000, cost: 550 }, "7U": { iops: 12000, cost: 2200 } } },
};

const SWITCH_OPTIONS = {
  rj45_16: { name: "16x 10Gbps RJ45", cost: 250, serverPorts: 16, needsSFP: false },
  sfp_4: { name: "4x SFP+/SFP28", cost: 400, serverPorts: 4, needsSFP: true },
  hybrid: { name: "4x QSFP+ 16x SFP+/SFP28", cost: 3500, serverPorts: 16, needsSFP: true },
  qsfp_32: { name: "32x QSFP+", cost: 3800, serverPorts: 32, needsSFP: true },
};

const SFP_OPTIONS = {
  none: { name: "N/A (RJ45 switch)", costPer: 0 },
  sfp_rj45: { name: "SFP+ RJ45 10Gbps", costPer: 50, speed: 10 },
  sfp_fiber: { name: "SFP+ Fiber 10Gbps", costPer: 70, speed: 10 },
  sfp28: { name: "SFP28 Fiber 25Gbps", costPer: 180, speed: 25 },
  qsfp: { name: "QSFP+ Fiber 40Gbps", costPer: 300, speed: 40 },
};

const PATCH_PANELS = {
  rj45: { name: "Patch Panel RJ45", cost: 250 },
  combo: { name: "Patch Panel Combo", cost: 450 },
  fiber: { name: "Patch Panel Fiber", cost: 450 },
};

const RACK_COST = 1250;
const CABLE_CAT6E = 500;
const CABLE_FIBER1 = 1000;
const SERVERS_PER_RACK_7U = 6;
const SERVERS_PER_RACK_3U = 14;
const MAX_SERVERS_PER_SUBNET = 30;

const CUSTOMERS = [
  { name: "Counterfeit Goods Distribution", rep: 90, reqs: { systemx: 120000, risc: 80000, mainframe: 90000 } },
  { name: "Madeep", rep: 110, reqs: { systemx: 140000, risc: 100000, mainframe: 110000 } },
  { name: "Flat Earth Servers", rep: 135, reqs: { systemx: 160000, risc: 120000, mainframe: 130000 } },
];

function calcApp(iopsNeeded, serverType, serverSize, switchType, sfpType, patchType) {
  const iopsPerServer = SERVERS[serverType].sizes[serverSize].iops;
  const costPerServer = SERVERS[serverType].sizes[serverSize].cost;
  const serversPerRack = serverSize === "7U" ? SERVERS_PER_RACK_7U : SERVERS_PER_RACK_3U;
  const iopsPerRack = serversPerRack * iopsPerServer;
  const serversNeeded = Math.ceil(iopsNeeded / iopsPerServer);
  const racksNeeded = Math.ceil(serversNeeded / serversPerRack);
  const actualIOPS = serversNeeded * iopsPerServer;
  const subnetsCapped = serversNeeded > MAX_SERVERS_PER_SUBNET;
  const lastRackServers = serversNeeded % serversPerRack || serversPerRack;
  const emptySlots = (racksNeeded * serversPerRack) - serversNeeded;

  const sw = SWITCH_OPTIONS[switchType];
  const sfp = SFP_OPTIONS[sfpType];
  const pp = PATCH_PANELS[patchType];

  const serverCost = serversNeeded * costPerServer;
  const rackCost = racksNeeded * RACK_COST;
  const switchCostTotal = racksNeeded * sw.cost;
  const patchCost = racksNeeded * pp.cost * 2;
  const sfpModuleCost = sw.needsSFP ? serversNeeded * sfp.costPer : 0;
  const cableCost = racksNeeded * CABLE_CAT6E;

  return {
    serverType, serverSize, iopsNeeded, iopsPerServer, iopsPerRack,
    serversNeeded, serversPerRack, actualIOPS, subnetsCapped,
    racksNeeded, lastRackServers, emptySlots,
    serverCost, rackCost, switchCostTotal, patchCost, sfpModuleCost, cableCost,
    appTotal: serverCost + rackCost + switchCostTotal + patchCost + sfpModuleCost + cableCost,
  };
}

function StatBlock({ label, value, sub, accent }) {
  return (
    <div style={{ padding: "14px 16px", background: "rgba(255,255,255,0.02)", border: "1px solid #ffffff0a", borderRadius: "6px" }}>
      <div style={{ fontSize: "10px", color: "#666", fontFamily: "'JetBrains Mono', monospace", textTransform: "uppercase", letterSpacing: "1.2px", marginBottom: "6px" }}>{label}</div>
      <div style={{ fontSize: "22px", fontWeight: 700, color: accent || "#e8e8e8", fontFamily: "'Space Mono', monospace", lineHeight: 1 }}>{value}</div>
      {sub && <div style={{ fontSize: "10px", color: "#555", marginTop: "4px" }}>{sub}</div>}
    </div>
  );
}

function Select({ label, value, onChange, options }) {
  return (
    <div>
      <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "6px", fontFamily: "'JetBrains Mono', monospace" }}>{label}</div>
      <select value={value} onChange={e => onChange(e.target.value)} style={{
        padding: "6px 12px", background: "#0e0e11", border: "1px solid #1a1a1f",
        borderRadius: "3px", color: "#ddd", fontFamily: "'JetBrains Mono', monospace",
        fontSize: "11px", outline: "none", cursor: "pointer", minWidth: "180px",
      }}>
        {options.map(([k, l]) => <option key={k} value={k}>{l}</option>)}
      </select>
    </div>
  );
}

function App() {
  const [mode, setMode] = useState("preset");
  const [selectedCustomer, setSelectedCustomer] = useState(0);
  const [customReqs, setCustomReqs] = useState({ systemx: 0, risc: 0, mainframe: 0, gpu: 0 });
  const [serverSize, setServerSize] = useState("7U");
  const [switchType, setSwitchType] = useState("rj45_16");
  const [sfpType, setSfpType] = useState("none");
  const [patchType, setPatchType] = useState("rj45");

  const needsSFP = SWITCH_OPTIONS[switchType].needsSFP;
  const reqs = mode === "preset" ? CUSTOMERS[selectedCustomer].reqs : customReqs;

  const results = useMemo(() => {
    const apps = [];
    let grandTotal = 0, totalServers = 0, totalRacks = 0, totalSFPs = 0, warnings = [];

    for (const [type, iops] of Object.entries(reqs)) {
      if (iops <= 0) continue;
      const effectiveSfp = needsSFP ? sfpType : "none";
      const r = calcApp(iops, type, serverSize, switchType, effectiveSfp, patchType);
      apps.push(r);
      grandTotal += r.appTotal;
      totalServers += r.serversNeeded;
      totalRacks += r.racksNeeded;
      if (needsSFP) totalSFPs += r.serversNeeded;
      if (r.subnetsCapped) warnings.push(`${SERVERS[type].name} needs ${r.serversNeeded} servers — exceeds /27 subnet cap of 30!`);
    }

    const fiberUplinks = apps.length;
    const fiberUplinkCost = fiberUplinks * CABLE_FIBER1;
    grandTotal += fiberUplinkCost;

    return { apps, grandTotal, totalServers, totalRacks, totalSFPs, warnings, fiberUplinks, fiberUplinkCost };
  }, [reqs, serverSize, switchType, sfpType, patchType, needsSFP]);

  return (
    <div style={{ minHeight: "100vh", background: "#08080a", color: "#e0e0e0", fontFamily: "'Space Mono', 'JetBrains Mono', 'Courier New', monospace" }}>
      <div style={{ borderBottom: "1px solid #151518", padding: "20px 28px", background: "linear-gradient(180deg, #0e0e11 0%, #08080a 100%)" }}>
        <div style={{ display: "flex", alignItems: "center", gap: "10px", marginBottom: "2px" }}>
          <div style={{ width: "6px", height: "6px", borderRadius: "50%", background: "#22ff66", boxShadow: "0 0 6px #22ff6666" }} />
          <span style={{ fontSize: "10px", color: "#22ff66", letterSpacing: "2.5px", textTransform: "uppercase", fontWeight: 700, fontFamily: "'JetBrains Mono', monospace" }}>SYSTEM ONLINE</span>
        </div>
        <h1 style={{ margin: "6px 0 0 0", fontSize: "22px", fontWeight: 700, color: "#fff", letterSpacing: "-0.3px" }}>Data Center Planner</h1>
        <p style={{ margin: "2px 0 0 0", fontSize: "11px", color: "#444" }}>Rack: Patch Panel → Switch → Patch Panel → 6× 12,000 IOPS Servers = 72,000 IOPS/rack</p>
      </div>

      <div style={{ padding: "20px 28px", maxWidth: "900px" }}>
        <div style={{ display: "flex", gap: "20px", marginBottom: "20px", flexWrap: "wrap", alignItems: "flex-end" }}>
          <div>
            <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "6px", fontFamily: "'JetBrains Mono', monospace" }}>Mode</div>
            <div style={{ display: "flex", gap: "4px" }}>
              {["preset", "custom"].map(m => (
                <button key={m} onClick={() => setMode(m)} style={{
                  padding: "6px 16px", background: mode === m ? "#1a1a1f" : "transparent",
                  border: `1px solid ${mode === m ? "#2a2a33" : "#151518"}`, borderRadius: "3px",
                  color: mode === m ? "#ddd" : "#444", fontFamily: "inherit", fontSize: "11px",
                  fontWeight: 600, cursor: "pointer", textTransform: "uppercase", letterSpacing: "0.8px",
                }}>{m === "preset" ? "Customers" : "Custom"}</button>
              ))}
            </div>
          </div>
          <div>
            <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "6px", fontFamily: "'JetBrains Mono', monospace" }}>Server Size</div>
            <div style={{ display: "flex", gap: "4px" }}>
              {["3U", "7U"].map(s => (
                <button key={s} onClick={() => setServerSize(s)} style={{
                  padding: "6px 16px", background: serverSize === s ? "#1a1a1f" : "transparent",
                  border: `1px solid ${serverSize === s ? "#2a2a33" : "#151518"}`, borderRadius: "3px",
                  color: serverSize === s ? "#ddd" : "#444", fontFamily: "inherit", fontSize: "11px", fontWeight: 600, cursor: "pointer",
                }}>{s} · {s === "3U" ? "5K" : "12K"} IOPS · {s === "3U" ? "14" : "6"}/rack</button>
              ))}
            </div>
          </div>
        </div>

        <div style={{ display: "flex", gap: "16px", marginBottom: "24px", flexWrap: "wrap", padding: "16px", background: "#0c0c0f", border: "1px solid #151518", borderRadius: "6px" }}>
          <Select label="Switch" value={switchType} onChange={v => { setSwitchType(v); if (!SWITCH_OPTIONS[v].needsSFP) setSfpType("none"); else if (sfpType === "none") setSfpType("sfp_rj45"); }}
            options={Object.entries(SWITCH_OPTIONS).map(([k, v]) => [k, `${v.name} — $${v.cost}`])} />
          <Select label="SFP Modules" value={needsSFP ? sfpType : "none"} onChange={v => setSfpType(v)}
            options={needsSFP ? Object.entries(SFP_OPTIONS).filter(([k]) => k !== "none").map(([k, v]) => [k, `${v.name} — $${v.costPer}/ea`]) : [["none", "Not needed (RJ45 switch)"]]} />
          <Select label="Patch Panels" value={patchType} onChange={setPatchType}
            options={Object.entries(PATCH_PANELS).map(([k, v]) => [k, `${v.name} — $${v.cost}`])} />
        </div>

        {mode === "preset" ? (
          <div style={{ marginBottom: "24px" }}>
            <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "8px", fontFamily: "'JetBrains Mono', monospace" }}>Customer</div>
            <div style={{ display: "flex", gap: "8px", flexWrap: "wrap" }}>
              {CUSTOMERS.map((c, i) => (
                <button key={i} onClick={() => setSelectedCustomer(i)} style={{
                  padding: "10px 16px", background: selectedCustomer === i ? "#13131a" : "transparent",
                  border: `1px solid ${selectedCustomer === i ? "#2a2a33" : "#151518"}`, borderRadius: "5px",
                  color: selectedCustomer === i ? "#e0e0e0" : "#555", fontFamily: "inherit", fontSize: "12px",
                  cursor: "pointer", textAlign: "left",
                }}>
                  <div style={{ fontWeight: 700, marginBottom: "2px" }}>{c.name}</div>
                  <div style={{ fontSize: "10px", color: "#444" }}>Rep {c.rep} · {Object.values(c.reqs).reduce((a,b)=>a+b,0).toLocaleString()} total IOPS</div>
                </button>
              ))}
            </div>
          </div>
        ) : (
          <div style={{ marginBottom: "24px" }}>
            <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "10px", fontFamily: "'JetBrains Mono', monospace" }}>IOPS per Server Type</div>
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "10px" }}>
              {Object.entries(SERVERS).map(([key, srv]) => (
                <div key={key} style={{ display: "flex", alignItems: "center", gap: "10px" }}>
                  <div style={{ width: "8px", height: "8px", borderRadius: "2px", background: srv.color, flexShrink: 0 }} />
                  <label style={{ fontSize: "11px", color: "#777", width: "72px", flexShrink: 0, fontFamily: "'JetBrains Mono', monospace" }}>{srv.name}</label>
                  <input type="number" min={0} step={1000} value={customReqs[key] || ""} placeholder="0"
                    onChange={e => setCustomReqs(prev => ({ ...prev, [key]: Math.max(0, parseInt(e.target.value) || 0) }))}
                    style={{ flex: 1, padding: "7px 10px", background: "#0e0e11", border: "1px solid #1a1a1f", borderRadius: "3px", color: "#e0e0e0", fontFamily: "inherit", fontSize: "13px", outline: "none" }}
                  />
                </div>
              ))}
            </div>
          </div>
        )}

        {results.warnings.length > 0 && (
          <div style={{ padding: "10px 14px", background: "#ff3c3c08", border: "1px solid #ff3c3c22", borderRadius: "5px", marginBottom: "18px" }}>
            {results.warnings.map((w, i) => (
              <div key={i} style={{ fontSize: "11px", color: "#ff6666", lineHeight: 1.5 }}>⚠ {w}</div>
            ))}
          </div>
        )}

        <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: "10px", marginBottom: "24px" }}>
          <StatBlock label="Total Servers" value={results.totalServers} sub={`${results.apps.length} server type${results.apps.length !== 1 ? "s" : ""}`} />
          <StatBlock label="Total Racks" value={results.totalRacks} sub="self-contained" />
          <StatBlock label="SFP Modules" value={results.totalSFPs} sub={results.totalSFPs > 0 ? SFP_OPTIONS[sfpType]?.name : "N/A"} />
          <StatBlock label="Total Cost" value={`$${results.grandTotal.toLocaleString()}`} accent="#22ff66" />
        </div>

        <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "10px", fontFamily: "'JetBrains Mono', monospace" }}>Per Server Type</div>
        <div style={{ display: "flex", flexDirection: "column", gap: "10px", marginBottom: "24px" }}>
          {results.apps.map((app, i) => (
            <div key={i} style={{ padding: "16px 18px", background: "#0d0d10", border: "1px solid #151518", borderRadius: "6px", borderLeft: `3px solid ${SERVERS[app.serverType].color}` }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "14px" }}>
                <div style={{ display: "flex", alignItems: "center", gap: "10px" }}>
                  <span style={{ fontWeight: 700, fontSize: "14px" }}>{SERVERS[app.serverType].name}</span>
                  <span style={{ fontSize: "10px", color: "#555", background: "#151518", padding: "2px 8px", borderRadius: "3px" }}>{app.serverSize}</span>
                </div>
                <span style={{ fontSize: "14px", fontWeight: 700, color: "#22ff66" }}>${app.appTotal.toLocaleString()}</span>
              </div>
              <div style={{ display: "grid", gridTemplateColumns: "repeat(6, 1fr)", gap: "12px", fontSize: "11px" }}>
                {[
                  ["Required IOPS", app.iopsNeeded.toLocaleString(), null],
                  ["Delivered IOPS", app.actualIOPS.toLocaleString(), app.actualIOPS >= app.iopsNeeded ? "#22ff66" : "#ff5555"],
                  ["Surplus", `+${(app.actualIOPS - app.iopsNeeded).toLocaleString()}`, app.actualIOPS > app.iopsNeeded ? "#D4A017" : "#555"],
                  ["Servers", `${app.serversNeeded} × ${app.iopsPerServer.toLocaleString()}`, app.subnetsCapped ? "#ff5555" : null],
                  ["Racks", `${app.racksNeeded} × ${app.iopsPerRack.toLocaleString()}`, null],
                  ["Empty Slots", app.emptySlots, app.emptySlots > 0 ? "#D4A017" : "#555"],
                ].map(([label, val, clr], j) => (
                  <div key={j}>
                    <div style={{ color: "#444", marginBottom: "3px", fontFamily: "'JetBrains Mono', monospace", fontSize: "9px", textTransform: "uppercase", letterSpacing: "0.5px" }}>{label}</div>
                    <div style={{ fontWeight: 700, color: clr || "#e0e0e0" }}>{val}</div>
                  </div>
                ))}
              </div>
              <div style={{ marginTop: "12px", display: "flex", gap: "6px", alignItems: "flex-end", flexWrap: "wrap" }}>
                {Array.from({ length: app.racksNeeded }).map((_, ri) => {
                  const isLast = ri === app.racksNeeded - 1;
                  const filled = isLast ? app.lastRackServers : app.serversPerRack;
                  return (
                    <div key={ri} style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: "3px" }}>
                      <div style={{ display: "flex", flexDirection: "column", gap: "1px", background: "#000", border: "1px solid #252528", borderRadius: "3px", padding: "2px", width: "42px" }}>
                        <div style={{ height: "3px", background: "#222", borderRadius: "1px" }} />
                        <div style={{ height: "4px", background: "#3a3a44", borderRadius: "1px" }} />
                        <div style={{ height: "3px", background: "#222", borderRadius: "1px" }} />
                        {Array.from({ length: app.serversPerRack }).map((_, si) => (
                          <div key={si} style={{
                            height: app.serverSize === "7U" ? "11px" : "5px",
                            background: si < filled ? SERVERS[app.serverType].color + "bb" : "#111",
                            borderRadius: "1px",
                            border: si >= filled ? "1px dashed #1a1a1a" : "none",
                          }} />
                        ))}
                      </div>
                      <span style={{ fontSize: "8px", color: "#444", fontFamily: "'JetBrains Mono', monospace" }}>
                        {(filled * app.iopsPerServer / 1000)}K
                      </span>
                    </div>
                  );
                })}
              </div>
              {app.subnetsCapped && (
                <div style={{ marginTop: "10px", fontSize: "10px", color: "#ff6666", background: "#ff3c3c08", padding: "5px 10px", borderRadius: "3px" }}>
                  ⚠ {app.serversNeeded} servers exceeds /27 limit of 30 — needs multiple subnets
                </div>
              )}
            </div>
          ))}
        </div>

        <div style={{ fontSize: "10px", color: "#555", textTransform: "uppercase", letterSpacing: "1px", marginBottom: "10px", fontFamily: "'JetBrains Mono', monospace" }}>Shopping List</div>
        <div style={{ padding: "16px 18px", background: "#0d0d10", border: "1px solid #151518", borderRadius: "6px", fontSize: "12px" }}>
          <table style={{ width: "100%", borderCollapse: "collapse" }}>
            <thead>
              <tr style={{ borderBottom: "1px solid #1a1a1f" }}>
                {["Item", "Qty", "Unit $", "Total $"].map((h, i) => (
                  <th key={h} style={{ textAlign: i === 0 ? "left" : "right", padding: "6px 0", color: "#444", fontSize: "9px", textTransform: "uppercase", letterSpacing: "1px", fontFamily: "'JetBrains Mono', monospace", fontWeight: 600 }}>{h}</th>
                ))}
              </tr>
            </thead>
            <tbody>
              {results.apps.map((app, i) => (
                <tr key={`srv-${i}`} style={{ borderBottom: "1px solid #111114" }}>
                  <td style={{ padding: "7px 0" }}>
                    <span style={{ display: "inline-block", width: "6px", height: "6px", borderRadius: "1px", background: SERVERS[app.serverType].color, marginRight: "8px", verticalAlign: "middle" }} />
                    {SERVERS[app.serverType].name} {app.serverSize}
                  </td>
                  <td style={{ textAlign: "right" }}>{app.serversNeeded}</td>
                  <td style={{ textAlign: "right", color: "#555" }}>{SERVERS[app.serverType].sizes[app.serverSize].cost.toLocaleString()}</td>
                  <td style={{ textAlign: "right", fontWeight: 700 }}>{app.serverCost.toLocaleString()}</td>
                </tr>
              ))}
              {(() => {
                const sw = SWITCH_OPTIONS[switchType];
                const sfp = SFP_OPTIONS[sfpType];
                const pp = PATCH_PANELS[patchType];
                const rows = [
                  ["Lanberg Racks", results.totalRacks, RACK_COST],
                  [sw.name + " Switches", results.totalRacks, sw.cost],
                  [pp.name + " (×2/rack)", results.totalRacks * 2, pp.cost],
                ];
                if (needsSFP && results.totalSFPs > 0) rows.push([sfp.name + " Modules", results.totalSFPs, sfp.costPer]);
                rows.push(["CAT6E Cables (per rack)", results.totalRacks, CABLE_CAT6E], ["Fiber 1-lane Uplinks", results.fiberUplinks, CABLE_FIBER1]);
                return rows.map(([name, qty, unit], i) => (
                  <tr key={`i-${i}`} style={{ borderBottom: "1px solid #111114" }}>
                    <td style={{ padding: "7px 0" }}>{name}</td>
                    <td style={{ textAlign: "right" }}>{qty}</td>
                    <td style={{ textAlign: "right", color: "#555" }}>{unit.toLocaleString()}</td>
                    <td style={{ textAlign: "right", fontWeight: 700 }}>{(qty * unit).toLocaleString()}</td>
                  </tr>
                ));
              })()}
              <tr style={{ borderTop: "2px solid #222" }}>
                <td colSpan={3} style={{ padding: "10px 0", fontWeight: 700, fontSize: "13px", color: "#fff" }}>GRAND TOTAL</td>
                <td style={{ textAlign: "right", padding: "10px 0", fontWeight: 700, fontSize: "16px", color: "#22ff66" }}>${results.grandTotal.toLocaleString()}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div style={{ marginTop: "16px", fontSize: "10px", color: "#333", lineHeight: 1.7, fontFamily: "'JetBrains Mono', monospace" }}>
          Per rack: Patch Panel (1U) → Switch (1U) → Patch Panel (1U) → {serverSize === "7U" ? "6" : "14"}× {serverSize} @ {serverSize === "7U" ? "12,000" : "5,000"} IOPS = {serverSize === "7U" ? "72,000" : "70,000"} IOPS/rack · /27 subnet = 30 usable IPs · SFP modules sold in boxes of 5
        </div>
      </div>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
</script>
</body>
</html>
