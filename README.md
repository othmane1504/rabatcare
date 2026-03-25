import { useState, useEffect, useCallback, useRef } from "react";

const THEME = {
  primary: "#0F2B3C",
  accent: "#1A8FB4",
  accentLight: "#E8F6FA",
  success: "#1B8C5A",
  warning: "#D4850A",
  danger: "#C43B3B",
  bg: "#F3F5F7",
  card: "#FFFFFF",
  text: "#1C2B36",
  textLight: "#6B7D8A",
  border: "#DFE4E8",
};

const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
const fmt = (d) => d ? new Date(d).toLocaleDateString("fr-FR") : "—";
const fmtLong = (d) => d ? new Date(d).toLocaleDateString("fr-FR", { day: "numeric", month: "long", year: "numeric" }) : "—";
const age = (bd) => { if (!bd) return ""; const a = Math.floor((Date.now() - new Date(bd).getTime()) / 31557600000); return `${a} ans`; };

const INIT_DOCTORS = [
  { id: "doc_martin", name: "Dr. Sarah Martin", email: "martin@clinique.fr", password: "medecin123", specialty: "Médecine Générale", phone: "06 12 34 56 78" },
  { id: "doc_dupont", name: "Dr. Pierre Dupont", email: "dupont@clinique.fr", password: "medecin123", specialty: "Cardiologie", phone: "06 98 76 54 32" },
];

const PROGNOSIS_OPTS = ["Stable", "Réservé", "Engagé"];
const PROGNOSIS_COLORS = { Stable: THEME.success, Réservé: THEME.warning, Engagé: THEME.danger };

/* ===== STORAGE ===== */
async function loadStore() {
  try {
    const r = await window.storage.get("med-app-v2");
    return r ? JSON.parse(r.value) : null;
  } catch { return null; }
}
async function saveStore(d) {
  try { await window.storage.set("med-app-v2", JSON.stringify(d)); } catch (e) { console.error("Save error", e); }
}

/* ===== ICONS (inline SVG) ===== */
const Icon = ({ d, size = 20, color = "currentColor", ...p }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke={color} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...p}>{d}</svg>
);
const Icons = {
  user: <><path d="M20 21v-2a4 4 0 00-4-4H8a4 4 0 00-4 4v2"/><circle cx="12" cy="7" r="4"/></>,
  users: <><path d="M17 21v-2a4 4 0 00-4-4H5a4 4 0 00-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 00-3-3.87"/><path d="M16 3.13a4 4 0 010 7.75"/></>,
  calendar: <><rect x="3" y="4" width="18" height="18" rx="2"/><line x1="16" y1="2" x2="16" y2="6"/><line x1="8" y1="2" x2="8" y2="6"/><line x1="3" y1="10" x2="21" y2="10"/></>,
  file: <><path d="M14 2H6a2 2 0 00-2 2v16a2 2 0 002 2h12a2 2 0 002-2V8z"/><polyline points="14,2 14,8 20,8"/></>,
  plus: <><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></>,
  search: <><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></>,
  back: <><polyline points="15,18 9,12 15,6"/></>,
  logout: <><path d="M9 21H5a2 2 0 01-2-2V5a2 2 0 012-2h4"/><polyline points="16,17 21,12 16,7"/><line x1="21" y1="12" x2="9" y2="12"/></>,
  heart: <><path d="M20.84 4.61a5.5 5.5 0 00-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 00-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 000-7.78z"/></>,
  clipboard: <><path d="M16 4h2a2 2 0 012 2v14a2 2 0 01-2 2H6a2 2 0 01-2-2V6a2 2 0 012-2h2"/><rect x="8" y="2" width="8" height="4" rx="1"/></>,
  activity: <><polyline points="22,12 18,12 15,21 9,3 6,12 2,12"/></>,
  edit: <><path d="M11 4H4a2 2 0 00-2 2v14a2 2 0 002 2h14a2 2 0 002-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 013 3L12 15l-4 1 1-4 9.5-9.5z"/></>,
  trash: <><polyline points="3,6 5,6 21,6"/><path d="M19 6v14a2 2 0 01-2 2H7a2 2 0 01-2-2V6m3 0V4a2 2 0 012-2h4a2 2 0 012 2v2"/></>,
  pdf: <><path d="M14 2H6a2 2 0 00-2 2v16a2 2 0 002 2h12a2 2 0 002-2V8z"/><polyline points="14,2 14,8 20,8"/><line x1="9" y1="15" x2="15" y2="15"/></>,
  shield: <><path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/></>,
  home: <><path d="M3 9l9-7 9 7v11a2 2 0 01-2 2H5a2 2 0 01-2-2z"/><polyline points="9,22 9,12 15,12 15,22"/></>,
  alert: <><path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></>,
  eye: <><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></>,
};

/* ===== UI COMPONENTS ===== */
const globalStyles = `
  @import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600;700&family=Lora:ital,wght@0,500;0,600;1,400&display=swap');
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: 'Outfit', sans-serif; background: ${THEME.bg}; color: ${THEME.text}; }
  input, textarea, select { font-family: 'Outfit', sans-serif; }
  ::-webkit-scrollbar { width: 6px; }
  ::-webkit-scrollbar-thumb { background: ${THEME.border}; border-radius: 3px; }
  @keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
  @keyframes slideIn { from { opacity: 0; transform: translateX(-12px); } to { opacity: 1; transform: translateX(0); } }
  @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.6; } }
  .fade-in { animation: fadeIn 0.35s ease-out; }
  .slide-in { animation: slideIn 0.3s ease-out; }
  @media print {
    .no-print { display: none !important; }
    body { background: white; }
  }
`;

function Btn({ children, variant = "primary", size = "md", icon, onClick, disabled, style, ...p }) {
  const base = { display: "inline-flex", alignItems: "center", gap: 6, border: "none", borderRadius: 10, cursor: disabled ? "default" : "pointer", fontFamily: "inherit", fontWeight: 500, transition: "all 0.2s", opacity: disabled ? 0.5 : 1 };
  const sizes = { sm: { padding: "6px 12px", fontSize: 13 }, md: { padding: "10px 18px", fontSize: 14 }, lg: { padding: "14px 24px", fontSize: 16 } };
  const variants = {
    primary: { background: THEME.accent, color: "#fff" },
    secondary: { background: THEME.accentLight, color: THEME.accent },
    danger: { background: "#FDE8E8", color: THEME.danger },
    ghost: { background: "transparent", color: THEME.textLight },
    outline: { background: "transparent", color: THEME.text, border: `1.5px solid ${THEME.border}` },
    success: { background: THEME.success, color: "#fff" },
  };
  return <button onClick={onClick} disabled={disabled} style={{ ...base, ...sizes[size], ...variants[variant], ...style }} {...p}>{icon && <Icon d={icon} size={size === "sm" ? 14 : 18} />}{children}</button>;
}

function Card({ children, style, className = "", onClick, ...p }) {
  return <div className={`fade-in ${className}`} onClick={onClick} style={{ background: THEME.card, borderRadius: 14, border: `1px solid ${THEME.border}`, padding: 20, ...style }} {...p}>{children}</div>;
}

function Input({ label, value, onChange, type = "text", required, rows, options, placeholder, style: s }) {
  const base = { width: "100%", padding: "10px 14px", borderRadius: 10, border: `1.5px solid ${THEME.border}`, fontSize: 14, fontFamily: "inherit", background: "#FAFBFC", transition: "border 0.2s", outline: "none" };
  return (
    <div style={{ marginBottom: 14, ...s }}>
      {label && <label style={{ display: "block", marginBottom: 5, fontSize: 13, fontWeight: 500, color: THEME.textLight }}>{label}{required && <span style={{ color: THEME.danger }}> *</span>}</label>}
      {options ? (
        <select value={value} onChange={e => onChange(e.target.value)} style={{ ...base, cursor: "pointer" }}>
          {options.map(o => <option key={o.value ?? o} value={o.value ?? o}>{o.label ?? o}</option>)}
        </select>
      ) : rows ? (
        <textarea value={value} onChange={e => onChange(e.target.value)} rows={rows} placeholder={placeholder} style={{ ...base, resize: "vertical", minHeight: 60 }} onFocus={e => e.target.style.borderColor = THEME.accent} onBlur={e => e.target.style.borderColor = THEME.border} />
      ) : (
        <input type={type} value={value} onChange={e => onChange(e.target.value)} required={required} placeholder={placeholder} style={base} onFocus={e => e.target.style.borderColor = THEME.accent} onBlur={e => e.target.style.borderColor = THEME.border} />
      )}
    </div>
  );
}

function Badge({ children, color = THEME.accent }) {
  return <span style={{ display: "inline-block", padding: "3px 10px", borderRadius: 20, fontSize: 11, fontWeight: 600, background: color + "18", color }}>{children}</span>;
}

function Empty({ icon, text }) {
  return <div style={{ textAlign: "center", padding: 48, color: THEME.textLight }}><Icon d={icon} size={40} color={THEME.border} style={{ display: "block", margin: "0 auto 12px" }} /><p style={{ fontSize: 14 }}>{text}</p></div>;
}

function Confirm({ message, onYes, onNo }) {
  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,0.4)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 1000, padding: 20 }}>
      <Card style={{ maxWidth: 360, textAlign: "center" }}>
        <Icon d={Icons.alert} size={36} color={THEME.warning} style={{ margin: "0 auto 12px", display: "block" }} />
        <p style={{ marginBottom: 20, fontSize: 15, lineHeight: 1.5 }}>{message}</p>
        <div style={{ display: "flex", gap: 10, justifyContent: "center" }}>
          <Btn variant="outline" onClick={onNo}>Annuler</Btn>
          <Btn variant="danger" onClick={onYes}>Confirmer</Btn>
        </div>
      </Card>
    </div>
  );
}

/* ===== PDF / REPORT GENERATION ===== */
function generateReportHTML(doctor, patient, consultation) {
  const isAdv = consultation.isAdvanced;
  return `<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Rapport - ${patient.lastName} ${patient.firstName}</title>
<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600&family=Lora:ital,wght@0,500;1,400&display=swap');
*{margin:0;padding:0;box-sizing:border-box}
body{font-family:'Outfit',sans-serif;color:#1C2B36;padding:40px;max-width:800px;margin:0 auto;line-height:1.6}
.header{border-bottom:3px solid #1A8FB4;padding-bottom:20px;margin-bottom:30px;display:flex;justify-content:space-between;align-items:flex-start}
.doc-info h1{font-family:'Lora',serif;font-size:22px;color:#0F2B3C}
.doc-info p{font-size:13px;color:#6B7D8A}
.report-date{text-align:right;font-size:13px;color:#6B7D8A}
.section{margin-bottom:24px}
.section h2{font-size:14px;text-transform:uppercase;letter-spacing:1.5px;color:#1A8FB4;margin-bottom:10px;padding-bottom:6px;border-bottom:1px solid #E8F0F4}
.section p,.section div{font-size:14px;white-space:pre-wrap}
.patient-box{background:#F3F7FA;padding:16px 20px;border-radius:10px;margin-bottom:24px;display:grid;grid-template-columns:1fr 1fr;gap:8px}
.patient-box .label{font-size:11px;text-transform:uppercase;letter-spacing:1px;color:#6B7D8A}
.patient-box .value{font-size:14px;font-weight:500}
.prognosis{display:inline-block;padding:4px 14px;border-radius:20px;font-weight:600;font-size:13px}
.signature{margin-top:50px;text-align:right;padding-top:20px;border-top:1px solid #DFE4E8}
.signature p{font-size:13px;color:#6B7D8A}
.signature .name{font-family:'Lora',serif;font-size:18px;color:#0F2B3C;margin-top:8px}
.footer{margin-top:40px;text-align:center;font-size:11px;color:#9BA8B2;border-top:1px solid #EEE;padding-top:12px}
@media print{body{padding:20px}@page{margin:1.5cm}}
</style></head><body>
<div class="header">
<div class="doc-info"><h1>${doctor.name}</h1><p>${doctor.specialty}</p>${doctor.phone ? `<p>${doctor.phone}</p>` : ""}</div>
<div class="report-date"><p>Date du rapport</p><p style="font-weight:600;color:#0F2B3C">${fmtLong(new Date())}</p></div>
</div>
<div class="patient-box">
<div><div class="label">Patient</div><div class="value">${patient.lastName} ${patient.firstName}</div></div>
<div><div class="label">Date de naissance</div><div class="value">${fmt(patient.birthDate)} (${age(patient.birthDate)})</div></div>
${patient.phone ? `<div><div class="label">Téléphone</div><div class="value">${patient.phone}</div></div>` : ""}
<div><div class="label">Date consultation</div><div class="value">${fmtLong(consultation.date)}</div></div>
</div>
<div class="section"><h2>Motif de consultation</h2><p>${consultation.motif || "—"}</p></div>
<div class="section"><h2>Histoire de la maladie</h2><p>${consultation.history || "—"}</p></div>
${isAdv && consultation.detailedHistory ? `<div class="section"><h2>Histoire détaillée</h2><p>${consultation.detailedHistory}</p></div>` : ""}
<div class="section"><h2>Antécédents</h2><p>${consultation.antecedents || "—"}</p></div>
<div class="section"><h2>Examen clinique</h2><p>${consultation.exam || "—"}</p></div>
${isAdv && consultation.complementaryExams ? `<div class="section"><h2>Examens complémentaires</h2><p>${consultation.complementaryExams}</p></div>` : ""}
<div class="section"><h2>Diagnostic</h2><p>${consultation.diagnostic || "—"}</p></div>
${isAdv && consultation.synthesis ? `<div class="section"><h2>Synthèse clinique</h2><p>${consultation.synthesis}</p></div>` : ""}
${isAdv && consultation.prognosis ? `<div class="section"><h2>Pronostic</h2><span class="prognosis" style="background:${(PROGNOSIS_COLORS[consultation.prognosis] || THEME.accent)}18;color:${PROGNOSIS_COLORS[consultation.prognosis] || THEME.accent}">${consultation.prognosis}</span></div>` : ""}
<div class="section"><h2>Traitement</h2><p>${consultation.treatment || "—"}</p></div>
${isAdv && consultation.conduct ? `<div class="section"><h2>Conduite à tenir</h2><p>${consultation.conduct}</p></div>` : ""}
<div class="signature"><p>Le médecin traitant,</p><div class="name">${doctor.name}</div><p>${doctor.specialty}</p></div>
<div class="footer">Document confidentiel — Usage médical uniquement</div>
</body></html>`;
}

/* ===== MAIN APPLICATION ===== */
export default function MedApp() {
  const [data, setData] = useState(null);
  const [user, setUser] = useState(null);
  const [view, setView] = useState("login");
  const [selected, setSelected] = useState({ patient: null, consultation: null });
  const [search, setSearch] = useState("");
  const [loginForm, setLoginForm] = useState({ email: "", password: "", error: "" });
  const [confirm, setConfirm] = useState(null);
  const [loading, setLoading] = useState(true);

  // Initialize
  useEffect(() => {
    (async () => {
      let d = await loadStore();
      if (!d) { d = { ...defaultData }; await saveStore(d); }
      // Ensure doctors exist
      if (!d.doctors || d.doctors.length === 0) { d.doctors = INIT_DOCTORS; await saveStore(d); }
      setData(d);
      setLoading(false);
    })();
  }, []);

  const persist = useCallback(async (newData) => {
    setData(newData);
    await saveStore(newData);
  }, []);

  // Auth
  const login = () => {
    const doc = data.doctors.find(d => d.email === loginForm.email.trim().toLowerCase() && d.password === loginForm.password);
    if (doc) { setUser(doc); setView("dashboard"); setLoginForm({ email: "", password: "", error: "" }); }
    else { setLoginForm(f => ({ ...f, error: "Identifiants incorrects" })); }
  };
  const logout = () => { setUser(null); setView("login"); setSelected({ patient: null, consultation: null }); };

  // Data helpers
  const myPatients = data ? data.patients.filter(p => p.doctorId === user?.id) : [];
  const myConsultations = data ? data.consultations.filter(c => c.doctorId === user?.id) : [];
  const patientConsultations = (pid) => myConsultations.filter(c => c.patientId === pid).sort((a, b) => new Date(b.date) - new Date(a.date));
  const filteredPatients = myPatients.filter(p => {
    const s = search.toLowerCase();
    return !s || `${p.firstName} ${p.lastName}`.toLowerCase().includes(s) || `${p.lastName} ${p.firstName}`.toLowerCase().includes(s);
  }).sort((a, b) => a.lastName.localeCompare(b.lastName));
  const recentConsultations = myConsultations.sort((a, b) => new Date(b.date) - new Date(a.date)).slice(0, 5);

  // CRUD operations
  const savePatient = async (patient) => {
    const nd = { ...data };
    if (patient.id) {
      nd.patients = nd.patients.map(p => p.id === patient.id ? patient : p);
    } else {
      nd.patients = [...nd.patients, { ...patient, id: uid(), doctorId: user.id, createdAt: new Date().toISOString() }];
    }
    await persist(nd);
  };
  const deletePatient = async (pid) => {
    const nd = { ...data };
    nd.patients = nd.patients.filter(p => p.id !== pid);
    nd.consultations = nd.consultations.filter(c => c.patientId !== pid);
    await persist(nd);
    nav("patients");
  };
  const saveConsultation = async (consultation) => {
    const nd = { ...data };
    if (consultation.id) {
      nd.consultations = nd.consultations.map(c => c.id === consultation.id ? consultation : c);
    } else {
      nd.consultations = [...nd.consultations, { ...consultation, id: uid(), doctorId: user.id, createdAt: new Date().toISOString() }];
    }
    await persist(nd);
  };
  const deleteConsultation = async (cid) => {
    const nd = { ...data };
    nd.consultations = nd.consultations.filter(c => c.id !== cid);
    await persist(nd);
    setSelected(s => ({ ...s, consultation: null }));
    setView("patient-detail");
  };

  const nav = (v, opts) => {
    setView(v);
    if (opts?.patient !== undefined) setSelected(s => ({ ...s, patient: opts.patient }));
    if (opts?.consultation !== undefined) setSelected(s => ({ ...s, consultation: opts.consultation }));
  };

  const openReport = (consult, patient) => {
    const html = generateReportHTML(user, patient || myPatients.find(p => p.id === consult.patientId), consult);
    const blob = new Blob([html], { type: "text/html" });
    const url = URL.createObjectURL(blob);
    window.open(url, "_blank");
  };

  if (loading) return (
    <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", background: THEME.bg }}>
      <div style={{ textAlign: "center" }}><div style={{ width: 40, height: 40, border: `3px solid ${THEME.border}`, borderTopColor: THEME.accent, borderRadius: "50%", animation: "pulse 1s infinite", margin: "0 auto 12px" }} /><p style={{ color: THEME.textLight, fontSize: 14 }}>Chargement...</p></div>
    </div>
  );

  /* ===== LOGIN SCREEN ===== */
  if (view === "login") return (
    <>
      <style>{globalStyles}</style>
      <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", background: `linear-gradient(135deg, ${THEME.primary} 0%, #163548 50%, ${THEME.accent} 100%)`, padding: 20 }}>
        <div className="fade-in" style={{ width: "100%", maxWidth: 380 }}>
          <div style={{ textAlign: "center", marginBottom: 32 }}>
            <div style={{ width: 64, height: 64, borderRadius: 16, background: "rgba(255,255,255,0.12)", display: "flex", alignItems: "center", justifyContent: "center", margin: "0 auto 16px", backdropFilter: "blur(8px)" }}>
              <Icon d={Icons.shield} size={32} color="#fff" />
            </div>
            <h1 style={{ fontFamily: "'Lora', serif", fontSize: 26, color: "#fff", fontWeight: 600 }}>MédiConsult</h1>
            <p style={{ color: "rgba(255,255,255,0.6)", fontSize: 13, marginTop: 4 }}>Gestion sécurisée des consultations</p>
          </div>
          <Card style={{ padding: 28 }}>
            <div style={{ marginBottom: 20 }}>
              <Input label="Adresse email" value={loginForm.email} onChange={v => setLoginForm(f => ({ ...f, email: v, error: "" }))} type="email" placeholder="docteur@clinique.fr" />
              <Input label="Mot de passe" value={loginForm.password} onChange={v => setLoginForm(f => ({ ...f, password: v, error: "" }))} type="password" placeholder="••••••••" />
            </div>
            {loginForm.error && <p style={{ color: THEME.danger, fontSize: 13, marginBottom: 12, textAlign: "center" }}>{loginForm.error}</p>}
            <Btn size="lg" onClick={login} style={{ width: "100%", justifyContent: "center", borderRadius: 12, fontWeight: 600 }}>Se connecter</Btn>
            <p style={{ textAlign: "center", marginTop: 16, fontSize: 11, color: THEME.textLight }}>Accès réservé aux médecins autorisés</p>
          </Card>
          <p style={{ textAlign: "center", marginTop: 16, fontSize: 11, color: "rgba(255,255,255,0.35)" }}>Démo : martin@clinique.fr / medecin123</p>
        </div>
      </div>
    </>
  );

  /* ===== HEADER ===== */
  const Header = ({ title, backTo, backLabel, actions }) => (
    <div className="no-print" style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 20, gap: 12, flexWrap: "wrap" }}>
      <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
        {backTo && <Btn variant="ghost" size="sm" icon={Icons.back} onClick={() => nav(backTo)} style={{ padding: 6 }} />}
        <h2 style={{ fontFamily: "'Lora', serif", fontSize: 20, fontWeight: 600, color: THEME.primary }}>{title}</h2>
      </div>
      {actions && <div style={{ display: "flex", gap: 8 }}>{actions}</div>}
    </div>
  );

  /* ===== STAT CARD ===== */
  const StatCard = ({ icon, label, value, color = THEME.accent }) => (
    <Card style={{ flex: 1, minWidth: 130, padding: 16, cursor: "default" }}>
      <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
        <div style={{ width: 36, height: 36, borderRadius: 10, background: color + "14", display: "flex", alignItems: "center", justifyContent: "center" }}>
          <Icon d={icon} size={18} color={color} />
        </div>
        <div>
          <div style={{ fontSize: 22, fontWeight: 700, color: THEME.primary }}>{value}</div>
          <div style={{ fontSize: 11, color: THEME.textLight, textTransform: "uppercase", letterSpacing: 0.5 }}>{label}</div>
        </div>
      </div>
    </Card>
  );

  /* ===== PATIENT FORM ===== */
  const PatientForm = ({ patient: initial, onSave, onCancel }) => {
    const [f, setF] = useState(initial || { firstName: "", lastName: "", birthDate: "", phone: "" });
    const set = (k, v) => setF(p => ({ ...p, [k]: v }));
    return (
      <Card>
        <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "0 14px" }}>
          <Input label="Nom" value={f.lastName} onChange={v => set("lastName", v)} required placeholder="Dupont" />
          <Input label="Prénom" value={f.firstName} onChange={v => set("firstName", v)} required placeholder="Jean" />
        </div>
        <Input label="Date de naissance" value={f.birthDate} onChange={v => set("birthDate", v)} type="date" required />
        <Input label="Téléphone (optionnel)" value={f.phone || ""} onChange={v => set("phone", v)} placeholder="06 12 34 56 78" />
        <div style={{ display: "flex", gap: 10, marginTop: 8 }}>
          <Btn variant="outline" onClick={onCancel}>Annuler</Btn>
          <Btn onClick={() => { if (f.lastName && f.firstName && f.birthDate) onSave(f); }} disabled={!f.lastName || !f.firstName || !f.birthDate}>Enregistrer</Btn>
        </div>
      </Card>
    );
  };

  /* ===== CONSULTATION FORM ===== */
  const ConsultationForm = ({ consultation: initial, patientId, onSave, onCancel }) => {
    const [f, setF] = useState(initial || { patientId, date: new Date().toISOString().split("T")[0], motif: "", history: "", antecedents: "", exam: "", diagnostic: "", treatment: "", isAdvanced: false, detailedHistory: "", complementaryExams: "", synthesis: "", prognosis: "", conduct: "" });
    const set = (k, v) => setF(p => ({ ...p, [k]: v }));
    const isAdv = f.isAdvanced;
    return (
      <Card style={{ padding: 24 }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 20 }}>
          <h3 style={{ fontFamily: "'Lora', serif", fontSize: 17 }}>{initial?.id ? "Modifier" : "Nouvelle"} consultation</h3>
          <button onClick={() => set("isAdvanced", !isAdv)} style={{ display: "flex", alignItems: "center", gap: 6, background: isAdv ? THEME.danger + "14" : THEME.accentLight, color: isAdv ? THEME.danger : THEME.accent, border: "none", borderRadius: 20, padding: "6px 14px", fontSize: 12, fontWeight: 600, cursor: "pointer", fontFamily: "inherit" }}>
            <Icon d={Icons.alert} size={14} /> {isAdv ? "Mode avancé ON" : "Mode avancé"}
          </button>
        </div>
        <Input label="Date" value={f.date} onChange={v => set("date", v)} type="date" required />
        <Input label="Motif de consultation" value={f.motif} onChange={v => set("motif", v)} rows={2} required placeholder="Motif principal..." />
        <Input label="Histoire de la maladie" value={f.history} onChange={v => set("history", v)} rows={3} placeholder="Début, évolution, facteurs..." />
        {isAdv && <Input label="Histoire détaillée (cas grave)" value={f.detailedHistory} onChange={v => set("detailedHistory", v)} rows={4} placeholder="Chronologie détaillée, facteurs aggravants..." style={{ background: "#FFF8F0", borderRadius: 10, padding: 2 }} />}
        <Input label="Antécédents" value={f.antecedents} onChange={v => set("antecedents", v)} rows={2} placeholder="Médicaux, chirurgicaux, familiaux..." />
        <Input label="Examen clinique" value={f.exam} onChange={v => set("exam", v)} rows={3} placeholder="Inspection, palpation, auscultation..." />
        {isAdv && <Input label="Examens complémentaires" value={f.complementaryExams} onChange={v => set("complementaryExams", v)} rows={3} placeholder="Biologie, imagerie, ECG..." style={{ background: "#FFF8F0", borderRadius: 10, padding: 2 }} />}
        <Input label="Diagnostic" value={f.diagnostic} onChange={v => set("diagnostic", v)} rows={2} placeholder="Diagnostic principal et différentiel..." />
        {isAdv && (
          <>
            <Input label="Synthèse clinique" value={f.synthesis} onChange={v => set("synthesis", v)} rows={3} placeholder="Vue d'ensemble du cas..." style={{ background: "#FFF8F0", borderRadius: 10, padding: 2 }} />
            <Input label="Pronostic" value={f.prognosis} onChange={v => set("prognosis", v)} options={["", ...PROGNOSIS_OPTS].map(o => ({ value: o, label: o || "— Choisir —" }))} style={{ background: "#FFF8F0", borderRadius: 10, padding: 2 }} />
          </>
        )}
        <Input label="Traitement" value={f.treatment} onChange={v => set("treatment", v)} rows={3} required placeholder="Médicaments, posologie, durée..." />
        {isAdv && <Input label="Conduite à tenir" value={f.conduct} onChange={v => set("conduct", v)} rows={3} placeholder="Surveillance, RDV, urgences..." style={{ background: "#FFF8F0", borderRadius: 10, padding: 2 }} />}
        <div style={{ display: "flex", gap: 10, marginTop: 12 }}>
          <Btn variant="outline" onClick={onCancel}>Annuler</Btn>
          <Btn onClick={() => { if (f.date && f.motif && f.treatment) onSave(f); }} disabled={!f.date || !f.motif || !f.treatment} icon={Icons.clipboard}>Enregistrer</Btn>
        </div>
      </Card>
    );
  };

  /* ===== VIEWS ===== */
  const renderView = () => {
    switch (view) {
      /* DASHBOARD */
      case "dashboard": return (
        <div className="fade-in">
          <Header title={`Bonjour, ${user.name.replace("Dr. ", "")}`} actions={<Btn variant="ghost" size="sm" icon={Icons.logout} onClick={logout}>Déconnexion</Btn>} />
          <div style={{ display: "flex", gap: 12, marginBottom: 20, flexWrap: "wrap" }}>
            <StatCard icon={Icons.users} label="Patients" value={myPatients.length} color={THEME.accent} />
            <StatCard icon={Icons.clipboard} label="Consultations" value={myConsultations.length} color={THEME.success} />
            <StatCard icon={Icons.alert} label="Cas graves" value={myConsultations.filter(c => c.isAdvanced).length} color={THEME.warning} />
          </div>
          <div style={{ display: "flex", gap: 12, marginBottom: 20, flexWrap: "wrap" }}>
            <Btn icon={Icons.plus} onClick={() => nav("new-patient")} size="lg" style={{ flex: 1, justifyContent: "center" }}>Nouveau patient</Btn>
            <Btn variant="secondary" icon={Icons.users} onClick={() => nav("patients")} size="lg" style={{ flex: 1, justifyContent: "center" }}>Mes patients</Btn>
          </div>
          <h3 style={{ fontSize: 14, color: THEME.textLight, textTransform: "uppercase", letterSpacing: 1, marginBottom: 12 }}>Consultations récentes</h3>
          {recentConsultations.length === 0 ? <Empty icon={Icons.clipboard} text="Aucune consultation" /> : recentConsultations.map(c => {
            const p = myPatients.find(pt => pt.id === c.patientId);
            return (
              <Card key={c.id} onClick={() => nav("consultation-detail", { consultation: c, patient: p })} style={{ marginBottom: 8, padding: 14, cursor: "pointer", transition: "box-shadow 0.2s" }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>{p ? `${p.lastName} ${p.firstName}` : "Patient inconnu"}</div>
                    <div style={{ fontSize: 13, color: THEME.textLight, marginTop: 2 }}>{c.motif}</div>
                  </div>
                  <div style={{ textAlign: "right" }}>
                    <div style={{ fontSize: 12, color: THEME.textLight }}>{fmt(c.date)}</div>
                    {c.isAdvanced && <Badge color={THEME.warning}>Avancé</Badge>}
                  </div>
                </div>
              </Card>
            );
          })}
        </div>
      );

      /* PATIENTS LIST */
      case "patients": return (
        <div className="fade-in">
          <Header title="Mes patients" backTo="dashboard" actions={<Btn size="sm" icon={Icons.plus} onClick={() => nav("new-patient")}>Ajouter</Btn>} />
          <div style={{ position: "relative", marginBottom: 16 }}>
            <Icon d={Icons.search} size={16} color={THEME.textLight} style={{ position: "absolute", left: 14, top: 12 }} />
            <input value={search} onChange={e => setSearch(e.target.value)} placeholder="Rechercher un patient..." style={{ width: "100%", padding: "10px 14px 10px 38px", borderRadius: 10, border: `1.5px solid ${THEME.border}`, fontSize: 14, fontFamily: "inherit", background: "#fff", outline: "none" }} />
          </div>
          {filteredPatients.length === 0 ? <Empty icon={Icons.users} text="Aucun patient trouvé" /> : filteredPatients.map((p, i) => (
            <Card key={p.id} onClick={() => nav("patient-detail", { patient: p })} className="slide-in" style={{ marginBottom: 8, padding: 14, cursor: "pointer", animationDelay: `${i * 40}ms`, animationFillMode: "backwards" }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                  <div style={{ width: 38, height: 38, borderRadius: 10, background: THEME.accentLight, display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 700, fontSize: 14, color: THEME.accent }}>{p.firstName[0]}{p.lastName[0]}</div>
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>{p.lastName} {p.firstName}</div>
                    <div style={{ fontSize: 12, color: THEME.textLight }}>{age(p.birthDate)} • {patientConsultations(p.id).length} consultation(s)</div>
                  </div>
                </div>
                <Icon d={Icons.back} size={16} color={THEME.border} style={{ transform: "rotate(180deg)" }} />
              </div>
            </Card>
          ))}
        </div>
      );

      /* NEW PATIENT */
      case "new-patient": return (
        <div className="fade-in">
          <Header title="Nouveau patient" backTo="patients" />
          <PatientForm onSave={async (p) => { await savePatient(p); nav("patients"); }} onCancel={() => nav("patients")} />
        </div>
      );

      /* EDIT PATIENT */
      case "edit-patient": return (
        <div className="fade-in">
          <Header title="Modifier patient" backTo="patient-detail" />
          <PatientForm patient={selected.patient} onSave={async (p) => { await savePatient(p); const up = { ...selected.patient, ...p }; nav("patient-detail", { patient: up }); }} onCancel={() => nav("patient-detail")} />
        </div>
      );

      /* PATIENT DETAIL */
      case "patient-detail": {
        const p = selected.patient;
        if (!p) return <div>Patient non trouvé</div>;
        const consults = patientConsultations(p.id);
        return (
          <div className="fade-in">
            <Header title={`${p.lastName} ${p.firstName}`} backTo="patients" actions={
              <div style={{ display: "flex", gap: 6 }}>
                <Btn variant="ghost" size="sm" icon={Icons.edit} onClick={() => nav("edit-patient", { patient: p })} />
                <Btn variant="ghost" size="sm" icon={Icons.trash} onClick={() => setConfirm({ msg: `Supprimer ${p.lastName} ${p.firstName} et toutes ses consultations ?`, action: () => deletePatient(p.id) })} style={{ color: THEME.danger }} />
              </div>
            } />
            <Card style={{ marginBottom: 16, padding: 16 }}>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10 }}>
                <div><div style={{ fontSize: 11, color: THEME.textLight, textTransform: "uppercase" }}>Date de naissance</div><div style={{ fontWeight: 500, marginTop: 2 }}>{fmt(p.birthDate)}</div></div>
                <div><div style={{ fontSize: 11, color: THEME.textLight, textTransform: "uppercase" }}>Âge</div><div style={{ fontWeight: 500, marginTop: 2 }}>{age(p.birthDate)}</div></div>
                {p.phone && <div><div style={{ fontSize: 11, color: THEME.textLight, textTransform: "uppercase" }}>Téléphone</div><div style={{ fontWeight: 500, marginTop: 2 }}>{p.phone}</div></div>}
              </div>
            </Card>
            <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 12 }}>
              <h3 style={{ fontSize: 14, color: THEME.textLight, textTransform: "uppercase", letterSpacing: 1 }}>Consultations ({consults.length})</h3>
              <Btn size="sm" icon={Icons.plus} onClick={() => nav("new-consultation", { patient: p })}>Nouvelle</Btn>
            </div>
            {consults.length === 0 ? <Empty icon={Icons.clipboard} text="Aucune consultation" /> : consults.map((c, i) => (
              <Card key={c.id} onClick={() => nav("consultation-detail", { consultation: c, patient: p })} className="slide-in" style={{ marginBottom: 8, padding: 14, cursor: "pointer", animationDelay: `${i * 40}ms`, animationFillMode: "backwards" }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                  <div>
                    <div style={{ fontWeight: 500, fontSize: 14 }}>{c.motif}</div>
                    <div style={{ fontSize: 12, color: THEME.textLight, marginTop: 2 }}>{c.diagnostic || "Diagnostic en attente"}</div>
                  </div>
                  <div style={{ textAlign: "right" }}>
                    <div style={{ fontSize: 12, color: THEME.textLight }}>{fmt(c.date)}</div>
                    {c.isAdvanced && <Badge color={THEME.danger}>Grave</Badge>}
                    {c.prognosis && <Badge color={PROGNOSIS_COLORS[c.prognosis]}>{c.prognosis}</Badge>}
                  </div>
                </div>
              </Card>
            ))}
          </div>
        );
      }

      /* NEW CONSULTATION */
      case "new-consultation": return (
        <div className="fade-in">
          <Header title="Nouvelle consultation" backTo="patient-detail" />
          <ConsultationForm patientId={selected.patient?.id} onSave={async (c) => { await saveConsultation(c); nav("patient-detail"); }} onCancel={() => nav("patient-detail")} />
        </div>
      );

      /* EDIT CONSULTATION */
      case "edit-consultation": return (
        <div className="fade-in">
          <Header title="Modifier consultation" backTo="consultation-detail" />
          <ConsultationForm consultation={selected.consultation} patientId={selected.patient?.id} onSave={async (c) => { await saveConsultation(c); nav("consultation-detail", { consultation: c }); }} onCancel={() => nav("consultation-detail")} />
        </div>
      );

      /* CONSULTATION DETAIL */
      case "consultation-detail": {
        const c = selected.consultation;
        const p = selected.patient || myPatients.find(pt => pt.id === c?.patientId);
        if (!c) return <div>Consultation non trouvée</div>;
        const Section = ({ title, content }) => content ? (
          <div style={{ marginBottom: 16 }}>
            <div style={{ fontSize: 11, textTransform: "uppercase", letterSpacing: 1, color: THEME.accent, fontWeight: 600, marginBottom: 4 }}>{title}</div>
            <div style={{ fontSize: 14, lineHeight: 1.65, whiteSpace: "pre-wrap", color: THEME.text }}>{content}</div>
          </div>
        ) : null;
        return (
          <div className="fade-in">
            <Header title="Consultation" backTo="patient-detail" actions={
              <div style={{ display: "flex", gap: 6 }}>
                <Btn variant="secondary" size="sm" icon={Icons.pdf} onClick={() => openReport(c, p)}>Rapport</Btn>
                <Btn variant="ghost" size="sm" icon={Icons.edit} onClick={() => nav("edit-consultation", { consultation: c, patient: p })} />
                <Btn variant="ghost" size="sm" icon={Icons.trash} onClick={() => setConfirm({ msg: "Supprimer cette consultation ?", action: () => deleteConsultation(c.id) })} style={{ color: THEME.danger }} />
              </div>
            } />
            <Card style={{ marginBottom: 12, padding: 16 }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12 }}>
                <div>
                  <div style={{ fontWeight: 600, fontSize: 15 }}>{p?.lastName} {p?.firstName}</div>
                  <div style={{ fontSize: 12, color: THEME.textLight }}>{age(p?.birthDate)}</div>
                </div>
                <div style={{ display: "flex", gap: 6, alignItems: "center" }}>
                  <div style={{ fontSize: 13, color: THEME.textLight }}>{fmtLong(c.date)}</div>
                  {c.isAdvanced && <Badge color={THEME.danger}>Avancé</Badge>}
                </div>
              </div>
              <div style={{ height: 1, background: THEME.border, margin: "8px 0 16px" }} />
              <Section title="Motif" content={c.motif} />
              <Section title="Histoire de la maladie" content={c.history} />
              {c.isAdvanced && <Section title="Histoire détaillée" content={c.detailedHistory} />}
              <Section title="Antécédents" content={c.antecedents} />
              <Section title="Examen clinique" content={c.exam} />
              {c.isAdvanced && <Section title="Examens complémentaires" content={c.complementaryExams} />}
              <Section title="Diagnostic" content={c.diagnostic} />
              {c.isAdvanced && <Section title="Synthèse clinique" content={c.synthesis} />}
              {c.isAdvanced && c.prognosis && (
                <div style={{ marginBottom: 16 }}>
                  <div style={{ fontSize: 11, textTransform: "uppercase", letterSpacing: 1, color: THEME.accent, fontWeight: 600, marginBottom: 6 }}>Pronostic</div>
                  <Badge color={PROGNOSIS_COLORS[c.prognosis]}>{c.prognosis}</Badge>
                </div>
              )}
              <Section title="Traitement" content={c.treatment} />
              {c.isAdvanced && <Section title="Conduite à tenir" content={c.conduct} />}
            </Card>
            <Btn size="lg" icon={Icons.pdf} onClick={() => openReport(c, p)} style={{ width: "100%", justifyContent: "center" }}>Générer le rapport PDF</Btn>
          </div>
        );
      }

      default: return <div>Vue inconnue</div>;
    }
  };

  /* ===== MAIN LAYOUT ===== */
  return (
    <>
      <style>{globalStyles}</style>
      <div style={{ minHeight: "100vh", background: THEME.bg }}>
        {/* Top bar */}
        <div className="no-print" style={{ background: THEME.primary, padding: "12px 20px", display: "flex", alignItems: "center", justifyContent: "space-between", position: "sticky", top: 0, zIndex: 100 }}>
          <div style={{ display: "flex", alignItems: "center", gap: 10, cursor: "pointer" }} onClick={() => nav("dashboard")}>
            <Icon d={Icons.shield} size={22} color={THEME.accent} />
            <span style={{ fontFamily: "'Lora', serif", fontSize: 17, fontWeight: 600, color: "#fff" }}>MédiConsult</span>
          </div>
          <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
            <span style={{ fontSize: 12, color: "rgba(255,255,255,0.6)" }}>{user.specialty}</span>
            <div style={{ width: 32, height: 32, borderRadius: 8, background: "rgba(255,255,255,0.12)", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 13, fontWeight: 600, color: "#fff" }}>{user.name.split(" ").map(w => w[0]).join("").slice(0, 2)}</div>
          </div>
        </div>
        {/* Bottom nav */}
        <div className="no-print" style={{ position: "fixed", bottom: 0, left: 0, right: 0, background: "#fff", borderTop: `1px solid ${THEME.border}`, display: "flex", justifyContent: "space-around", padding: "8px 0 12px", zIndex: 100 }}>
          {[
            { id: "dashboard", icon: Icons.home, label: "Accueil" },
            { id: "patients", icon: Icons.users, label: "Patients" },
            { id: "new-patient", icon: Icons.plus, label: "Nouveau" },
          ].map(item => {
            const active = view === item.id || (item.id === "patients" && ["patient-detail", "edit-patient", "new-consultation", "edit-consultation", "consultation-detail"].includes(view));
            return (
              <button key={item.id} onClick={() => nav(item.id)} style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 2, background: "none", border: "none", cursor: "pointer", padding: "4px 16px", fontFamily: "inherit" }}>
                <div style={{ width: item.id === "new-patient" ? 40 : "auto", height: item.id === "new-patient" ? 40 : "auto", borderRadius: item.id === "new-patient" ? 12 : 0, background: item.id === "new-patient" ? THEME.accent : "transparent", display: "flex", alignItems: "center", justifyContent: "center", marginBottom: item.id === "new-patient" ? -4 : 0, boxShadow: item.id === "new-patient" ? `0 4px 12px ${THEME.accent}40` : "none" }}>
                  <Icon d={item.icon} size={item.id === "new-patient" ? 20 : 22} color={item.id === "new-patient" ? "#fff" : active ? THEME.accent : THEME.textLight} />
                </div>
                <span style={{ fontSize: 10, fontWeight: active ? 600 : 400, color: active ? THEME.accent : THEME.textLight }}>{item.label}</span>
              </button>
            );
          })}
        </div>
        {/* Content */}
        <div style={{ padding: "16px 16px 80px", maxWidth: 640, margin: "0 auto" }}>
          {renderView()}
        </div>
      </div>
      {confirm && <Confirm message={confirm.msg} onYes={() => { confirm.action(); setConfirm(null); }} onNo={() => setConfirm(null)} />}
    </>
  );
}
