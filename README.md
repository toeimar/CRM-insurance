# CRM-insurance
CRM insurance v.1
```react
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, signInWithCustomToken, signInAnonymously, 
  signOut, onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, collection, doc, setDoc, getDocs, 
  updateDoc, deleteDoc, onSnapshot, serverTimestamp 
} from 'firebase/firestore';
import { 
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip as RechartsTooltip, Legend, 
  ResponsiveContainer, PieChart, Pie, Cell, LineChart, Line 
} from 'recharts';
import { 
  LayoutDashboard, Users, UserPlus, CalendarDays, RefreshCw, 
  TrendingUp, BarChart3, Bot, Settings, LogOut, Menu, X, 
  Search, Bell, Plus, Phone, Mail, Calendar, FileText, 
  ChevronRight, Award, Target, Clock, AlertCircle, Sparkles, CheckCircle2,
  ShieldCheck, Edit2, Save, Trash2, Info, Cake, ClipboardList
} from 'lucide-react';

// ==========================================
// 1. FIREBASE & APP SETUP
// ==========================================
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'insurance-crm-app';

// ==========================================
// 2. CONSTANTS & THEME
// ==========================================
const COLORS = ['#0ea5e9', '#f59e0b', '#10 b981', '#6366f1', '#f43f5e', '#8b5cf6'];
const STATUS_OPTIONS = ['Lead New', 'Contacted', 'Appointment', 'Proposal Sent', 'Waiting Decision', 'Closed Sale', 'Follow Later', 'Inactive'];
const PAYMENT_CYCLES = [
  { value: 'monthly', label: 'รายเดือน' },
  { value: 'quarterly', label: 'ราย 3 เดือน' },
  { value: 'half-yearly', label: 'ราย 6 เดือน' },
  { value: 'yearly', label: 'รายปี' }
];

// ==========================================
// 3. MAIN APP COMPONENT
// ==========================================
export default function App() {
  const [user, setUser] = useState(null);
  const [authLoading, setAuthLoading] = useState(true);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      setAuthLoading(false);
    });
    return () => unsubscribe();
  }, []);

  if (authLoading) return <div className="min-h-screen flex items-center justify-center bg-[#FDFBF7]"><div className="animate-spin rounded-full h-12 w-12 border-b-2 border-sky-600"></div></div>;

  return user ? <MainWorkspace user={user} /> : <AuthScreen />;
}

// ==========================================
// 4. AUTHENTICATION SCREEN
// ==========================================
function AuthScreen() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleLogin = async () => {
    setLoading(true);
    setError('');
    try {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    } catch (err) {
      console.error(err);
      setError('ไม่สามารถเชื่อมต่อระบบยืนยันตัวตนได้ กรุณาลองใหม่อีกครั้ง');
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-[#FDFBF7] p-4 font-sans">
      <div className="max-w-md w-full bg-white rounded-2xl shadow-xl border border-slate-100 overflow-hidden">
        <div className="bg-sky-600 p-8 text-center relative overflow-hidden">
          <div className="absolute top-0 right-0 -mt-10 -mr-10 w-40 h-40 bg-white opacity-10 rounded-full blur-2xl"></div>
          <Award className="h-14 w-14 text-amber-400 mx-auto mb-4" />
          <h2 className="text-3xl font-bold text-white tracking-tight">Agent CRM Pro</h2>
          <p className="text-sky-100 mt-2 text-sm">ระบบบริหารงานขาย AI สำหรับตัวแทนมืออาชีพ</p>
        </div>
        <div className="p-8 text-center">
          <div className="bg-sky-50 rounded-xl p-4 mb-6 border border-sky-100 flex items-start text-left">
            <ShieldCheck className="h-5 w-5 text-sky-600 mt-0.5 mr-3 shrink-0" />
            <p className="text-sm text-sky-800">
              เข้าใช้งานได้ทันที ข้อมูลของคุณจะถูกเก็บเป็นความลับแบบ Private Session
            </p>
          </div>
          
          {error && <div className="mb-4 p-3 bg-red-50 text-red-600 rounded-lg text-sm border border-red-100">{error}</div>}
          
          <button 
            onClick={handleLogin} 
            disabled={loading} 
            className="w-full bg-sky-600 hover:bg-sky-700 text-white font-semibold py-3.5 rounded-xl transition-all shadow-md hover:shadow-lg flex justify-center items-center group"
          >
            {loading ? (
              <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white"></div>
            ) : (
              <>
                <span>เข้าสู่ระบบ Workspace</span>
                <ChevronRight className="h-5 w-5 ml-2 group-hover:translate-x-1 transition-transform" />
              </>
            )}
          </button>
        </div>
      </div>
    </div>
  );
}

// ==========================================
// 5. MAIN WORKSPACE
// ==========================================
function MainWorkspace({ user }) {
  const [currentView, setCurrentView] = useState('home');
  const [isMobileMenuOpen, setIsMobileMenuOpen] = useState(false);
  const [customers, setCustomers] = useState([]);
  const [tasks, setTasks] = useState([]);
  const [goals, setGoals] = useState({ monthlyTarget: 100000, currentSales: 0 });
  const [loadingData, setLoadingData] = useState(true);

  const userPath = `artifacts/${appId}/users/${user.uid}`;
  const customersRef = collection(db, `${userPath}/customers`);
  const tasksRef = collection(db, `${userPath}/tasks`);
  const settingsRef = doc(db, `${userPath}/settings/profile`);

  useEffect(() => {
    if (!user) return;
    let isMounted = true;

    const unsubCustomers = onSnapshot(customersRef, (snapshot) => {
      if (!isMounted) return;
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      data.sort((a, b) => new Date(b.createdAt || 0) - new Date(a.createdAt || 0));
      setCustomers(data);
    }, (err) => console.error(err));

    const unsubTasks = onSnapshot(tasksRef, (snapshot) => {
      if (!isMounted) return;
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setTasks(data);
    }, (err) => console.error(err));

    const unsubSettings = onSnapshot(settingsRef, (docSnap) => {
      if (!isMounted) return;
      if (docSnap.exists()) {
        setGoals(prev => ({ ...prev, ...docSnap.data().goals }));
      }
      setLoadingData(false);
    }, (err) => console.error(err));

    return () => { 
      isMounted = false;
      unsubCustomers(); 
      unsubTasks(); 
      unsubSettings(); 
    };
  }, [user]);

  const navItems = [
    { id: 'home', label: 'หน้าหลัก', icon: LayoutDashboard },
    { id: 'dashboard', label: 'แดชบอร์ด', icon: BarChart3 },
    { id: 'add-lead', label: 'เพิ่ม Lead', icon: UserPlus },
    { id: 'customers', label: 'ลูกค้าทั้งหมด', icon: Users },
    { id: 'pipeline', label: 'Pipeline', icon: TrendingUp },
    { id: 'renewals', label: 'ต่ออายุกรมธรรม์', icon: RefreshCw },
    { id: 'ai-assistant', label: 'AI Assistant', icon: Bot },
    { id: 'settings', label: 'ตั้งค่า', icon: Settings },
  ];

  const renderView = () => {
    if (loadingData) return <div className="flex-1 flex items-center justify-center"><div className="animate-spin rounded-full h-8 w-8 border-b-2 border-sky-600"></div></div>;

    switch (currentView) {
      case 'home': return <HomeView customers={customers} tasks={tasks} goals={goals} userPath={userPath} setCurrentView={setCurrentView} />;
      case 'dashboard': return <DashboardView customers={customers} />;
      case 'add-lead': return <AddLeadView userPath={userPath} setCurrentView={setCurrentView} />;
      case 'customers': return <CustomersView customers={customers} userPath={userPath} />;
      case 'pipeline': return <PipelineView customers={customers} userPath={userPath} />;
      case 'renewals': return <RenewalsView customers={customers} />;
      case 'ai-assistant': return <AiAssistantView />;
      case 'settings': return <SettingsView goals={goals} userPath={userPath} user={user} />;
      default: return <HomeView customers={customers} tasks={tasks} goals={goals} userPath={userPath} setCurrentView={setCurrentView} />;
    }
  };

  const handleLogout = async () => {
    try { await signOut(auth); } catch (err) { console.error("Logout error", err); }
  };

  return (
    <div className="flex h-screen bg-[#FDFBF7] font-sans text-slate-800">
      <aside className={`fixed inset-y-0 left-0 z-50 w-64 bg-white border-r border-slate-200 transform transition-transform duration-300 ease-in-out ${isMobileMenuOpen ? 'translate-x-0' : '-translate-x-full'} md:relative md:translate-x-0`}>
        <div className="flex items-center justify-between h-16 px-6 border-b border-slate-100 bg-sky-600">
          <div className="flex items-center space-x-2 text-white">
            <Award className="h-6 w-6 text-amber-400" />
            <span className="text-lg font-bold tracking-wide">Agent CRM<span className="text-amber-400">Pro</span></span>
          </div>
          <button onClick={() => setIsMobileMenuOpen(false)} className="md:hidden text-white"><X className="h-5 w-5" /></button>
        </div>
        <div className="p-4 flex flex-col h-[calc(100vh-4rem)] overflow-y-auto">
          <div className="space-y-1 flex-1">
            {navItems.map((item) => {
              const Icon = item.icon;
              const isActive = currentView === item.id;
              return (
                <button
                  key={item.id}
                  onClick={() => { setCurrentView(item.id); setIsMobileMenuOpen(false); }}
                  className={`w-full flex items-center space-x-3 px-4 py-3 rounded-lg transition-colors ${isActive ? 'bg-sky-50 text-sky-700 font-semibold shadow-sm border border-sky-100' : 'text-slate-600 hover:bg-slate-50 hover:text-sky-600'}`}
                >
                  <Icon className={`h-5 w-5 ${isActive ? 'text-sky-600' : 'text-slate-400'}`} />
                  <span>{item.label}</span>
                </button>
              );
            })}
          </div>
          <div className="mt-auto pt-4 border-t border-slate-100">
            <button onClick={handleLogout} className="w-full flex items-center space-x-3 px-4 py-3 rounded-lg text-red-600 hover:bg-red-50 transition-colors">
              <LogOut className="h-5 w-5" />
              <span>ออกจากระบบ</span>
            </button>
          </div>
        </div>
      </aside>

      <main className="flex-1 flex flex-col overflow-hidden">
        <header className="h-16 bg-white border-b border-slate-200 flex items-center justify-between px-4 sm:px-6 z-10 shadow-sm">
          <div className="flex items-center">
            <button onClick={() => setIsMobileMenuOpen(true)} className="md:hidden mr-4 text-slate-500 hover:text-sky-600"><Menu className="h-6 w-6" /></button>
            <h1 className="text-xl font-semibold text-slate-800 capitalize hidden sm:block">
              {navItems.find(i => i.id === currentView)?.label}
            </h1>
          </div>
          <div className="flex items-center space-x-4">
            <div className="relative">
              <Bell className="h-5 w-5 text-slate-400 hover:text-sky-600 cursor-pointer transition-colors" />
              {customers.filter(c => c.status === 'Lead New').length > 0 && (
                <span className="absolute -top-1 -right-1 bg-red-500 text-white text-[10px] font-bold rounded-full h-4 w-4 flex items-center justify-center border-2 border-white">
                  {customers.filter(c => c.status === 'Lead New').length}
                </span>
              )}
            </div>
            <div className="flex items-center space-x-2 cursor-pointer" onClick={() => setCurrentView('settings')}>
              <div className="h-8 w-8 rounded-full bg-sky-100 flex items-center justify-center text-sky-700 font-bold border border-sky-200">A</div>
            </div>
          </div>
        </header>

        <div className="flex-1 overflow-auto bg-[#FDFBF7] p-4 sm:p-6 lg:p-8">
          {renderView()}
        </div>
      </main>
    </div>
  );
}

// ==========================================
// VIEWS COMPONENTS
// ==========================================

function HomeView({ customers, tasks, goals, userPath, setCurrentView }) {
  const currentMonthSales = customers
    .filter(c => c.status === 'Closed Sale' && c.updatedAt && new Date(c.updatedAt.seconds * 1000).getMonth() === new Date().getMonth())
    .reduce((sum, c) => sum + (Number(c.premium) || 0), 0);
  
  const progress = Math.min(100, Math.round((currentMonthSales / (goals.monthlyTarget || 1)) * 100));
  const todayTasks = tasks.filter(t => new Date(t.date).toDateString() === new Date().toDateString());
  const upcomingRenewals = customers.filter(c => c.renewalDate && new Date(c.renewalDate) > new Date() && new Date(c.renewalDate) < new Date(Date.now() + 30 * 24 * 60 * 60 * 1000));
  const newLeads = customers.filter(c => c.status === 'Lead New');

  // Birthday logic: 15 days ahead
  const upcomingBirthdays = customers.filter(c => {
    if (!c.dob) return false;
    const birthDate = new Date(c.dob);
    const today = new Date();
    const currentYear = today.getFullYear();
    const nextBirthday = new Date(currentYear, birthDate.getMonth(), birthDate.getDate());
    
    if (nextBirthday < today) nextBirthday.setFullYear(currentYear + 1);
    
    const diffTime = nextBirthday - today;
    const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    return diffDays >= 0 && diffDays <= 15;
  });

  const [newTaskText, setNewTaskText] = useState('');

  const addTask = async (e) => {
    e.preventDefault();
    if (!newTaskText) return;
    try {
      await setDoc(doc(collection(db, `${userPath}/tasks`)), {
        title: newTaskText,
        date: new Date().toISOString(),
        completed: false,
        priority: 'Medium',
        createdAt: serverTimestamp()
      });
      setNewTaskText('');
    } catch (err) { console.error(err); }
  };

  const toggleTask = async (task) => {
    try { await updateDoc(doc(db, `${userPath}/tasks/${task.id}`), { completed: !task.completed }); } catch (err) { console.error(err); }
  };

  const deleteTask = async (taskId) => {
    try { await deleteDoc(doc(db, `${userPath}/tasks/${taskId}`)); } catch (err) { console.error(err); }
  };

  return (
    <div className="space-y-6 max-w-7xl mx-auto">
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <KpiCard title="ยอดขายเดือนนี้" value={`฿${currentMonthSales.toLocaleString()}`} subtitle={`เป้าหมาย: ฿${(goals.monthlyTarget || 0).toLocaleString()}`} icon={Target} color="bg-sky-500" />
        <KpiCard title="% ความสำเร็จ" value={`${progress || 0}%`} subtitle="ของเป้าหมายรายเดือน" icon={Award} color="bg-emerald-500" />
        <KpiCard title="ลูกค้าทั้งหมด" value={customers.length} subtitle={`Lead ใหม่: ${newLeads.length} ราย`} icon={Users} color="bg-amber-500" />
        <KpiCard title="วันเกิดถัดไป" value={upcomingBirthdays.length} subtitle="ในอีก 15 วันข้างหน้า" icon={Cake} color="bg-pink-500" />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2 bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden flex flex-col h-full">
          <div className="px-6 py-4 border-b border-slate-100 flex justify-between items-center bg-slate-50">
            <h2 className="text-lg font-semibold flex items-center text-slate-800"><CalendarDays className="mr-2 h-5 w-5 text-sky-600" /> งานวันนี้ (Smart Planner)</h2>
          </div>
          <div className="p-6 flex-1">
            <form onSubmit={addTask} className="flex gap-2 mb-6">
              <input type="text" value={newTaskText} onChange={e => setNewTaskText(e.target.value)} placeholder="เพิ่มงานใหม่สำหรับวันนี้..." className="flex-1 px-4 py-2 border border-slate-200 rounded-lg focus:ring-2 focus:ring-sky-500 outline-none" />
              <button type="submit" className="bg-sky-600 text-white px-4 py-2 rounded-lg hover:bg-sky-700 flex items-center transition-colors"><Plus className="h-5 w-5" /></button>
            </form>
            <div className="space-y-3 overflow-y-auto max-h-[400px]">
              {todayTasks.length === 0 ? (
                <div className="text-center py-8 text-slate-400">
                  <CheckCircle2 className="h-12 w-12 mx-auto text-slate-200 mb-2" />
                  <p>ไม่มีงานค้างสำหรับวันนี้ ยอดเยี่ยม!</p>
                </div>
              ) : todayTasks.map(task => (
                <div key={task.id} className={`flex items-center p-3 rounded-lg border group transition-all ${task.completed ? 'bg-slate-50 border-slate-100' : 'bg-white border-slate-200 hover:border-sky-300 shadow-sm'}`}>
                  <button onClick={() => toggleTask(task)} className={`mr-3 h-6 w-6 rounded-full border-2 flex items-center justify-center transition-colors ${task.completed ? 'bg-emerald-500 border-emerald-500 text-white' : 'border-slate-300 hover:border-sky-400'}`}>
                    {task.completed && <CheckCircle2 className="h-4 w-4" />}
                  </button>
                  <span className={`flex-1 ${task.completed ? 'line-through text-slate-400' : 'text-slate-700 font-medium'}`}>{task.title}</span>
                  <button onClick={() => deleteTask(task.id)} className="opacity-0 group-hover:opacity-100 text-red-400 hover:text-red-600 p-1"><X className="h-4 w-4"/></button>
                </div>
              ))}
            </div>
          </div>
        </div>

        <div className="bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden flex flex-col h-full">
          <div className="px-6 py-4 border-b border-slate-100 bg-slate-50">
            <h2 className="text-lg font-semibold flex items-center text-slate-800"><Bell className="mr-2 h-5 w-5 text-amber-500" /> แจ้งเตือนด่วน</h2>
          </div>
          <div className="p-0 flex-1 overflow-y-auto max-h-[500px]">
            {upcomingRenewals.length === 0 && newLeads.length === 0 && upcomingBirthdays.length === 0 && (
              <div className="p-8 text-center text-slate-400">ไม่มีการแจ้งเตือนใหม่</div>
            )}
            <div className="divide-y divide-slate-100">
              {upcomingBirthdays.map(c => (
                <div key={c.id} className="p-4 hover:bg-pink-50 transition-colors flex items-start">
                  <Cake className="h-5 w-5 text-pink-500 mt-0.5 mr-3 shrink-0" />
                  <div>
                    <p className="text-sm font-medium text-slate-800">วันเกิดลูกค้าล่วงหน้า</p>
                    <p className="text-xs text-slate-500 mt-1">คุณ {c.name} - ({new Date(c.dob).toLocaleDateString('th-TH')})</p>
                  </div>
                </div>
              ))}
              {newLeads.slice(0, 5).map(c => (
                <div key={c.id} onClick={() => setCurrentView('pipeline')} className="p-4 hover:bg-slate-50 transition-colors flex items-start cursor-pointer">
                  <Sparkles className="h-5 w-5 text-sky-500 mt-0.5 mr-3 shrink-0" />
                  <div>
                    <p className="text-sm font-medium text-slate-800">Lead ใหม่รอติดต่อ</p>
                    <p className="text-xs text-slate-500 mt-1">คุณ {c.name} - เบอร์: {c.phone}</p>
                  </div>
                </div>
              ))}
              {upcomingRenewals.slice(0, 5).map(c => (
                <div key={c.id} onClick={() => setCurrentView('renewals')} className="p-4 hover:bg-slate-50 transition-colors flex items-start cursor-pointer">
                  <AlertCircle className="h-5 w-5 text-amber-500 mt-0.5 mr-3 shrink-0" />
                  <div>
                    <p className="text-sm font-medium text-slate-800">กรมธรรม์ใกล้หมดอายุ</p>
                    <p className="text-xs text-slate-500 mt-1">คุณ {c.name} - หมดอายุ: {new Date(c.renewalDate).toLocaleDateString('th-TH')}</p>
                  </div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

function KpiCard({ title, value, subtitle, icon: Icon, color }) {
  return (
    <div className="bg-white rounded-xl p-6 shadow-sm border border-slate-200 flex items-start space-x-4 hover:shadow-md transition-shadow">
      <div className={`${color} p-3 rounded-xl text-white shadow-sm`}>
        <Icon className="h-6 w-6" />
      </div>
      <div>
        <p className="text-sm font-medium text-slate-500">{title}</p>
        <h3 className="text-2xl font-bold text-slate-800 mt-1">{value}</h3>
        {subtitle && <p className="text-xs text-slate-400 mt-1">{subtitle}</p>}
      </div>
    </div>
  );
}

function DashboardView({ customers }) {
  const statusCounts = customers.reduce((acc, curr) => {
    acc[curr.status] = (acc[curr.status] || 0) + 1;
    return acc;
  }, {});
  
  const funnelData = Object.keys(statusCounts).map(key => ({ name: key, value: statusCounts[key] })).sort((a,b) => b.value - a.value);

  const revenueData = [
    { name: 'ม.ค.', sales: 40000 }, { name: 'ก.พ.', sales: 30000 },
    { name: 'มี.ค.', sales: 50000 }, { name: 'เม.ย.', sales: 85000 },
    { name: 'พ.ค.', sales: 120000 }, { name: 'มิ.ย.', sales: 90000 },
  ];

  return (
    <div className="space-y-6 max-w-7xl mx-auto">
      <h2 className="text-2xl font-bold text-slate-800">Performance Dashboard</h2>
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-200">
          <h3 className="text-lg font-semibold text-slate-800 mb-4">แนวโน้มยอดขาย (Revenue Trend)</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <LineChart data={revenueData}>
                <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#e2e8f0"/>
                <XAxis dataKey="name" axisLine={false} tickLine={false} tick={{fill: '#64748b'}} />
                <YAxis axisLine={false} tickLine={false} tick={{fill: '#64748b'}} />
                <RechartsTooltip contentStyle={{borderRadius: '8px', border: 'none', boxShadow: '0 4px 6px -1px rgb(0 0 0 / 0.1)'}} />
                <Line type="monotone" dataKey="sales" stroke="#0ea5e9" strokeWidth={3} dot={{r: 4, fill: '#0ea5e9', strokeWidth: 2, stroke: '#fff'}} activeDot={{r: 6}} />
              </LineChart>
            </ResponsiveContainer>
          </div>
        </div>

        <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-200">
          <h3 className="text-lg font-semibold text-slate-800 mb-4">อัตราการเปลี่ยนสถานะ (Sales Funnel)</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={funnelData.length > 0 ? funnelData : [{name: 'No Data', value: 0}]} layout="vertical" margin={{ left: 40 }}>
                <CartesianGrid strokeDasharray="3 3" horizontal={false} stroke="#e2e8f0" />
                <XAxis type="number" axisLine={false} tickLine={false} />
                <YAxis dataKey="name" type="category" axisLine={false} tickLine={false} tick={{fill: '#475569', fontSize: 12}} />
                <RechartsTooltip cursor={{fill: '#f8fafc'}} contentStyle={{borderRadius: '8px'}} />
                <Bar dataKey="value" fill="#f59e0b" radius={[0, 4, 4, 0]}>
                  {funnelData.map((entry, index) => (
                    <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />
                  ))}
                </Bar>
              </BarChart>
            </ResponsiveContainer>
          </div>
        </div>
      </div>
    </div>
  );
}

function AddLeadView({ userPath, setCurrentView }) {
  const [formData, setFormData] = useState({
    name: '', phone: '', email: '', gender: 'Male', dob: '', 
    age: '', job: '', income: '', familyStatus: 'Single', kids: '0',
    interestPlan: '', source: 'Facebook', notes: '', status: 'Lead New',
    premium: '', renewalDate: '', appointmentDate: '',
    paymentCycle: 'yearly' // Default
  });
  const [saving, setSaving] = useState(false);

  const handleChange = (e) => setFormData({...formData, [e.target.name]: e.target.value});

  const handleSubmit = async (e) => {
    e.preventDefault();
    setSaving(true);
    try {
      const newDocRef = doc(collection(db, `${userPath}/customers`));
      await setDoc(newDocRef, {
        ...formData,
        age: Number(formData.age) || null,
        premium: Number(formData.premium) || 0,
        createdAt: serverTimestamp(),
        updatedAt: serverTimestamp(),
      });
      setCurrentView('customers');
    } catch (err) {
      console.error(err);
    } finally {
      setSaving(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden">
      <div className="px-6 py-5 border-b border-slate-100 bg-sky-50 flex justify-between items-center">
        <div>
          <h2 className="text-xl font-bold text-sky-900">เพิ่ม Lead / ลูกค้าใหม่</h2>
          <p className="text-sm text-sky-700 mt-1">บันทึกข้อมูลและระบุสถานะเบื้องต้น</p>
        </div>
        <UserPlus className="h-8 w-8 text-sky-600 opacity-50" />
      </div>
      <form onSubmit={handleSubmit} className="p-6 md:p-8 space-y-8">
        <div>
          <h3 className="text-sm font-semibold text-slate-400 uppercase tracking-wider mb-4 border-b pb-2 flex items-center"><Users className="h-4 w-4 mr-2"/> ข้อมูลส่วนตัว</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">ชื่อ-นามสกุล <span className="text-red-500">*</span></label>
              <input required type="text" name="name" value={formData.name} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">เบอร์โทรศัพท์ <span className="text-red-500">*</span></label>
              <input required type="tel" name="phone" value={formData.phone} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">อายุ</label>
              <input type="number" name="age" value={formData.age} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">วันเกิด</label>
              <input type="date" name="dob" value={formData.dob} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
          </div>
        </div>

        <div>
          <h3 className="text-sm font-semibold text-slate-400 uppercase tracking-wider mb-4 border-b pb-2 flex items-center"><ShieldCheck className="h-4 w-4 mr-2"/> ข้อมูลกรมธรรม์และการเงิน</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">ชื่อแผนประกันที่สนใจ/ที่ถือครอง</label>
              <input type="text" name="interestPlan" value={formData.interestPlan} onChange={handleChange} placeholder="เช่น แผนประกันชีวิต 10/1" className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">เบี้ยประกัน (Premium)</label>
              <input type="number" name="premium" value={formData.premium} onChange={handleChange} placeholder="0.00" className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">งวดการชำระเบี้ย</label>
              <select name="paymentCycle" value={formData.paymentCycle} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none">
                {PAYMENT_CYCLES.map(cycle => (
                  <option key={cycle.value} value={cycle.value}>{cycle.label}</option>
                ))}
              </select>
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">วันครบกำหนดชำระเบี้ย (Renewal Date)</label>
              <input type="date" name="renewalDate" value={formData.renewalDate} onChange={handleChange} className="w-full px-4 py-2 bg-amber-50 border border-amber-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-amber-500 outline-none" />
            </div>
          </div>
        </div>

        <div>
          <h3 className="text-sm font-semibold text-slate-400 uppercase tracking-wider mb-4 border-b pb-2 flex items-center"><Target className="h-4 w-4 mr-2"/> แผนและการนัดหมาย</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
             <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">สถานะปัจจุบัน <span className="text-red-500">*</span></label>
              <select name="status" value={formData.status} onChange={handleChange} className="w-full px-4 py-2 bg-amber-50 border border-amber-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-amber-500 outline-none font-semibold text-amber-800">
                {STATUS_OPTIONS.map(s => <option key={s} value={s}>{s}</option>)}
              </select>
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">วันนัดหมาย (Appointment Date)</label>
              <input type="date" name="appointmentDate" value={formData.appointmentDate} onChange={handleChange} className="w-full px-4 py-2 bg-sky-50 border border-sky-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none" />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-1">แหล่งที่มา</label>
              <select name="source" value={formData.source} onChange={handleChange} className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none">
                <option value="Facebook">Facebook</option><option value="Line OA">Line OA</option>
                <option value="Referral">คนรู้จักแนะนำ</option><option value="Website">Website</option>
              </select>
            </div>
          </div>
        </div>

        <div>
          <h3 className="text-sm font-semibold text-slate-400 uppercase tracking-wider mb-4 border-b pb-2 flex items-center"><ClipboardList className="h-4 w-4 mr-2"/> หมายเหตุเพิ่มเติม</h3>
          <textarea name="notes" value={formData.notes} onChange={handleChange} placeholder="รายละเอียดเพิ่มเติม เช่น ความกังวลของลูกค้า หรือเงื่อนไขพิเศษ..." className="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-lg focus:bg-white focus:ring-2 focus:ring-sky-500 outline-none min-h-[100px]" />
        </div>

        <div className="flex justify-end space-x-4 pt-6 border-t border-slate-100">
          <button type="button" onClick={() => setCurrentView('home')} className="px-6 py-2.5 border border-slate-300 text-slate-700 rounded-lg hover:bg-slate-50 font-medium">ยกเลิก</button>
          <button type="submit" disabled={saving} className="px-6 py-2.5 bg-sky-600 text-white rounded-lg hover:bg-sky-700 font-medium shadow-sm flex items-center">
            {saving ? <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white mr-2"></div> : <span className="mr-2">บันทึกข้อมูล</span>}
            {!saving && <ChevronRight className="h-4 w-4"/>}
          </button>
        </div>
      </form>
    </div>
  );
}

function CustomerDetailModal({ customer, onClose, userPath }) {
  const [isEditing, setIsEditing] = useState(false);
  const [editData, setEditData] = useState({...customer});
  const [aiAnalysis, setAiAnalysis] = useState('');
  const [aiLoading, setAiLoading] = useState(false);

  const handleUpdate = async () => {
    try {
      await updateDoc(doc(db, `${userPath}/customers/${customer.id}`), {
        ...editData,
        age: Number(editData.age) || null,
        premium: Number(editData.premium) || 0,
        updatedAt: serverTimestamp()
      });
      setIsEditing(false);
    } catch (err) { console.error(err); }
  };

  const getAiAdvice = async () => {
    setAiLoading(true);
    setAiAnalysis('');
    const apiKey = ""; 
    const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
    
    const prompt = `วิเคราะห์ลูกค้า: ชื่อ: ${customer.name}, อายุ: ${customer.age || 'ไม่ระบุ'}, สถานะ: ${customer.status}, แผน: ${customer.interestPlan}. แนะนำเทคนิคการขาย`;

    try {
      const res = await fetch(url, {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] })
      });
      const data = await res.json();
      setAiAnalysis(data.candidates?.[0]?.content?.parts?.[0]?.text || "ไม่สามารถวิเคราะห์ได้");
    } catch (err) { setAiAnalysis("Error"); }
    finally { setAiLoading(false); }
  };

  if (!customer) return null;

  return (
    <div className="fixed inset-0 z-[100] flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div className="bg-white w-full max-w-5xl max-h-[90vh] rounded-2xl shadow-2xl overflow-hidden flex flex-col">
        <div className="bg-sky-600 px-6 py-4 flex justify-between items-center text-white shrink-0">
          <div className="flex items-center space-x-3">
            <Users className="h-5 w-5" />
            <h2 className="text-xl font-bold">{customer.name}</h2>
          </div>
          <button onClick={onClose} className="p-1 hover:bg-white/20 rounded-full transition-colors"><X className="h-6 w-6" /></button>
        </div>

        <div className="flex-1 overflow-y-auto p-6 md:p-8">
          <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
            <div className="space-y-6">
              <div className="flex justify-between items-center">
                <h3 className="text-lg font-bold text-slate-800 flex items-center"><Info className="h-5 w-5 mr-2 text-sky-600"/> รายละเอียด</h3>
                <button onClick={() => isEditing ? handleUpdate() : setIsEditing(true)} className="flex items-center px-4 py-2 rounded-lg text-sm bg-slate-100">
                  {isEditing ? <><Save className="h-4 w-4 mr-2" /> บันทึก</> : <><Edit2 className="h-4 w-4 mr-2" /> แก้ไข</>}
                </button>
              </div>

              <div className="grid grid-cols-2 gap-4">
                <DataField label="ชื่อ" value={editData.name} isEditing={isEditing} onChange={v => setEditData({...editData, name: v})} />
                <DataField label="เบอร์" value={editData.phone} isEditing={isEditing} onChange={v => setEditData({...editData, phone: v})} />
                <DataField label="อายุ" value={editData.age} type="number" isEditing={isEditing} onChange={v => setEditData({...editData, age: v})} />
                <DataField label="สถานะ" value={editData.status} isEditing={isEditing} isStatus onChange={v => setEditData({...editData, status: v})} />
                <DataField label="งวดการชำระ" value={editData.paymentCycle} isEditing={isEditing} isCycle onChange={v => setEditData({...editData, paymentCycle: v})} />
                <DataField label="วันครบกำหนดชำระ" value={editData.renewalDate} type="date" isEditing={isEditing} onChange={v => setEditData({...editData, renewalDate: v})} />
                <div className="col-span-2">
                  <label className="text-xs font-semibold text-slate-400 mb-1 block">หมายเหตุ</label>
                  {isEditing ? (
                    <textarea value={editData.notes || ''} onChange={e => setEditData({...editData, notes: e.target.value})} className="w-full border rounded-lg p-3 text-sm h-32 outline-none" />
                  ) : (
                    <p className="text-sm text-slate-600 bg-slate-50 p-4 rounded-lg italic min-h-[100px] whitespace-pre-wrap">
                      {editData.notes || 'ไม่มีบันทึก'}
                    </p>
                  )}
                </div>
              </div>
            </div>

            <div className="bg-slate-50 rounded-2xl p-6 border flex flex-col">
              <div className="flex items-center justify-between mb-4">
                <h3 className="text-lg font-bold flex items-center"><Bot className="h-6 w-6 mr-2 text-sky-600"/> AI Consultant</h3>
                <button onClick={getAiAdvice} disabled={aiLoading} className="bg-sky-600 text-white px-4 py-2 rounded-xl text-sm disabled:opacity-50">วิเคราะห์</button>
              </div>
              <div className="flex-1 bg-white rounded-xl p-4 overflow-y-auto max-h-[400px]">
                {aiLoading ? <div className="text-center py-10">วิเคราะห์...</div> : aiAnalysis ? <div className="prose prose-sm whitespace-pre-wrap">{aiAnalysis}</div> : <div className="text-center text-slate-400 py-10">กดปุ่มเพื่อเริ่มวิเคราะห์</div>}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

function DataField({ label, value, isEditing, onChange, type = "text", isStatus = false, isCycle = false }) {
  return (
    <div>
      <span className="text-xs font-semibold text-slate-400 uppercase mb-1">{label}</span>
      {isEditing ? (
        isStatus ? (
          <select value={value} onChange={e => onChange(e.target.value)} className="w-full border rounded-lg p-2 text-sm">
            {STATUS_OPTIONS.map(s => <option key={s} value={s}>{s}</option>)}
          </select>
        ) : isCycle ? (
          <select value={value} onChange={e => onChange(e.target.value)} className="w-full border rounded-lg p-2 text-sm">
            {PAYMENT_CYCLES.map(c => <option key={cycle.value} value={c.value}>{c.label}</option>)}
          </select>
        ) : (
          <input type={type} value={value || ''} onChange={e => onChange(e.target.value)} className="w-full border rounded-lg p-2 text-sm" />
        )
      ) : (
        <span className="text-sm font-medium block p-2">
          {isCycle ? (PAYMENT_CYCLES.find(c => c.value === value)?.label || value) : (value || '-')}
        </span>
      )}
    </div>
  );
}

function CustomersView({ customers, userPath }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [filterStatus, setFilterStatus] = useState('All');
  const [selectedCustomer, setSelectedCustomer] = useState(null);

  const filteredData = customers.filter(c => {
    const matchesSearch = c.name?.toLowerCase().includes(searchTerm.toLowerCase()) || c.phone?.includes(searchTerm);
    const matchesStatus = filterStatus === 'All' || c.status === filterStatus;
    return matchesSearch && matchesStatus;
  });

  return (
    <div className="space-y-6 max-w-7xl mx-auto">
      <div className="flex flex-col sm:flex-row gap-4 bg-white p-4 rounded-xl border">
        <div className="relative flex-1">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-5 w-5 text-slate-400" />
          <input type="text" placeholder="ค้นหา..." value={searchTerm} onChange={e => setSearchTerm(e.target.value)} className="w-full pl-10 pr-4 py-2 border rounded-lg outline-none" />
        </div>
        <select value={filterStatus} onChange={e => setFilterStatus(e.target.value)} className="border rounded-lg px-4 py-2 bg-white">
          <option value="All">ทุกสถานะ</option>
          {STATUS_OPTIONS.map(s => <option key={s} value={s}>{s}</option>)}
        </select>
      </div>

      <div className="bg-white rounded-xl border overflow-hidden">
        <table className="w-full text-left text-sm">
          <thead className="bg-slate-50 border-b font-semibold">
            <tr>
              <th className="px-6 py-4">ชื่อ</th>
              <th className="px-6 py-4">เบอร์</th>
              <th className="px-6 py-4">สถานะ</th>
              <th className="px-6 py-4">จัดการ</th>
            </tr>
          </thead>
          <tbody className="divide-y">
            {filteredData.map(c => (
              <tr key={c.id} className="hover:bg-slate-50">
                <td className="px-6 py-4 font-bold text-sky-700 cursor-pointer" onClick={() => setSelectedCustomer(c)}>{c.name}</td>
                <td className="px-6 py-4">{c.phone}</td>
                <td className="px-6 py-4"><span className="px-3 py-1 rounded-full text-xs border">{c.status}</span></td>
                <td className="px-6 py-4">
                  <button onClick={() => setSelectedCustomer(c)} className="text-sky-600 flex items-center"><Info className="h-4 w-4 mr-1"/> รายละเอียด</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      {selectedCustomer && <CustomerDetailModal customer={selectedCustomer} onClose={() => setSelectedCustomer(null)} userPath={userPath} />}
    </div>
  );
}

function PipelineView({ customers, userPath }) {
  const updateStatus = async (id, newStatus) => {
    try { await updateDoc(doc(db, `${userPath}/customers/${id}`), { status: newStatus, updatedAt: serverTimestamp() }); } catch (err) { console.error(err); }
  };
  const stages = ['Lead New', 'Contacted', 'Proposal Sent', 'Closed Sale'];

  return (
    <div className="flex space-x-4 overflow-x-auto pb-4">
      {stages.map(stage => (
        <div key={stage} className="w-80 bg-slate-100 rounded-xl border p-3 shrink-0 min-h-[500px]">
          <h3 className="font-semibold text-slate-700 mb-4 flex justify-between">{stage} <span className="text-xs bg-white px-2 py-1 rounded">{customers.filter(c => c.status === stage).length}</span></h3>
          <div className="space-y-3">
            {customers.filter(c => c.status === stage).map(c => (
              <div key={c.id} className="bg-white p-4 rounded-lg shadow-sm border-l-4 border-sky-500">
                <h4 className="font-bold text-sm mb-1">{c.name}</h4>
                <p className="text-[10px] text-slate-500 mb-2">{c.phone}</p>
                <select value={c.status} onChange={(e) => updateStatus(c.id, e.target.value)} className="text-[10px] border rounded w-full">
                  {STATUS_OPTIONS.map(s => <option key={s} value={s}>{s}</option>)}
                </select>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

function RenewalsView({ customers }) {
  const renewals = customers.filter(c => c.renewalDate).sort((a,b) => new Date(a.renewalDate) - new Date(b.renewalDate));
  return (
    <div className="max-w-5xl mx-auto space-y-6">
      <div className="bg-sky-600 p-6 rounded-xl text-white flex items-center space-x-4 shadow-md">
        <RefreshCw className="h-10 w-10 text-sky-200" />
        <div><h2 className="text-2xl font-bold">Renewal Center</h2><p className="text-sky-100 text-sm">ติดตามกรมธรรม์หมดอายุล่วงหน้า</p></div>
      </div>
      <div className="bg-white rounded-xl border overflow-hidden shadow-sm">
        <table className="w-full text-left text-sm"><thead className="bg-slate-50"><tr><th className="px-6 py-4">ลูกค้า</th><th className="px-6 py-4">งวดการชำระ</th><th className="px-6 py-4">วันหมดอายุ</th><th className="px-6 py-4">สถานะ</th></tr></thead>
        <tbody>{renewals.map(c => <tr key={c.id} className="border-t">
          <td className="px-6 py-4 font-bold">{c.name}</td>
          <td className="px-6 py-4">{PAYMENT_CYCLES.find(cy => cy.value === c.paymentCycle)?.label || c.paymentCycle || 'ไม่ระบุ'}</td>
          <td className="px-6 py-4">{new Date(c.renewalDate).toLocaleDateString('th-TH')}</td>
          <td className="px-6 py-4"><span className="px-2 py-1 bg-amber-50 text-amber-700 rounded text-xs border border-amber-200">รอติดตาม</span></td>
        </tr>)}</tbody></table>
      </div>
    </div>
  );
}

function AiAssistantView() {
  const [prompt, setPrompt] = useState('');
  const [response, setResponse] = useState('');
  const [loading, setLoading] = useState(false);
  const askAI = async (text) => {
    setLoading(true); setResponse('');
    const apiKey = ""; const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
    try {
      const res = await fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ contents: [{ parts: [{ text }] }] }) });
      const data = await res.json(); setResponse(data.candidates?.[0]?.content?.parts?.[0]?.text || "No Response");
    } catch (e) { setResponse("Error"); } finally { setLoading(false); }
  };
  return (
    <div className="max-w-4xl mx-auto flex flex-col h-[600px] bg-white rounded-xl border shadow-sm">
      <div className="p-6 flex-1 overflow-y-auto bg-slate-50 prose prose-sm max-w-none">{response || "พิมพ์คำถามที่ต้องการให้ AI แนะนำ..."}</div>
      <div className="p-4 border-t flex gap-2"><input type="text" value={prompt} onChange={e => setPrompt(e.target.value)} className="flex-1 p-2 border rounded-lg outline-none" placeholder="เช่น ขอสคริปต์การโทรนัดลูกค้าครั้งแรก..." /><button onClick={() => askAI(prompt)} className="bg-sky-600 text-white p-2 px-4 rounded-lg"><Sparkles className="h-5 w-5"/></button></div>
    </div>
  );
}

function SettingsView({ goals, userPath, user }) {
  return (
    <div className="max-w-xl mx-auto bg-white p-8 rounded-xl border shadow-sm space-y-6">
      <h2 className="text-xl font-bold flex items-center"><Settings className="mr-2"/> การตั้งค่า</h2>
      <div className="p-4 bg-slate-50 rounded-lg">
        <p className="text-sm font-bold text-slate-500 mb-1 uppercase">User ID</p>
        <p className="font-mono text-sky-700 text-sm break-all">{user.uid}</p>
      </div>
      <p className="text-slate-400 text-xs text-center italic">Insurance CRM Professional Edition</p>
    </div>
  );
}

```
