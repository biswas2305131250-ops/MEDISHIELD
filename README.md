"use client";

import React, { useState, useEffect, useMemo, useRef } from "react";
import { jsPDF } from "jspdf";
import autoTable from "jspdf-autotable";

// 🧪 TypeScript Interfaces
interface Drug {
  name: string;
  sku: string;
  manufacturer: string;
  batchStatus: "CRITICAL_EXPIRED" | "VERIFIED_SAFE" | "COUNTERFEIT_RISK";
  expiry: string;
  violations: number;
  severity: "High" | "Medium" | "None";
  color: string;
  siliconChipId: string;       
  totalSiliconNodes: number;   
  activeNodesMask: boolean[];  
  intakeDates: string[]; 
  soldCount: number;     
}

interface ViolationLog {
  id: string;
  date: string; 
  drug: string;
  info: string;
  status: string;
  color: string;
  pharmacyId: string; 
}

interface Pharmacy {
  id: string;
  name: string;
  licenseNo: string;
  coordinates: string;
  zone: "Dhaka North" | "Dhaka South";
  threatIndex: "LOW" | "MODERATE" | "HIGH";
  integrityRate: number;
}

const ModernPharmaLogo = ({ className = "w-10 h-10" }: { className?: string }) => (
  <svg className={className} viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
    <defs>
      <linearGradient id="shieldGrad" x1="0%" y1="0%" x2="100%" y2="100%">
        <stop offset="0%" stopColor="#10b981" />
        <stop offset="100%" stopColor="#059669" />
      </linearGradient>
      <linearGradient id="coreGrad" x1="0%" y1="0%" x2="100%" y2="100%">
        <stop offset="0%" stopColor="#34d399" />
        <stop offset="100%" stopColor="#047857" />
      </linearGradient>
    </defs>
    <path d="M50 5L85 20V50C85 72 70 88 50 95C30 88 15 72 15 50V20L50 5Z" fill="none" stroke="url(#shieldGrad)" strokeWidth="4" strokeLinejoin="round"/>
    <path d="M50 12L78 24V48C78 66 66 80 50 86C34 80 22 66 22 48V24L50 12Z" fill="url(#coreGrad)" opacity="0.15"/>
    <path d="M50 28V68" stroke="#10b981" strokeWidth="6" strokeLinecap="round"/>
    <path d="M30 48H70" stroke="#10b981" strokeWidth="6" strokeLinecap="round"/>
    <circle cx="50" cy="48" r="8" fill="#030712" stroke="#34d399" strokeWidth="3"/>
    <circle cx="50" cy="48" r="3" fill="#34d399"/>
  </svg>
);

const generateThousandDrugs = (): Drug[] => {
  const genericNames = ["Napa Extend", "Seclo 20", "Sergel 20", "Xinc 20", "Pantodex 40", "Ace Plus", "Esoral 20", "Fexo 120", "Monas 10", "Alatrol 10", "Maxpro 20", "Zimax 500"];
  const manufacturers = ["Beximco Pharmaceuticals", "Square Pharmaceuticals", "Healthcare Pharma", "Aristopharma Ltd.", "Incepta Pharma", "Renata Limited"];
  const statuses: ("CRITICAL_EXPIRED" | "VERIFIED_SAFE" | "COUNTERFEIT_RISK")[] = ["VERIFIED_SAFE", "VERIFIED_SAFE", "VERIFIED_SAFE", "COUNTERFEIT_RISK", "CRITICAL_EXPIRED"];
  const months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];

  const generatedList: Drug[] = [];
  for (let i = 1; i <= 1000; i++) {
    const nameStr = `${genericNames[i % genericNames.length]}`;
    const skuStr = `${genericNames[i % genericNames.length].substring(0, 3).toUpperCase()}-${1000 + i}`;
    const statusStr = statuses[i % statuses.length];
    
    let expiryStr = `${months[i % months.length]} 2027`;
    let violationsNum = 0;
    let colorStr = "#10b981";

    if (statusStr === "CRITICAL_EXPIRED") {
      expiryStr = `${months[i % months.length]} 2026`;
      violationsNum = (i % 6) + 3; 
      colorStr = "#f43f5e";
    } else if (statusStr === "COUNTERFEIT_RISK") {
      expiryStr = "Unknown Silicon Signature";
      violationsNum = (i % 3) + 1; 
      colorStr = "#f59e0b";
    }

    const nodeCount = 10; 
    const soldPieces = i % 4; 
    const mask = Array.from({ length: nodeCount }, (_, idx) => idx >= soldPieces);

    generatedList.push({
      name: nameStr,
      sku: skuStr,
      manufacturer: manufacturers[i % manufacturers.length],
      batchStatus: statusStr,
      expiry: expiryStr,
      violations: violationsNum,
      severity: statusStr === "CRITICAL_EXPIRED" ? "High" : statusStr === "COUNTERFEIT_RISK" ? "Medium" : "None",
      color: colorStr,
      siliconChipId: `HEX-Si77_${10000 + i}_${skuStr}`,
      totalSiliconNodes: nodeCount,
      activeNodesMask: mask,
      intakeDates: soldPieces > 0 ? [`2026-06-27 10:14:${10 + (i % 40)}`, `2026-06-27 18:32:${15 + (i % 35)}`].slice(0, soldPieces) : ["No sales recorded"],
      soldCount: soldPieces
    });
  }
  return generatedList;
};

const pharmacyDatabase: Pharmacy[] = [
  { id: "p1", name: "Al-Madina Pharmacy", licenseNo: "DL-44821/DHK-2026", coordinates: "Dhanmondi Node (23.7509° N, 90.3726° E)", zone: "Dhaka North", threatIndex: "MODERATE", integrityRate: 94.2 },
  { id: "p2", name: "Lazz Pharma (Uttara)", licenseNo: "DL-99210/DHK-2026", coordinates: "Uttara Sector 4 (23.8759° N, 90.4007° E)", zone: "Dhaka North", threatIndex: "LOW", integrityRate: 99.1 },
  { id: "p3", name: "Green Plus Healthcare", licenseNo: "DL-11029/DHK-2026", coordinates: "Gulshan-2 Circle (23.7925° N, 90.4154° E)", zone: "Dhaka North", threatIndex: "HIGH", integrityRate: 81.5 },
  { id: "p4", name: "Shahbagh Medicine Corner", licenseNo: "DL-88321/DHK-2026", coordinates: "Shahbagh Intersection (23.7384° N, 90.3958° E)", zone: "Dhaka South", threatIndex: "LOW", integrityRate: 98.4 },
  { id: "p5", name: "Dhaka Pharma & Surgical", licenseNo: "DL-33412/DHK-2026", coordinates: "Lalbagh Fort Road (23.7189° N, 90.3882° E)", zone: "Dhaka South", threatIndex: "HIGH", integrityRate: 74.8 }
];

export default function MediShieldUltimateDashboard() {
  const [isMounted, setIsMounted] = useState(false);
  const [activeTab, setActiveTab] = useState<"counter" | "dncc_portal">("counter");
  const [selectedZone, setSelectedZone] = useState<"Dhaka North" | "Dhaka South">("Dhaka North");
  const [selectedPharmacy, setSelectedPharmacy] = useState<Pharmacy>(pharmacyDatabase[0]);
  
  const [localDrugDatabase, setLocalDrugDatabase] = useState<Drug[]>([]);
  const [selectedDrug, setSelectedDrug] = useState<Drug | null>(null);

  const [scanState, setScanState] = useState<"idle" | "scanning" | "done">("idle");
  const [licenseLocked, setLicenseLocked] = useState<boolean>(false);
  const [drugSearch, setDrugSearch] = useState<string>("");
  
  const [logs, setLogs] = useState<ViolationLog[]>([]);
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 });

  // 🎥 Live Camera States
  const videoRef = useRef<HTMLVideoElement | null>(null);
  const streamRef = useRef<MediaStream | null>(null);
  const [isCameraActive, setIsCameraActive] = useState<boolean>(false);
  const [cameraError, setCameraError] = useState<string | null>(null);

  const getLiveTimestamp = () => {
    const now = new Date();
    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, '0');
    const day = String(now.getDate()).padStart(2, '0');
    const hours = String(now.getHours()).padStart(2, '0');
    const minutes = String(now.getMinutes()).padStart(2, '0');
    const seconds = String(now.getSeconds()).padStart(2, '0');
    return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
  };

  useEffect(() => {
    setIsMounted(true);
    const initialDrugs = generateThousandDrugs();
    setLocalDrugDatabase(initialDrugs);
    setSelectedDrug(initialDrugs[0]);

    setLogs([
      { id: "init-1", date: "2026-06-28 08:12:04", drug: "Napa Extend", info: "Silicon Telemetry Mesh Disrupted // 2 Nodes Disconnected", status: "STRIP_MODIFIED", color: "text-cyan-400 bg-cyan-500/10 border-cyan-500/20", pharmacyId: "p1" },
      { id: "init-2", date: "2026-06-28 08:14:12", drug: "Sergel 20", info: "SKU SER-1003 // Mismatched Hologram Core Signature Detected", status: "COUNTERFEIT_ALERT", color: "text-amber-400 bg-amber-500/10 border-amber-500/20", pharmacyId: "p1" },
      { id: "init-3", date: "2026-06-28 08:15:45", drug: "Napa Extend", info: "Silicon Telemetry Mesh Disrupted // 2 Nodes Disconnected", status: "STRIP_MODIFIED", color: "text-cyan-400 bg-cyan-500/10 border-cyan-500/20", pharmacyId: "p3" }
    ]);

    const handleMouseMove = (e: MouseEvent) => {
      setMousePos({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener("mousemove", handleMouseMove);
    return () => window.removeEventListener("mousemove", handleMouseMove);
  }, []);

  // 🎥 ক্যামেরা স্টার্ট করার আপগ্রেডেড ফাংশন (উইথ ফলব্যাক)
  const startCamera = async () => {
    setCameraError(null);
    try {
      const constraints = {
        video: { 
          facingMode: { ideal: "environment" }, 
          width: { ideal: 640 }, 
          height: { ideal: 480 } 
        }
      };
      
      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      streamRef.current = stream;
      if (videoRef.current) {
        videoRef.current.srcObject = stream;
      }
      setIsCameraActive(true);
    } catch (err: any) {
      console.error("Camera access error, trying fallback:", err);
      try {
        // ফলব্যাক: ডিভাইস ফ্রন্ট/ব্যাক কনফ্লিক্ট এড়াতে যেকোনো সচল ক্যামেরা দিয়ে ট্রাই
        const fallbackStream = await navigator.mediaDevices.getUserMedia({ video: true });
        streamRef.current = fallbackStream;
        if (videoRef.current) {
          videoRef.current.srcObject = fallbackStream;
        }
        setIsCameraActive(true);
      } catch (fallbackErr: any) {
        console.error("All camera access denied:", fallbackErr);
        setCameraError("Camera permission blocked or not found. Please allow camera access from address bar.");
        setIsCameraActive(false);
      }
    }
  };

  // 🎥 ক্যামেরা স্টপ করার ফাংশন
  const stopCamera = () => {
    if (streamRef.current) {
      streamRef.current.getTracks().forEach(track => track.stop());
      streamRef.current = null;
    }
    if (videoRef.current) {
      videoRef.current.srcObject = null;
    }
    setIsCameraActive(false);
  };

  useEffect(() => {
    if (activeTab !== "counter") {
      stopCamera();
    }
    return () => stopCamera();
  }, [activeTab]);

  const filteredPharmacies = useMemo(() => {
    return pharmacyDatabase.filter(p => p.zone === selectedZone);
  }, [selectedZone]);

  useEffect(() => {
    if (!isMounted || filteredPharmacies.length === 0) return;
    setSelectedPharmacy(filteredPharmacies[0]);
  }, [selectedZone, filteredPharmacies, isMounted]);

  useEffect(() => {
    if (!isMounted) return;
    
    const timestamp = getLiveTimestamp();
    const logExist = logs.some(log => log.id === `pharm-sync-${selectedPharmacy.id}`);
    
    if (!logExist) {
      setLogs(prev => [
        {
          id: `pharm-sync-${selectedPharmacy.id}`,
          date: timestamp,
          drug: "SYSTEM COUPLING",
          info: `Connected endpoint to core registry cluster: ${selectedPharmacy.name}`,
          status: "NODE_CONNECTED",
          color: "text-indigo-400 bg-indigo-500/10 border-indigo-500/20",
          pharmacyId: selectedPharmacy.id
        },
        ...prev
      ]);
    }
  }, [selectedPharmacy, isMounted]);

  const filteredDrugs = useMemo(() => {
    return localDrugDatabase.filter(drug => 
      drug.name.toLowerCase().includes(drugSearch.toLowerCase()) || 
      drug.sku.toLowerCase().includes(drugSearch.toLowerCase())
    );
  }, [drugSearch, localDrugDatabase]);

  const handleTriggerScan = () => {
    if (licenseLocked || !selectedDrug) {
      alert("Access Denied or No Asset Loaded.");
      return;
    }

    if (!isCameraActive) {
      startCamera();
    }

    setScanState("scanning");
    
    setTimeout(() => {
      setScanState("done");
      const timestamp = getLiveTimestamp(); 
      
      setLogs(prev => [
        {
          id: `scan-ok-${Date.now()}`,
          date: timestamp,
          drug: selectedDrug.name,
          info: `Silicon Link Passed [ID: ${selectedDrug.siliconChipId.substring(0, 16)}...]. Blister matrix telemetry is fully sync'd.`,
          status: "LIVE_NODES_SYNCED",
          color: "text-emerald-400 bg-emerald-500/10 border-emerald-500/20",
          pharmacyId: selectedPharmacy.id
        },
        ...prev
      ]);
    }, 1500);
  };

  const handleCutSinglePiece = (pillIndex: number) => {
    if (!selectedDrug || selectedDrug.activeNodesMask[pillIndex] === false) return;
    
    const timestamp = getLiveTimestamp();

    const updatedMask = [...selectedDrug.activeNodesMask];
    updatedMask[pillIndex] = false; 

    const updatedIntakeDates = [...selectedDrug.intakeDates];
    if (updatedIntakeDates[0] === "No sales recorded") {
      updatedIntakeDates[0] = timestamp;
    } else {
      updatedIntakeDates.push(timestamp);
    }

    const updatedDrug: Drug = {
      ...selectedDrug,
      activeNodesMask: updatedMask,
      soldCount: selectedDrug.soldCount + 1,
      intakeDates: updatedIntakeDates
    };

    setLocalDrugDatabase(prev => prev.map(d => d.sku === selectedDrug.sku ? updatedDrug : d));
    setSelectedDrug(updatedDrug);

    setLogs(prev => [
      {
        id: `node-cut-${Date.now()}-${pillIndex}`,
        date: timestamp, 
        drug: selectedDrug.name,
        info: `Blister Hardware Node-${pillIndex + 1} ruptured manually. Sold piece tracked instantly.`,
        status: "PIECE_DISPATCHED",
        color: "text-rose-400 bg-rose-500/10 border-rose-500/20",
        pharmacyId: selectedPharmacy.id
      },
      ...prev
    ]);
  };

  const toggleLockdown = () => {
    const nextState = !licenseLocked;
    setLicenseLocked(nextState);
    const timestamp = getLiveTimestamp();
    
    setLogs(prev => [
      {
        id: `lockdown-toggle-${Date.now()}`,
        date: timestamp,
        drug: "CORE OVERRIDE",
        info: nextState ? `Terminal Operational Block enforced on ${selectedPharmacy.name}. Dossier suspended.` : `Emergency lockdown cleared for ${selectedPharmacy.name}`,
        status: nextState ? "TERMINATED" : "REINSTATED",
        color: nextState ? "text-rose-500 bg-rose-500/20 border-rose-500/40" : "text-emerald-400 bg-emerald-500/10 border-emerald-500/20",
        pharmacyId: selectedPharmacy.id
      },
      ...prev
    ]);
  };

  const generatePDFReport = () => {
    try {
      const doc = new jsPDF("p", "mm", "a4");
      
      doc.setFillColor(16, 185, 129); 
      doc.circle(105, 22, 10, "F");
      doc.setFillColor(3, 7, 18); 
      doc.circle(105, 22, 8, "F");
      
      doc.setTextColor(16, 185, 129);
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(13);
      doc.text("MINISTRY OF HEALTH & CONSUMER RIGHTS PORTAL", 105, 38, { align: "center" });
      
      doc.setTextColor(148, 163, 184);
      doc.setFont("Helvetica", "normal");
      doc.setFontSize(9);
      doc.text("Directorate General of Consumer Rights Protection (DNCRP) Audit Hub", 105, 43, { align: "center" });

      doc.setFont("Courier", "bold");
      doc.setFontSize(8);
      doc.text(`GOVT_REF: DNCRP/AUDIT/${selectedZone.toUpperCase().replace(/\s+/g, '_')}/2026`, 14, 54);
      doc.text(`REPORT GEN TIME: ${getLiveTimestamp()}`, 125, 54);

      doc.setDrawColor(16, 185, 129);
      doc.line(14, 57, 196, 57);

      doc.setFont("Helvetica", "bold");
      doc.setFontSize(10);
      doc.setTextColor(15, 23, 42);
      doc.text(`OFFICIAL COMPLIANCE & FRACTIONAL MEDICINE DOSSIER`, 14, 65);

      doc.setFillColor(248, 250, 252);
      doc.rect(14, 70, 182, 28, "FD");

      doc.setFont("Helvetica", "normal");
      doc.setFontSize(8.5);
      doc.setTextColor(71, 85, 105);
      doc.text(`Target Pharmacy:  ${selectedPharmacy.name}`, 18, 79);
      doc.text(`Drug License Key: ${selectedPharmacy.licenseNo}`, 18, 84);
      doc.text(`Threat Matrix Profile: ${selectedPharmacy.threatIndex} RISK`, 120, 79);
      doc.text(`Compliance Rating:    ${selectedPharmacy.integrityRate}%`, 120, 84);

      const tableRows = localDrugDatabase.slice(0, 25).map((drug) => {
        const remaining = drug.activeNodesMask.filter(Boolean).length;
        const formattedDates = drug.intakeDates.join(", \n");
        return [
          drug.name,
          drug.sku,
          drug.soldCount.toString(),
          remaining.toString(),
          formattedDates
        ];
      });

      autoTable(doc, {
        startY: 105,
        head: [["Medicine Name", "SKU Key", "Sold Qty", "Remaining", "Intake/Dispatched Timestamps (Real-time)"]],
        body: tableRows,
        theme: "striped",
        headStyles: { fillColor: [15, 23, 42], textColor: [255, 255, 255], fontSize: 8 },
        bodyStyles: { fontSize: 7.5, cellPadding: 3 },
        columnStyles: { 4: { cellWidth: 65 } }
      });

      const currentFinalY = (doc as any).lastAutoTable.finalY || 180;
      const signatureYStart = Math.min(currentFinalY + 20, 270);

      doc.setDrawColor(203, 213, 225); 
      doc.line(14, signatureYStart, 74, signatureYStart); 
      doc.text("// Verified Ministry Oversight Hash Signed", 14, signatureYStart + 4);
      doc.text("Government Lead Auditor Signature", 14, signatureYStart + 9);

      doc.save(`Govt_Audit_Report_${selectedPharmacy.name.replace(/\s+/g, '_')}.pdf`);
    } catch (error) {
      console.error(error);
    }
  };

  if (!isMounted || !selectedDrug) {
    return <div className="min-h-screen bg-[#030712] text-slate-400 flex items-center justify-center font-mono text-xs">INITIALIZING SECURE DATA ENGINE COMPONENTS...</div>;
  }

  return (
    <div className="min-h-screen bg-[#030712] text-slate-200 p-8 font-sans selection:bg-[#10b981] selection:text-black overflow-x-hidden relative">
      
      <div 
        className="pointer-events-none fixed transition-transform duration-75 ease-out -translate-x-1/2 -translate-y-1/2 rounded-full z-50 hidden lg:block"
        style={{
          left: `${mousePos.x}px`,
          top: `${mousePos.y}px`,
          width: "400px",
          height: "400px",
          background: "radial-gradient(circle, rgba(16,185,129,0.08) 0%, rgba(5,150,105,0.01) 60%, transparent 100%)",
        }}
      />

      <header className="max-w-7xl mx-auto flex flex-col md:flex-row justify-between items-start md:items-center border-b border-slate-800 pb-6 mb-10 gap-6">
        <div className="flex items-center gap-4">
          <div className="relative p-1 bg-slate-950 rounded-xl border border-emerald-500/30 shadow-[0_0_20px_rgba(16,185,129,0.15)]">
            <ModernPharmaLogo className="w-12 h-12" />
          </div>
          <div>
            <h1 className="text-2xl font-extrabold tracking-tight text-white flex items-center gap-2">
              MEDISHIELD <span className="text-emerald-400 font-mono text-xs px-2 py-0.5 bg-emerald-500/10 rounded-md border border-emerald-500/20">SILICON MATRIX v6.5</span>
            </h1>
            <p className="text-xs text-slate-400 font-mono tracking-wider mt-0.5">LIVE CAMERA OPTICAL SCANNERS // GOVERNMENT COMPLIANCE AUDIT ENGINE</p>
          </div>
        </div>

        <div className="flex bg-slate-900/90 p-1 rounded-xl border border-slate-800 shadow-2xl backdrop-blur-md">
          <button 
            onClick={() => setActiveTab("counter")}
            className={`px-5 py-2.5 rounded-lg text-xs font-bold uppercase tracking-wider transition-all duration-200 ${activeTab === "counter" ? "bg-gradient-to-r from-slate-800 to-slate-900 border border-slate-700 text-emerald-400 shadow-lg" : "text-slate-400 hover:text-white"}`}
          >
            📸 Live Retail Cell Scanner
          </button>
          <button 
            onClick={() => setActiveTab("dncc_portal")}
            className={`px-5 py-2.5 rounded-lg text-xs font-bold uppercase tracking-wider transition-all duration-200 ${activeTab === "dncc_portal" ? "bg-gradient-to-r from-emerald-800 to-emerald-950 border border-emerald-700 text-white shadow-lg" : "text-slate-400 hover:text-white"}`}
          >
            🏛️ Government Ministry Portal
          </button>
        </div>
      </header>

      <main className="max-w-7xl mx-auto relative z-10 space-y-8">
        
        {activeTab === "counter" && (
          <>
            <div className="bg-gradient-to-r from-slate-950 to-slate-900 border border-slate-800 rounded-2xl p-6 shadow-xl relative overflow-hidden">
              <h3 className="text-xs font-bold text-emerald-400 font-mono tracking-widest uppercase mb-4 flex items-center gap-2">
                <ModernPharmaLogo className="w-4 h-4" /> // Live Terminal Feed: {selectedPharmacy.name}
              </h3>
              
              <div className="grid grid-cols-1 md:grid-cols-3 gap-6 text-sm">
                <div className="bg-slate-900/40 p-4 rounded-xl border border-slate-800/80">
                  <span className="text-[10px] text-slate-400 uppercase font-mono block">Active Pharmacy Node</span>
                  <span className="text-white font-bold text-base block mt-1">{selectedPharmacy.name}</span>
                </div>
                <div className="bg-slate-900/40 p-4 rounded-xl border border-slate-800/80">
                  <span className="text-[10px] text-slate-400 uppercase font-mono block">Core Drug License Key</span>
                  <span className="text-emerald-400 font-mono font-bold block mt-1.5">{selectedPharmacy.licenseNo}</span>
                </div>
                <div className="bg-slate-900/40 p-4 rounded-xl border border-slate-800/80">
                  <span className="text-[10px] text-slate-400 uppercase font-mono block">Active Audit Horizon</span>
                  <span className="text-slate-200 font-bold block mt-1.5">REAL-TIME TELEMETRY TRACKING</span>
                </div>
              </div>
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
              <div className="bg-slate-900/20 border border-slate-800 rounded-2xl p-6 backdrop-blur-xl flex flex-col justify-between">
                <div>
                  <h3 className="text-xs font-bold text-slate-400 uppercase tracking-widest mb-3">Asset Inventory List</h3>
                  
                  <input 
                    type="text" 
                    placeholder="Search medicines..."
                    value={drugSearch}
                    onChange={(e) => setDrugSearch(e.target.value)}
                    className="w-full mb-4 px-3 py-2 bg-slate-950 border border-slate-800 rounded-lg text-xs focus:outline-none focus:border-emerald-500/40 text-white font-mono"
                  />

                  <div className="space-y-2 max-h-[160px] overflow-y-auto pr-1 mb-4">
                    {filteredDrugs.slice(0, 15).map((drug, idx) => {
                      const itemsLeft = drug.activeNodesMask.filter(Boolean).length;
                      return (
                        <div 
                          key={`drug-item-${idx}`} 
                          onClick={() => { setSelectedDrug(drug); setScanState("idle"); }}
                          className={`p-2.5 rounded-xl border transition cursor-pointer text-xs ${selectedDrug.sku === drug.sku ? "bg-slate-900 border-slate-700" : "bg-slate-950 border-transparent"}`}
                        >
                          <div className="flex justify-between items-center">
                            <h4 className="font-bold text-white truncate">{drug.name}</h4>
                            <span className="text-[9px] font-mono px-1 py-0.5 rounded" style={{ color: drug.color }}>
                              {itemsLeft} Left
                            </span>
                          </div>
                        </div>
                      );
                    })}
                  </div>

                  {/* 🎥 LIVE CAMERA SUB-SYSTEM COMPONENT */}
                  <div className="border border-slate-800 rounded-xl overflow-hidden bg-slate-950 relative p-2 mb-2">
                    <div className="flex justify-between items-center mb-2 px-1">
                      <span className="text-[9px] font-mono font-bold tracking-wider text-slate-400 uppercase flex items-center gap-1">
                        <span className={`w-1.5 h-1.5 rounded-full ${isCameraActive ? "bg-emerald-500 animate-ping" : "bg-slate-600"}`} />
                        OPTICAL CAM MODULE
                      </span>
                      <button 
                        type="button"
                        onClick={isCameraActive ? stopCamera : startCamera}
                        className={`text-[9px] font-mono px-2 py-0.5 rounded border font-bold transition ${isCameraActive ? "bg-rose-500/10 text-rose-400 border-rose-500/20" : "bg-emerald-500/10 text-emerald-400 border-emerald-500/20"}`}
                      >
                        {isCameraActive ? "SHUTDOWN CAM" : "INITIALIZE CAM"}
                      </button>
                    </div>

                    <div className="relative aspect-video w-full bg-slate-900/60 rounded-lg border border-slate-800 flex flex-col items-center justify-center overflow-hidden">
                      {isCameraActive ? (
                        <>
                          <video ref={videoRef} autoPlay playsInline muted className="w-full h-full object-cover" />
                          <div className="absolute top-0 left-0 w-full h-[2px] bg-gradient-to-r from-transparent via-emerald-400 to-transparent shadow-[0_0_10px_#10b981] animate-bounce" style={{ animationDuration: '2.5s' }} />
                          <div className="absolute inset-4 border border-emerald-500/20 pointer-events-none rounded flex items-center justify-center">
                            <div className="w-12 h-12 border-t-2 border-l-2 border-emerald-400 absolute top-0 left-0" />
                            <div className="w-12 h-12 border-t-2 border-r-2 border-emerald-400 absolute top-0 right-0" />
                            <div className="w-12 h-12 border-b-2 border-l-2 border-emerald-400 absolute bottom-0 left-0" />
                            <div className="w-12 h-12 border-b-2 border-r-2 border-emerald-400 absolute bottom-0 right-0" />
                          </div>
                        </>
                      ) : (
                        <div className="text-center p-4">
                          <span className="text-xl block mb-1">📷</span>
                          <p className="text-[10px] text-slate-500 font-mono">Camera Feed Idle</p>
                          {cameraError && <p className="text-[8px] text-rose-400 font-mono mt-1 px-2">{cameraError}</p>}
                        </div>
                      )}
                    </div>
                  </div>
                </div>

                <button 
                  onClick={handleTriggerScan} 
                  disabled={scanState === "scanning" || licenseLocked} 
                  className="w-full mt-2 py-3 bg-gradient-to-r from-emerald-600 to-teal-600 hover:from-emerald-500 hover:to-teal-500 text-white font-black rounded-xl text-xs uppercase tracking-wider transition disabled:opacity-40"
                >
                  📡 TRIGGER CHIP MATRIX LOOP SCANNERS
                </button>
              </div>

              <div className="lg:col-span-2 bg-slate-900/20 border border-slate-800 rounded-2xl p-8 backdrop-blur-xl flex flex-col justify-center min-h-[360px]">
                {scanState === "idle" && (
                  <div className="text-center max-w-sm mx-auto">
                    <div className="w-14 h-14 bg-slate-950 rounded-2xl border border-slate-800 flex items-center justify-center text-xl mx-auto mb-4 text-emerald-400">📡</div>
                    <h4 className="text-xs font-bold text-slate-400 uppercase tracking-wider">Awaiting Silicon Hardware Signal</h4>
                    <p className="text-[11px] text-slate-500 mt-1.5">Select a medication or switch on the optical module cam to map hardware blister parameters.</p>
                  </div>
                )}
                {scanState === "scanning" && (
                  <div className="text-center">
                    <div className="w-10 h-10 border-4 border-emerald-500/20 border-t-emerald-400 rounded-full animate-spin mx-auto mb-4" />
                    <p className="text-[10px] font-mono text-emerald-400 tracking-widest animate-pulse">LOCKING CAMERA SCAN TARGET WITH CORE TELEMETRY...</p>
                  </div>
                )}
                {scanState === "done" && (
                  <div className="space-y-6">
                    <div className="flex justify-between items-center border-b border-slate-800 pb-3">
                      <div>
                        <span className="text-[10px] font-mono text-cyan-400 block">[REAL-TIME TELEMETRY CONNECTED]</span>
                        <h3 className="text-xl font-black text-white mt-0.5">{selectedDrug.name}</h3>
                        <p className="text-xs text-slate-500 font-mono mt-0.5">Chip Core ID: <span className="text-emerald-400 font-semibold">{selectedDrug.siliconChipId}</span></p>
                      </div>
                    </div>

                    <div className="space-y-3">
                      <h4 className="text-xs font-bold text-slate-400 uppercase font-mono tracking-wider">
                        // Interactive Strip Layout (Click any live node to sell and record real date/time)
                      </h4>
                      <div className="grid grid-cols-5 gap-3 bg-slate-950 p-6 rounded-2xl border border-slate-900">
                        {selectedDrug.activeNodesMask.map((isPillIntact, idx) => (
                          <div 
                            key={`node-pill-${idx}`}
                            onClick={() => handleCutSinglePiece(idx)}
                            className={`relative p-4 rounded-xl border transition text-center ${
                              isPillIntact 
                                ? "bg-gradient-to-b from-slate-900 to-slate-950 border-emerald-500/30 hover:border-emerald-400 cursor-pointer shadow-[0_0_15px_rgba(16,185,129,0.05)]" 
                                : "bg-slate-900/10 border-slate-900 opacity-20 cursor-not-allowed filter grayscale"
                            }`}
                          >
                            <span className={`absolute top-1.5 left-1.5 w-1.5 h-1.5 rounded-full ${isPillIntact ? "bg-emerald-400" : "bg-slate-700"}`} />
                            <span className="text-lg block mb-0.5">{isPillIntact ? "💊" : "❌"}</span>
                            <span className="text-[9px] font-mono font-bold block text-slate-300">NODE-0{idx + 1}</span>
                            <span className="text-[8px] font-mono text-slate-500 block mt-0.5 uppercase">{isPillIntact ? "Intact" : "Sold"}</span>
                          </div>
                        ))}
                      </div>
                    </div>

                    <div className="bg-slate-950 p-4 rounded-xl border border-slate-900">
                      <span className="text-[9px] text-slate-400 font-mono block">RECENT LOCAL DISPATCH HISTORY FOR THIS STRIP (REAL-TIME TIMESTAMPS)</span>
                      <div className="max-h-[80px] overflow-y-auto mt-2 space-y-1 text-xs font-mono text-emerald-400">
                        {selectedDrug.intakeDates.map((date, index) => (
                          <p key={index}>⚡ Sold Piece {index + 1} Timestamp: {date}</p>
                        ))}
                      </div>
                    </div>
                  </div>
                )}
              </div>
            </div>
          </>
        )}

        {activeTab === "dncc_portal" && (
          <div className="space-y-8">
            <div className="relative bg-gradient-to-br from-slate-950 via-slate-900 to-slate-950 border border-slate-800 rounded-3xl p-6 overflow-hidden shadow-2xl">
              <div className="relative z-10">
                <div className="inline-flex items-center gap-2 px-2.5 py-1 rounded-full bg-emerald-500/10 border border-emerald-500/20 text-emerald-400 text-[10px] font-mono uppercase tracking-widest mb-2">
                  ● GOVERNMENT EXECUTIVE OVERWATCH MODALITY ACTIVE
                </div>
                <h2 className="text-2xl font-black text-white tracking-tight">
                  DNCRP National Compliance Dashboard
                </h2>
              </div>
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
              <div className="space-y-6">
                <div className="bg-slate-900/80 border border-slate-800 rounded-2xl p-5 shadow-xl backdrop-blur-md">
                  <span className="text-[10px] text-emerald-400 uppercase font-mono block tracking-widest mb-3">// Territorial Sector Selection</span>
                  
                  <div className="grid grid-cols-2 bg-slate-950 p-1 rounded-xl border border-slate-800 mb-5">
                    <button onClick={() => setSelectedZone("Dhaka North")} className={`py-2 rounded-lg text-xs font-bold uppercase transition ${selectedZone === "Dhaka North" ? "bg-emerald-600 text-white" : "text-slate-400"}`}>Dhaka North</button>
                    <button onClick={() => setSelectedZone("Dhaka South")} className={`py-2 rounded-lg text-xs font-bold uppercase transition ${selectedZone === "Dhaka South" ? "bg-emerald-600 text-white" : "text-slate-400"}`}>Dhaka South</button>
                  </div>

                  <span className="text-[10px] text-slate-400 uppercase font-mono block tracking-widest mb-2">// Registered Facility Clusters ({filteredPharmacies.length})</span>
                  <div className="space-y-2 max-h-[220px] overflow-y-auto pr-1">
                    {filteredPharmacies.map((pharmacy) => (
                      <div key={pharmacy.id} onClick={() => setSelectedPharmacy(pharmacy)} className={`p-3 rounded-xl border cursor-pointer ${selectedPharmacy.id === pharmacy.id ? "bg-slate-950 border-emerald-500/50" : "bg-slate-950/50 border-slate-800"}`}>
                        <h4 className="font-bold text-xs text-white">🏪 {pharmacy.name}</h4>
                        <span className="text-[10px] text-slate-500 font-mono block mt-0.5">{pharmacy.licenseNo}</span>
                      </div>
                    ))}
                  </div>
                </div>

                <div className="bg-slate-900/40 border border-slate-800 rounded-2xl p-5 space-y-3">
                  <span className="text-[10px] text-slate-400 uppercase font-mono block tracking-widest">// Executive Authority Actions</span>
                  <button 
                    onClick={generatePDFReport}
                    className="w-full py-3.5 bg-gradient-to-r from-emerald-600 to-teal-600 text-white font-bold text-xs rounded-xl uppercase tracking-wider shadow-lg"
                  >
                    📝 GENERATE AUDIT REPORT WITH LIVE SALES DOSSIERS
                  </button>
                  <button onClick={toggleLockdown} className={`w-full py-3 font-bold text-xs rounded-xl uppercase transition border ${licenseLocked ? "bg-rose-600 text-white" : "text-rose-400 border-rose-900/40"}`}>
                    {licenseLocked ? "🔓 Release Lockdown" : "🔒 Enforce Facility Lockdown"}
                  </button>
                </div>
              </div>

              <div className="lg:col-span-2 bg-gradient-to-b from-slate-950 to-slate-900 border border-slate-800 rounded-2xl p-6 shadow-xl flex flex-col justify-between">
                <div>
                  <div className="flex justify-between items-start border-b border-slate-800 pb-4 mb-4">
                    <div>
                      <span className="text-[10px] font-mono text-emerald-400 block">[AUDIT TARGET FIELD INTEL]</span>
                      <h3 className="text-xl font-black text-white mt-1">🏪 {selectedPharmacy.name}</h3>
                      <p className="text-[11px] text-slate-500 font-mono mt-0.5">{selectedPharmacy.coordinates}</p>
                    </div>
                  </div>

                  <div className="flex justify-between items-center mb-3">
                    <h4 className="text-xs font-bold text-slate-400 uppercase tracking-wider font-mono">// Intercept Registry Logs (Live Dynamic Clock Linked)</h4>
                  </div>

                  <div className="overflow-hidden rounded-xl border border-slate-900 bg-slate-950/40">
                    <div className="overflow-y-auto max-h-[180px] divide-y divide-slate-900/80">
                      {logs
                        .filter(log => log.pharmacyId === selectedPharmacy.id) 
                        .map((log) => (
                          <div key={log.id} className="p-3 transition flex flex-col sm:flex-row justify-between items-start sm:items-center gap-2 text-xs">
                            <div className="space-y-0.5">
                              <div className="flex items-center gap-2">
                                <span className="font-mono text-amber-400 text-[10px]">{log.date}</span>
                                <span className="text-white font-bold">{log.drug}</span>
                              </div>
                              <p className="text-slate-400 text-[11px] font-mono">{log.info}</p>
                            </div>
                            <span className={`font-mono text-[9px] font-bold px-2 py-0.5 rounded border ${log.color}`}>
                              {log.status}
                            </span>
                          </div>
                        ))}
                    </div>
                  </div>
                </div>
              </div>

            </div>
          </div>
        )}

      </main>

      <footer className="max-w-7xl mx-auto border-t border-slate-900 mt-16 pt-6 text-[11px] font-mono text-slate-600 tracking-widest text-center">
        <span>MEDISHIELD SECURITY SYSTEMS // 2026</span>
      </footer>

    </div>
  );
}
