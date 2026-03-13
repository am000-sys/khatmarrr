import React, { useState, useEffect, useMemo } from 'react';
import { 
  BookOpen, 
  Calendar, 
  Moon, 
  Printer, 
  Calculator, 
  CheckCircle2, 
  Clock, 
  Target,
  Star,
  Sparkles,
  LayoutGrid,
  List,
  Plus,
  Minus,
  Info,
  Share2,
  Bookmark
} from 'lucide-react';
import { motion, AnimatePresence } from 'motion/react';

// --- Types ---

interface PlanDay {
  night: number;
  startPage: number;
  endPage: number;
  pagesCount: number;
  juzText: string;
  isOdd: boolean;
  isLastTen: boolean;
  distribution: {
    fajr: number;
    isha: number;
    tarawih: number;
    qiyam: number;
  };
}

interface Stats {
  pagesDone: number;
  pagesLeft: number;
  nightsLeft: number;
  pagesPerNight: number;
  pagesPerRakat: string;
  progressPct: number;
}

// --- Constants ---

const TOTAL_PAGES = 604;
const ODD_NIGHTS = [21, 23, 25, 27, 29];

// --- Helper Functions ---

const getJuz = (p: number) => Math.min(Math.ceil(p / 20), 30);

const getHijriDate = () => {
  const now = new Date();
  const h = new Intl.DateTimeFormat('ar-SA-u-ca-islamic', {
    day: 'numeric',
    month: 'long',
    year: 'numeric'
  }).format(now);
  const g = now.toLocaleDateString('ar-SA');
  return { g, h };
};

export default function App() {
  // --- State ---
  const [currentPage, setCurrentPage] = useState<number>(326);
  const [currentNight, setCurrentNight] = useState<number>(24);
  const [finishNight, setFinishNight] = useState<number>(29);
  
  const [fajr, setFajr] = useState<number>(2);
  const [isha, setIsha] = useState<number>(2);
  const [tarawih, setTarawih] = useState<number>(10);
  const [qiyam, setQiyam] = useState<number>(0);

  const [viewMode, setViewMode] = useState<'grid' | 'list'>('list');
  const [showCopyToast, setShowCopyToast] = useState(false);
  const [isSetupComplete, setIsSetupComplete] = useState<boolean>(false);
  const [setupStep, setSetupStep] = useState<number>(1);

  // --- Persistence ---
  useEffect(() => {
    if (showCopyToast) {
      const timer = setTimeout(() => setShowCopyToast(false), 3000);
      return () => clearTimeout(timer);
    }
  }, [showCopyToast]);

  useEffect(() => {
    const saved = localStorage.getItem('khatmati_taseel_data');
    const setupStatus = localStorage.getItem('khatmati_setup_complete');
    
    if (setupStatus === 'true') {
      setIsSetupComplete(true);
    }

    if (saved) {
      try {
        const data = JSON.parse(saved);
        setCurrentPage(data.currentPage ?? 326);
        setCurrentNight(data.currentNight ?? 24);
        setFinishNight(data.finishNight ?? 29);
        setFajr(data.fajr ?? 2);
        setIsha(data.isha ?? 2);
        setTarawih(data.tarawih ?? 10);
        setQiyam(data.qiyam ?? 0);
      } catch (e) {
        console.error("Failed to load saved data", e);
      }
    }
  }, []);

  useEffect(() => {
    const data = { currentPage, currentNight, finishNight, fajr, isha, tarawih, qiyam };
    localStorage.setItem('khatmati_taseel_data', JSON.stringify(data));
    localStorage.setItem('khatmati_setup_complete', isSetupComplete.toString());
  }, [currentPage, currentNight, finishNight, fajr, isha, tarawih, qiyam, isSetupComplete]);

  const applyMethodology = (type: 'balanced' | 'qiyam-heavy') => {
    if (type === 'balanced') {
      setTarawih(10);
      setQiyam(10);
      setFajr(2);
      setIsha(2);
    } else {
      setTarawih(6);
      setQiyam(14);
      setFajr(2);
      setIsha(2);
    }
    setIsSetupComplete(true);
  };

  // --- Calculations ---
  const { plan, stats } = useMemo(() => {
    const nightsCount = Math.max(1, finishNight - currentNight + 1);
    const pagesLeft = Math.max(0, TOTAL_PAGES - currentPage + 1);
    const dailyAverage = Math.ceil(pagesLeft / nightsCount);
    const totalPreferred = fajr + isha + tarawih + qiyam;
    const ppr = totalPreferred > 0 ? (dailyAverage / totalPreferred).toFixed(2) : '—';
    const progress = Math.round(((currentPage - 1) / TOTAL_PAGES) * 100);

    const statsObj: Stats = {
      pagesDone: currentPage - 1,
      pagesLeft,
      nightsLeft: nightsCount,
      pagesPerNight: dailyAverage,
      pagesPerRakat: ppr,
      progressPct: progress
    };

    const planArr: PlanDay[] = [];
    let start = currentPage;

    for (let i = currentNight; i <= finishNight; i++) {
      const isLast = i === finishNight;
      const pagesForDay = isLast ? (TOTAL_PAGES - start + 1) : dailyAverage;
      const end = Math.min(start + pagesForDay - 1, TOTAL_PAGES);
      
      const j1 = getJuz(start);
      const j2 = getJuz(end);
      const juzText = j1 === j2 ? `${j1}` : `${j1}–${j2}`;

      // Calculate distribution
      let distribution = { fajr: 0, isha: 0, tarawih: 0, qiyam: 0 };
      if (totalPreferred > 0) {
        const ratio = pagesForDay / totalPreferred;
        distribution = {
          fajr: Math.round(fajr * ratio),
          isha: Math.round(isha * ratio),
          tarawih: Math.round(tarawih * ratio),
          qiyam: Math.round(qiyam * ratio),
        };
        
        const currentTotal = distribution.fajr + distribution.isha + distribution.tarawih + distribution.qiyam;
        const diff = pagesForDay - currentTotal;
        if (diff !== 0) {
          if (distribution.tarawih > 0) distribution.tarawih += diff;
          else if (distribution.qiyam > 0) distribution.qiyam += diff;
          else if (distribution.fajr > 0) distribution.fajr += diff;
          else distribution.isha += diff;
        }
      }
      
      planArr.push({
        night: i,
        startPage: start,
        endPage: end,
        pagesCount: pagesForDay,
        juzText,
        isOdd: ODD_NIGHTS.includes(i),
        isLastTen: i >= 21,
        distribution
      });

      start = end + 1;
    }

    return { plan: planArr, stats: statsObj };
  }, [currentPage, currentNight, finishNight, fajr, isha, tarawih, qiyam]);

  const { g: gregorianDate, h: hijriDate } = useMemo(() => getHijriDate(), []);

  return (
    <div dir="rtl" className="min-h-screen bg-taseel relative overflow-x-hidden font-sans selection:bg-[#d48c5c]/30">
      {/* Decorative Top Bar */}
      <div className="h-2 bg-[#2c4c64] w-full" />

      <AnimatePresence mode="wait">
        {!isSetupComplete ? (
          <motion.div
            key="setup"
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, y: -20 }}
            className="relative max-w-2xl mx-auto px-6 py-20 min-h-[80vh] flex flex-col justify-center"
          >
            <div className="text-center mb-12 space-y-4">
              <div className="flex items-center justify-center gap-4 mb-6">
                <div className="w-12 h-12 bg-[#d48c5c] rotate-45 flex items-center justify-center">
                  <Star size={24} className="text-white -rotate-45" fill="currentColor" />
                </div>
                <h1 className="text-4xl font-serif font-bold text-[#2c4c64]">ختمتي</h1>
              </div>
              <h2 className="text-2xl font-bold text-[#2c4c64]">أهلاً بك يا إمام</h2>
              <p className="text-[#d48c5c] font-medium">لنقم بإعداد خطة الختم الخاصة بك في خطوات بسيطة</p>
            </div>

            <div className="card-taseel p-10 space-y-10">
              {setupStep === 1 ? (
                <div className="space-y-8">
                  <SectionHeader title="المسار الحالي" />
                  <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <TaseelInput label="الصفحة الحالية" value={currentPage} onChange={setCurrentPage} min={1} max={604} />
                    <TaseelInput label="الليلة الحالية" value={currentNight} onChange={setCurrentNight} min={1} max={30} />
                    <TaseelInput label="ليلة الختم" value={finishNight} onChange={setFinishNight} min={1} max={30} />
                  </div>
                  <button
                    onClick={() => setSetupStep(2)}
                    className="w-full bg-[#2c4c64] text-white font-bold py-5 rounded-2xl shadow-xl hover:bg-[#2c4c64]/90 transition-all flex items-center justify-center gap-3"
                  >
                    التالي
                  </button>
                </div>
              ) : (
                <div className="space-y-8">
                  <SectionHeader title="اختر منهجية التوزيع" />
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <button
                      onClick={() => applyMethodology('balanced')}
                      className="p-8 rounded-2xl border-2 border-[#2c4c64]/10 hover:border-[#d48c5c] hover:bg-[#d48c5c]/5 transition-all text-right space-y-3 group"
                    >
                      <div className="w-10 h-10 rounded-full bg-[#2c4c64]/5 flex items-center justify-center text-[#2c4c64] group-hover:bg-[#d48c5c] group-hover:text-white transition-colors">
                        <Calculator size={20} />
                      </div>
                      <h4 className="font-bold text-[#2c4c64]">الموازنة بين التراويح والقيام</h4>
                      <p className="text-xs text-[#2c4c64]/60 leading-relaxed">توزيع متساوٍ تقريباً بين صلاتي التراويح والقيام (مثال: 10 أوجه لكل منهما).</p>
                    </button>

                    <button
                      onClick={() => applyMethodology('qiyam-heavy')}
                      className="p-8 rounded-2xl border-2 border-[#2c4c64]/10 hover:border-[#d48c5c] hover:bg-[#d48c5c]/5 transition-all text-right space-y-3 group"
                    >
                      <div className="w-10 h-10 rounded-full bg-[#2c4c64]/5 flex items-center justify-center text-[#2c4c64] group-hover:bg-[#d48c5c] group-hover:text-white transition-colors">
                        <Moon size={20} />
                      </div>
                      <h4 className="font-bold text-[#2c4c64]">الإطالة في القيام</h4>
                      <p className="text-xs text-[#2c4c64]/60 leading-relaxed">تخصيص الثقل الأكبر لصلاة القيام (مثال: 6 أوجه تراويح و 14 وجهاً للقيام).</p>
                    </button>
                  </div>
                  <button
                    onClick={() => setSetupStep(1)}
                    className="w-full text-[#2c4c64]/60 font-bold py-2 hover:text-[#2c4c64] transition-all"
                  >
                    العودة للخطوة السابقة
                  </button>
                </div>
              )}
            </div>
          </motion.div>
        ) : (
          <motion.div
            key="dashboard"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            className="relative max-w-5xl mx-auto px-6 py-12"
          >
            {/* Header Section */}
            <header className="flex flex-col items-center mb-16 space-y-8">
              <div className="flex justify-between w-full items-start">
                <button 
                  onClick={() => {
                    setIsSetupComplete(false);
                    setSetupStep(1);
                  }}
                  className="text-xs font-bold text-[#d48c5c] flex items-center gap-2 hover:underline"
                >
                  <Target size={14} />
                  تعديل الخطة الأساسية
                </button>
              </div>
              <motion.div
                initial={{ opacity: 0, scale: 0.9 }}
                animate={{ opacity: 1, scale: 1 }}
                className="text-center space-y-4"
              >
                <div className="flex items-center justify-center gap-4">
                  <h1 className="text-4xl md:text-5xl font-serif font-bold text-[#2c4c64] tracking-tight">ختمتي</h1>
                  <div className="w-8 h-8 bg-[#d48c5c] rotate-45 flex items-center justify-center">
                    <Star size={16} className="text-white -rotate-45" fill="currentColor" />
                  </div>
                </div>
                <p className="text-lg text-[#d48c5c] font-bold">حاسبة المحراب لخطة ختم القرآن الكريم</p>
              </motion.div>

              <div className="flex items-center justify-center gap-4 w-full">
                <div className="h-px flex-1 bg-[#2c4c64]/10" />
                <div className="text-xs text-[#2c4c64]/60 font-mono bg-white/50 px-4 py-1 rounded-full border border-[#2c4c64]/5">
                  {hijriDate} | {gregorianDate}
                </div>
                <div className="h-px flex-1 bg-[#2c4c64]/10" />
              </div>
            </header>

        <div className="grid grid-cols-1 lg:grid-cols-12 gap-10">
          {/* Left Column: Controls & Stats */}
          <div className="lg:col-span-5 space-y-8">
            {/* Stats Grid */}
            <div className="grid grid-cols-2 gap-4">
              <StatCard label="صفحات مقروءة" value={stats.pagesDone} color="blue" />
              <StatCard label="صفحات متبقية" value={stats.pagesLeft} color="orange" />
              <StatCard label="ليالٍ متبقية" value={stats.nightsLeft} color="blue" />
              <StatCard label="صفحات / ليلة" value={stats.pagesPerNight} color="orange" />
            </div>

            {/* Progress Section */}
            <div className="card-taseel p-8 space-y-6">
              <div className="flex items-center justify-between">
                <h3 className="text-lg font-bold text-[#2c4c64] flex items-center gap-2">
                  <Sparkles size={18} className="text-[#d48c5c]" />
                  مسار الإنجاز
                </h3>
                <span className="text-2xl font-bold text-[#d48c5c]">{stats.progressPct}%</span>
              </div>
              <div className="h-2 bg-[#2c4c64]/5 rounded-full overflow-hidden">
                <motion.div 
                  initial={{ width: 0 }}
                  animate={{ width: `${stats.progressPct}%` }}
                  className="h-full bg-[#d48c5c]"
                />
              </div>
            </div>

            {/* Settings Form */}
            <div className="card-taseel p-8 space-y-8">
              <div className="space-y-6">
                <SectionHeader title="الجدول الزمني" />
                <div className="grid grid-cols-3 gap-4">
                  <TaseelInput label="الصفحة" value={currentPage} onChange={setCurrentPage} min={1} max={604} />
                  <TaseelInput label="الليلة" value={currentNight} onChange={setCurrentNight} min={1} max={30} />
                  <TaseelInput label="الختم" value={finishNight} onChange={setFinishNight} min={1} max={30} />
                </div>
              </div>

              <div className="space-y-6">
                <SectionHeader title="عدد الأوجه" />
                <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
                  <TaseelInput label="الفجر" value={fajr} onChange={setFajr} min={0} />
                  <TaseelInput label="العشاء" value={isha} onChange={setIsha} min={0} />
                  <TaseelInput label="تراويح" value={tarawih} onChange={setTarawih} min={0} />
                  <TaseelInput label="قيام" value={qiyam} onChange={setQiyam} min={0} />
                </div>
              </div>

              <div className="flex gap-4 pt-4">
                <button 
                  onClick={() => window.print()}
                  className="p-4 rounded-xl border border-[#2c4c64]/10 text-[#2c4c64] hover:bg-[#2c4c64]/5 transition-all"
                  title="طباعة الخطة"
                >
                  <Printer size={20} />
                </button>
                <button 
                  onClick={() => {
                    const url = window.location.href;
                    navigator.clipboard.writeText(url).then(() => {
                      setShowCopyToast(true);
                    }).catch(() => {
                      // Fallback if clipboard fails
                    });
                  }}
                  className="flex-1 bg-[#2c4c64] text-white font-bold py-4 rounded-xl shadow-lg shadow-[#2c4c64]/20 hover:bg-[#2c4c64]/90 transition-all flex items-center justify-center gap-3 relative"
                >
                  <Share2 size={20} />
                  مشاركة الخطة
                  <AnimatePresence>
                    {showCopyToast && (
                      <motion.div
                        initial={{ opacity: 0, y: 10 }}
                        animate={{ opacity: 1, y: 0 }}
                        exit={{ opacity: 0 }}
                        className="absolute -top-12 left-1/2 -translate-x-1/2 bg-[#d48c5c] text-white text-[10px] py-2 px-4 rounded-full shadow-xl whitespace-nowrap"
                      >
                        تم نسخ الرابط بنجاح
                      </motion.div>
                    )}
                  </AnimatePresence>
                </button>
              </div>
            </div>
          </div>

          {/* Right Column: Plan Details */}
          <div className="lg:col-span-7 space-y-6">
            <div className="flex items-center justify-between px-2">
              <h2 className="text-2xl font-serif font-bold text-[#2c4c64]">الخطة التفصيلية</h2>
              <div className="flex bg-[#2c4c64]/5 p-1 rounded-lg">
                <button 
                  onClick={() => setViewMode('list')}
                  className={`p-2 rounded-md transition-all ${viewMode === 'list' ? 'bg-[#2c4c64] text-white' : 'text-[#2c4c64]/40'}`}
                >
                  <List size={16} />
                </button>
                <button 
                  onClick={() => setViewMode('grid')}
                  className={`p-2 rounded-md transition-all ${viewMode === 'grid' ? 'bg-[#2c4c64] text-white' : 'text-[#2c4c64]/40'}`}
                >
                  <LayoutGrid size={16} />
                </button>
              </div>
            </div>

            <div className={`
              ${viewMode === 'grid' ? 'grid grid-cols-1 md:grid-cols-2 gap-4' : 'space-y-4'}
            `}>
              <AnimatePresence mode="popLayout">
                {plan.map((day, idx) => (
                  <motion.div
                    key={day.night}
                    initial={{ opacity: 0, y: 10 }}
                    animate={{ opacity: 1, y: 0 }}
                    transition={{ delay: idx * 0.03 }}
                    className={`
                      card-taseel p-6 relative overflow-hidden group
                      ${day.isLastTen ? 'border-[#d48c5c]/30' : ''}
                    `}
                  >
                    <div className="flex flex-col gap-4">
                      <div className="flex items-center justify-between">
                        <div className="flex items-center gap-4">
                          <div className={`
                            w-12 h-12 rounded-lg flex items-center justify-center text-lg font-bold
                            ${day.isOdd ? 'bg-[#d48c5c] text-white' : 'bg-[#2c4c64]/5 text-[#2c4c64]'}
                          `}>
                            {day.night}
                          </div>
                          <div className="space-y-1">
                            <div className="flex items-center gap-2">
                              <span className="text-lg font-bold text-[#2c4c64]">الليلة {day.night}</span>
                              {day.isOdd && <span className="text-[10px] bg-[#d48c5c]/10 text-[#d48c5c] px-2 py-0.5 rounded-full font-bold">وتر</span>}
                            </div>
                            <div className="text-xs text-[#2c4c64]/40 flex items-center gap-1">
                              <Bookmark size={10} />
                              الجزء {day.juzText}
                            </div>
                          </div>
                        </div>

                        <div className="text-right">
                          <div className="text-xl font-bold text-[#2c4c64]">{day.pagesCount} <span className="text-[10px] font-normal opacity-50">صفحة</span></div>
                          <div className="text-[10px] text-[#d48c5c] font-bold uppercase tracking-wider">
                            {day.startPage} ← {day.endPage}
                          </div>
                        </div>
                      </div>

                      {/* Prayer Distribution */}
                      <div className="grid grid-cols-4 gap-2 pt-4 border-t border-[#2c4c64]/5">
                        <div className="flex flex-col items-center p-1.5 bg-[#2c4c64]/5 rounded-lg">
                          <span className="text-[8px] opacity-40 uppercase font-bold">الفجر</span>
                          <span className="text-xs font-bold text-[#2c4c64]">{day.distribution.fajr}</span>
                        </div>
                        <div className="flex flex-col items-center p-1.5 bg-[#2c4c64]/5 rounded-lg">
                          <span className="text-[8px] opacity-40 uppercase font-bold">العشاء</span>
                          <span className="text-xs font-bold text-[#2c4c64]">{day.distribution.isha}</span>
                        </div>
                        <div className="flex flex-col items-center p-1.5 bg-[#2c4c64]/5 rounded-lg">
                          <span className="text-[8px] opacity-40 uppercase font-bold">تراويح</span>
                          <span className="text-xs font-bold text-[#2c4c64]">{day.distribution.tarawih}</span>
                        </div>
                        <div className="flex flex-col items-center p-1.5 bg-[#2c4c64]/5 rounded-lg">
                          <span className="text-[8px] opacity-40 uppercase font-bold">قيام</span>
                          <span className="text-xs font-bold text-[#2c4c64]">{day.distribution.qiyam}</span>
                        </div>
                      </div>
                    </div>
                  </motion.div>
                ))}
              </AnimatePresence>
            </div>

            {/* Footer Bar */}
            <div className="bg-[#2c4c64] text-white p-8 rounded-2xl flex flex-col md:flex-row items-center justify-between gap-6 shadow-xl relative overflow-hidden">
              <div className="absolute top-0 right-0 w-32 h-32 bg-white/5 rotate-45 -mr-16 -mt-16" />
              <div className="flex items-center gap-4 relative z-10">
                <div className="w-12 h-12 rounded-full bg-white/10 flex items-center justify-center">
                  <Info size={24} />
                </div>
                <div className="text-center md:text-right">
                  <p className="font-bold">تقبل الله طاعتكم</p>
                  <p className="text-[10px] opacity-60">خطة الختم مبنية على المصحف الورقي (604 صفحة)</p>
                </div>
              </div>
              <div className="flex gap-4 relative z-10">
                <div className="flex flex-col items-center">
                  <span className="text-[10px] opacity-60 uppercase tracking-widest">المعدل</span>
                  <span className="text-xl font-bold">{stats.pagesPerNight}</span>
                </div>
                <div className="w-px h-10 bg-white/10" />
                <div className="flex flex-col items-center">
                  <span className="text-[10px] opacity-60 uppercase tracking-widest">المتبقي</span>
                  <span className="text-xl font-bold">{stats.pagesLeft}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </motion.div>
    )}
  </AnimatePresence>

      {/* Social Links Bar */}
      <div className="max-w-5xl mx-auto px-6 pb-12">
        <div className="bg-[#2c4c64]/5 border border-[#2c4c64]/10 rounded-2xl p-4 flex flex-wrap justify-center gap-8 text-[#2c4c64]/60 text-sm">
          <div className="flex items-center gap-2">
            <Share2 size={16} />
            <span>ختمتي</span>
          </div>
          <div className="flex items-center gap-2">
            <Star size={16} />
            <span>حاسبة الختم</span>
          </div>
        </div>
      </div>

      <style dangerouslySetInnerHTML={{ __html: `
        @media print {
          body { background: white !important; padding: 0 !important; }
          .bg-taseel-blue { background-color: black !important; color: white !important; }
          .text-taseel-blue { color: black !important; }
          .text-taseel-orange { color: #666 !important; }
          .card-taseel { border: 1px solid #ddd !important; box-shadow: none !important; }
          .lg\\:col-span-5, .no-print { display: none !important; }
          .lg\\:col-span-7 { width: 100% !important; }
        }
      `}} />
    </div>
  );
}

// --- Sub-components ---

function StatCard({ label, value, color }: { label: string, value: number | string, color: 'blue' | 'orange' }) {
  const styles = {
    blue: 'border-[#2c4c64]/10 text-[#2c4c64] bg-[#2c4c64]/5',
    orange: 'border-[#d48c5c]/10 text-[#d48c5c] bg-[#d48c5c]/5'
  };

  return (
    <div className={`card-taseel p-5 border-2 ${styles[color]} flex flex-col items-center justify-center text-center`}>
      <span className="text-[10px] font-bold uppercase tracking-widest mb-1 opacity-60">{label}</span>
      <span className="text-2xl font-bold tabular-nums">{value}</span>
    </div>
  );
}

function SectionHeader({ title }: { title: string }) {
  return (
    <div className="flex items-center gap-3">
      <div className="w-1 h-6 bg-[#d48c5c] rounded-full" />
      <h3 className="font-bold text-[#2c4c64]">{title}</h3>
    </div>
  );
}

function TaseelInput({ label, value, onChange, min, max }: { 
  label: string, 
  value: number, 
  onChange: (v: number) => void,
  min?: number,
  max?: number
}) {
  return (
    <div className="space-y-2">
      <label className="block text-[10px] font-bold text-[#2c4c64]/60 uppercase tracking-widest pr-1">{label}</label>
      <div className="relative group">
        <input 
          type="number" 
          value={value}
          onChange={(e) => onChange(parseInt(e.target.value) || 0)}
          min={min}
          max={max}
          className="w-full bg-[#2c4c64]/5 border border-[#2c4c64]/10 rounded-xl px-4 py-3 text-lg text-[#2c4c64] focus:outline-none focus:border-[#d48c5c] focus:ring-4 focus:ring-[#d48c5c]/5 transition-all text-center font-bold"
        />
        <div className="absolute left-2 top-1/2 -translate-y-1/2 flex flex-col gap-1 opacity-0 group-hover:opacity-100 transition-opacity">
          <button onClick={() => onChange(value + 1)} className="p-1 hover:text-[#d48c5c] text-[#2c4c64]/40"><Plus size={12} /></button>
          <button onClick={() => onChange(Math.max(min ?? 0, value - 1))} className="p-1 hover:text-[#d48c5c] text-[#2c4c64]/40"><Minus size={12} /></button>
        </div>
      </div>
    </div>
  );
}
