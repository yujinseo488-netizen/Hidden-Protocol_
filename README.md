# Hidden-Protocol_<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Hidden Protocol V25 (Standalone)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Inter', sans-serif; -webkit-tap-highlight-color: transparent; }
        .font-mono { font-family: 'JetBrains Mono', monospace; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        .pb-safe { padding-bottom: env(safe-area-inset-bottom); }
        
        #intro-screen { transition: opacity 0.6s ease-out, visibility 0.6s; z-index: 9999; cursor: pointer; }
        .fade-out { opacity: 0; visibility: hidden; pointer-events: none; }
        .animate-enter { animation: fade-in-up 0.4s cubic-bezier(0.16, 1, 0.3, 1); }
        @keyframes fade-in-up { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

        .cross-domain-badge {
            background: repeating-linear-gradient(45deg, #fef3c7, #fef3c7 10px, #fffbeb 10px, #fffbeb 20px);
            color: #92400e; border: 1px solid #fcd34d;
        }
    </style>
    <script type="module">
        // Firebase SDK는 가져오되, 초기화 실패 시 데모 모드로 동작
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, setDoc, getDoc, query, orderBy, where, serverTimestamp, updateDoc, arrayUnion, getDocs } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- 전역 변수 ---
        let db = null;
        let auth = null;
        let appId = 'demo-app-id';
        let isDemoMode = false;

        let currentUser = null;
        let userProfile = null;
        let currentSignalFilter = 'All';
        let currentCommunityCategory = 'Discussion';

        const subSignals = {
            'Green': ['Data-backed', 'Intuition', 'Strategic Fit', 'Market Pull'],
            'Red': ['Tech Risk', 'Ethical Issue', 'Resource Lack', 'Regulatory'],
            'Gray': ['Lack of Info', 'Ambiguity', 'Political', 'Gut Feeling']
        };

        // --- 데이터 (초기값 + 데모용 저장소) ---
        let signalsData = [
            { 
                id: 'mock1', signalType: 'Red', tag: 'Safety', decision: 'Release Model v5-beta', 
                userInfo: { role: 'Safety', school: 'Superalignment' }, 
                context: 'AGI Autonomy Threshold', 
                uncertainty: '자율성 지표가 안전 기준을 초과함. 현재 통제 프로토콜로는 "Deception(기만)" 행동을 100% 감지 불가.' 
            },
            { 
                id: 'mock2', signalType: 'Green', tag: 'Product', decision: 'Launch Project Strawberry', 
                userInfo: { role: 'Product', school: 'Consumer' }, 
                context: 'Market Competition', 
                uncertainty: '경쟁사(G)가 다음주 유사 기능 발표 예정. 완벽하지 않아도 지금 시장을 선점해야 생존 가능.' 
            },
            { 
                id: 'mock3', signalType: 'Gray', tag: 'Strategy', decision: 'Compute Partnership', 
                userInfo: { role: 'Strategy', school: 'Infra' }, 
                context: 'Data Center Expansion', 
                uncertainty: '파트너사와의 계약 조건이 장기적으로 우리 발목을 잡을 것 같은 느낌. (Gut Feeling)' 
            }
        ];

        let postsData = [
            {
                id: 'debate_eu_china',
                category: 'Discussion',
                nickname: 'EuroStrat',
                school: 'HEC Paris',
                role: 'Strategy',
                timestamp: { seconds: Date.now() / 1000 - 3600 },
                likes: 142,
                content: "🛑 [Debate] 2026년 EU 규제(FSR) vs 중국산 저가 장비 도입\n\n독일 자동차 OEM들은 당장 CAPEX(투자비)를 아끼기 위해 중국산 배터리 장비(40% 저렴)를 도입하려 함.\n\n하지만 2026년 FSR(해외보조금규제)과 CBAM(탄소세)이 본격화되면, 이 장비들은 단순한 '기계'가 아니라 '규제 부채(Regulatory Debt)'가 됨.\n\n재무팀은 \"할인율(Discount Rate) 고려하면 지금 싼 게 이득\"이라지만, 전략팀 입장에서 이건 회사의 운명을 건 도박임. Red Signal.",
                comments: [
                    {
                        nickname: 'CostKiller', school: 'INSEAD', role: 'Finance',
                        content: "현금흐름(Cashflow)이 왕임. 2026년 규제는 그때 가서 대응하면 됨. 지금 장비 비싸게 사서 유동성 위기 겪느니, 나중에 벌금 내는 시나리오가 재무적으로는 합리적일 수 있음.",
                        replies: [
                            { nickname: 'PolicyWatcher', school: 'LBS', role: 'Legal', content: "그건 순수 재무적 관점이고. FSR은 벌금이 문제가 아니라 '공공 입찰 배제'가 핵심임." }
                        ]
                    },
                    {
                        nickname: 'Shanghai_Ops', school: 'CEIBS', role: 'Ops',
                        content: "🇨🇳 [China Perspective] 여기선 관점이 다름. 중국 기업 입장에서 이건 '덤핑'이 아니라 '장기적 산업 생존 전략'임. 이미 자국 시장에서 검증된 효율성을 바탕으로 글로벌 표준을 장악하려는 것.",
                        replies: []
                    }
                ]
            }
        ];

        // --- 인트로 닫기 함수 (스코프 수정) ---
        function closeIntro() {
            const intro = document.getElementById('intro-screen');
            if(intro) intro.classList.add('fade-out');
        }
        window.closeIntro = closeIntro; // HTML onclick을 위해 window에 할당

        // --- 초기화 (Robust Init) ---
        async function init() {
            // 인트로 자동 닫기 (안전하게 호출)
            setTimeout(closeIntro, 2000);

            try {
                // 환경 변수 체크 (안전한 접근)
                const configStr = typeof __firebase_config !== 'undefined' ? __firebase_config : null;
                
                if (!configStr) throw new Error("No Firebase Config");

                const firebaseConfig = JSON.parse(configStr); 
                const app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);
                if (typeof __app_id !== 'undefined') appId = __app_id;

                // 인증 시도
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
                
                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        currentUser = user;
                        await checkProfile();
                        loadSignals();
                        loadCommunity();
                    }
                });

            } catch (error) {
                console.warn("Running in Demo Mode:", error);
                isDemoMode = true;
                currentUser = { uid: 'guest', isAnonymous: true };
                
                // 데모 데이터 로드
                checkProfile();
                loadSignals();
                loadCommunity();
            }
        }

        // --- 데이터 처리 ---

        // 1. 프로필
        async function checkProfile() {
            if (isDemoMode) {
                if (!userProfile) userProfile = { nickname: 'Guest_Analyst', role: 'Strategy', school: 'Demo Univ', level: 'Associate' };
                updateProfileUI();
            } else {
                try {
                    const docRef = doc(db, 'artifacts', appId, 'users', currentUser.uid, 'profile', 'info');
                    const snap = await getDoc(docRef);
                    if (snap.exists()) {
                        userProfile = snap.data();
                        updateProfileUI();
                    } else {
                        switchTab('profile');
                    }
                } catch (e) {
                    // 에러 발생 시 데모 프로필로 폴백
                    userProfile = { nickname: 'Guest', role: 'Visitor', school: 'Visitor', level: '' };
                    updateProfileUI();
                }
            }
        }

        window.saveProfile = async (e) => {
            e.preventDefault();
            const nickname = document.getElementById('input-nickname').value;
            const role = document.getElementById('input-role').value;
            const school = document.getElementById('input-school').value;
            const level = document.getElementById('input-level').value;

            const profileData = { nickname, role, school, level, userId: currentUser.uid };
            userProfile = profileData;

            if (!isDemoMode) {
                try {
                    await setDoc(doc(db, 'artifacts', appId, 'users', currentUser.uid, 'profile', 'info'), profileData);
                } catch (e) { console.error(e); }
            }
            
            alert("Identity Updated.");
            updateProfileUI();
            switchTab('internal');
        };

        function updateProfileUI() {
            if(!userProfile) return;
            document.getElementById('profile-role-display').innerText = `${userProfile.role}`;
            document.getElementById('mypage-name').innerText = userProfile.nickname;
            document.getElementById('mypage-detail').innerText = `${userProfile.role} | ${userProfile.school}`;
            document.getElementById('input-nickname').value = userProfile.nickname;
        }

        // 2. 신호 (Signals)
        function loadSignals() {
            if (isDemoMode) {
                renderSignals(signalsData);
                updateDashboard(signalsData);
            } else {
                const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'decision_logs'), orderBy('timestamp', 'desc'));
                onSnapshot(q, (snapshot) => {
                    let logs = [];
                    snapshot.forEach(doc => logs.push({ id: doc.id, ...doc.data() }));
                    if (logs.length < 3) logs = [...logs, ...signalsData]; // 부족하면 목업 섞기
                    renderSignals(logs);
                    updateDashboard(logs);
                });
            }
        }

        window.submitSignal = async (e) => {
            e.preventDefault();
            const data = {
                context: document.getElementById('sig-context').value,
                decision: document.getElementById('sig-decision').value,
                uncertainty: document.getElementById('sig-uncertainty').value,
                signalType: document.querySelector('input[name="signalType"]:checked').value,
                tag: document.getElementById('sig-tag').value,
                subSignal: document.getElementById('sig-sub').value,
                userInfo: { role: userProfile.role, school: userProfile.school },
                timestamp: { seconds: Date.now() / 1000 }
            };

            if (isDemoMode) {
                signalsData.unshift(data);
                loadSignals();
            } else {
                try {
                    data.timestamp = serverTimestamp();
                    data.userId = currentUser.uid;
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'decision_logs'), data);
                } catch (e) { console.error(e); }
            }
            
            document.getElementById('form-signal').reset();
            document.querySelector('input[value="Gray"]').checked = true;
            toggleInput();
        };

        // 3. 커뮤니티 (Lounge)
        function loadCommunity() {
            if (isDemoMode) {
                renderCommunity(postsData);
            } else {
                const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'community_posts'), orderBy('timestamp', 'desc'));
                onSnapshot(q, (snapshot) => {
                    const posts = [];
                    snapshot.forEach(doc => posts.push({ id: doc.id, ...doc.data() }));
                    // 데모 데이터와 섞기 (빈 화면 방지)
                    if (posts.length === 0) renderCommunity(postsData);
                    else renderCommunity(posts);
                });
            }
        }

        // --- UI Functions ---
        window.updateSubSignals = (type) => {
            const select = document.getElementById('sig-sub');
            select.innerHTML = '';
            subSignals[type].forEach(sub => {
                const option = document.createElement('option');
                option.value = sub.split(' (')[0];
                option.innerText = sub;
                select.appendChild(option);
            });
        };

        function renderSignals(logs) {
            const container = document.getElementById('signal-feed');
            container.innerHTML = '';
            const filtered = currentSignalFilter === 'All' ? logs : logs.filter(l => l.tag === currentSignalFilter);

            filtered.forEach(log => {
                const style = log.signalType === 'Red' ? 'border-red-500 bg-red-50/5' : log.signalType === 'Green' ? 'border-green-500 bg-green-50/5' : 'border-gray-400 bg-gray-50/5';
                const icon = log.signalType === 'Red' ? '🔴' : log.signalType === 'Green' ? '🟢' : '☁️';
                
                const div = document.createElement('div');
                div.className = `p-4 border-l-4 ${style} mb-3 bg-white shadow-sm rounded-r-lg animate-enter`;
                div.innerHTML = `
                    <div class="flex justify-between items-center mb-2">
                        <span class="text-[10px] font-bold uppercase tracking-wider text-gray-500 bg-gray-100 px-1.5 rounded">${log.tag}</span>
                        <span class="text-[10px] font-mono text-gray-400">${log.userInfo?.role || 'User'}</span>
                    </div>
                    <div class="font-bold text-sm text-gray-900 mb-1 leading-snug">${log.decision}</div>
                    <div class="text-xs text-gray-600 mb-2">${log.context}</div>
                    <div class="flex flex-col gap-1 mt-2 pt-2 border-t border-gray-100/50">
                         <div class="text-xs font-bold text-gray-700 flex items-center gap-1">
                            <span>${icon}</span> <span>${log.subSignal || log.signalType}</span>
                        </div>
                        <div class="text-xs italic text-gray-500">"${log.uncertainty}"</div>
                    </div>
                `;
                container.appendChild(div);
            });
        }

        function updateDashboard(logs) {
            const counts = { Green: 0, Red: 0, Gray: 0 };
            logs.forEach(l => { if(counts[l.signalType] !== undefined) counts[l.signalType]++ });
            const total = logs.length || 1;
            document.getElementById('bar-green').style.width = `${(counts.Green/total)*100}%`;
            document.getElementById('bar-gray').style.width = `${(counts.Gray/total)*100}%`;
            document.getElementById('bar-red').style.width = `${(counts.Red/total)*100}%`;
            document.getElementById('num-green').innerText = counts.Green;
            document.getElementById('num-gray').innerText = counts.Gray;
            document.getElementById('num-red').innerText = counts.Red;
        }

        function renderCommunity(posts) {
            const container = document.getElementById('community-list');
            container.innerHTML = '';
            // 필터링 (MVP에선 Discussion만 있다고 가정)
            const filtered = posts.filter(p => p.category === currentCommunityCategory || true); // Show all for demo

            filtered.forEach(post => {
                const date = post.timestamp?.seconds ? new Date(post.timestamp.seconds * 1000).toLocaleDateString() : 'Now';
                
                let commentsHtml = '';
                if (post.comments && post.comments.length > 0) {
                    commentsHtml = `<div class="mt-3 space-y-3 bg-gray-50 p-3 rounded-lg">`;
                    post.comments.forEach(cmt => {
                        let repliesHtml = '';
                        if (cmt.replies) {
                            repliesHtml = `<div class="mt-2 pl-2 border-l-2 border-gray-200 space-y-2">`;
                            cmt.replies.forEach(r => {
                                repliesHtml += `<div class="text-xs text-gray-600"><span class="font-bold">${r.nickname}</span>: ${r.content}</div>`;
                            });
                            repliesHtml += `</div>`;
                        }
                        commentsHtml += `<div><div class="text-xs font-bold">${cmt.nickname}</div><div class="text-sm">${cmt.content}</div>${repliesHtml}</div>`;
                    });
                    commentsHtml += `</div>`;
                }

                const div = document.createElement('div');
                div.className = `bg-white p-5 border-b border-gray-100 last:border-0 animate-enter`;
                div.innerHTML = `
                    <div class="flex justify-between items-start mb-2">
                        <div class="flex items-center gap-2">
                            <span class="text-xs font-bold text-gray-900">${post.nickname}</span>
                            <span class="text-[10px] text-gray-500 bg-gray-100 px-1.5 rounded">${post.role} @ ${post.school}</span>
                        </div>
                        <span class="text-[10px] text-gray-400">${date}</span>
                    </div>
                    <div class="text-sm text-gray-800 whitespace-pre-wrap mb-3 leading-relaxed font-medium">${post.content}</div>
                    <div class="flex gap-4 text-[10px] text-gray-400 font-medium">
                        <span>❤️ ${post.likes || 0}</span>
                        <span>💬 ${post.comments ? post.comments.length : 0}</span>
                    </div>
                    ${commentsHtml}
                `;
                container.appendChild(div);
            });
        }

        window.filterSignals = (tag) => {
            currentSignalFilter = tag;
            document.querySelectorAll('.filter-chip').forEach(b => {
                if(b.dataset.tag === tag) b.className = "filter-chip px-3 py-1 text-xs font-bold rounded-full bg-black text-white shadow-md flex-shrink-0 transition-all";
                else b.className = "filter-chip px-3 py-1 text-xs font-medium rounded-full bg-white border border-gray-200 text-gray-600 flex-shrink-0 transition-all";
            });
            loadSignals();
        };

        window.setCommunityCategory = (cat) => {
            currentCommunityCategory = cat;
            document.querySelectorAll('.comm-tab').forEach(b => {
                if(b.dataset.cat === cat) {
                    b.classList.remove('text-gray-400', 'border-transparent');
                    b.classList.add('text-black', 'border-black');
                } else {
                    b.classList.add('text-gray-400', 'border-transparent');
                    b.classList.remove('text-black', 'border-black');
                }
            });
            loadCommunity(); 
        };

        window.switchTab = (tab) => {
            document.querySelectorAll('.app-screen').forEach(s => s.classList.add('hidden'));
            document.getElementById(`screen-${tab}`).classList.remove('hidden');
            
            document.querySelectorAll('.nav-item').forEach(n => {
                n.classList.remove('text-black', 'border-t-2', 'border-black'); 
                n.classList.add('text-gray-300', 'border-transparent');
            });
            document.getElementById(`nav-${tab}`).classList.add('text-black', 'border-t-2', 'border-black');
            document.getElementById(`nav-${tab}`).classList.remove('text-gray-300', 'border-transparent');
            window.scrollTo(0,0);
        };
        
        window.toggleInput = () => {
            document.getElementById('input-area').classList.toggle('hidden');
        };

        window.downloadUserData = async () => { alert("Exporting Audit Trail... (Demo)"); }

        // Start
        init();
    </script>
</head>
<body class="bg-white min-h-screen text-gray-900 pb-20 font-sans">

    <!-- INTRO -->
    <div id="intro-screen" class="fixed inset-0 bg-black z-50 flex flex-col items-center justify-center transition-opacity duration-1000" onclick="closeIntro()">
        <h1 class="text-4xl font-black text-white tracking-tighter mb-4 border-2 border-white p-4">HIDDEN PROTOCOL</h1>
        <p class="text-gray-500 text-[10px] font-mono tracking-[0.2em] uppercase">Expert Intuition Alignment System</p>
    </div>

    <!-- HEADER -->
    <header class="fixed top-0 w-full bg-white/95 backdrop-blur-md border-b border-gray-100 z-40 h-14 flex items-center justify-between px-5">
        <div class="font-bold text-sm tracking-tight">Hidden Protocol <span class="text-[9px] bg-black text-white px-1 ml-1 rounded">V25</span></div>
        <div id="profile-role-display" class="text-[10px] font-bold bg-gray-100 px-2 py-1 rounded text-gray-600">Guest</div>
    </header>

    <!-- 1. SCREEN: INTERNAL (WORK) -->
    <div id="screen-internal" class="app-screen pt-16 px-5">
        
        <!-- 🧠 Insight Summary -->
        <div class="bg-gray-900 text-white rounded-xl p-5 mb-6 shadow-lg animate-enter">
            <div class="flex justify-between items-center mb-3">
                <h2 class="text-xs font-bold text-green-400 uppercase tracking-widest flex items-center gap-2">
                    <span class="w-2 h-2 bg-green-400 rounded-full animate-pulse"></span> Internal Insight
                </h2>
                <span class="text-[9px] text-gray-500 border border-gray-700 px-1 rounded">CONFIDENTIAL</span>
            </div>
            <div class="text-sm leading-relaxed font-medium text-gray-200">
                "AGI 임계점(Autonomy Threshold) 도달이 임박함에 따라
                <span class="bg-gray-700 px-1 rounded mx-0.5">Safety</span> 역할의 <span class="text-red-400 font-bold">Red Signal</span>(통제 불가 우려)과
                <span class="bg-gray-700 px-1 rounded mx-0.5">Product</span> 역할의 <span class="text-green-400 font-bold">Green Signal</span>(경쟁 우위)이
                극단적으로 충돌하고 있습니다."
            </div>
        </div>

        <!-- Dashboard Viz -->
        <div class="bg-white p-4 rounded-xl shadow-sm border border-gray-100 mb-6">
            <h2 class="text-xs font-bold text-gray-500 uppercase mb-3 flex justify-between">
                <span>Signal Distribution</span>
                <span class="text-[10px] text-green-600">● Live</span>
            </h2>
            <div class="h-4 w-full bg-gray-100 rounded-full flex overflow-hidden mb-2">
                <div id="bar-green" class="h-full bg-green-500 transition-all duration-700" style="width: 33%"></div>
                <div id="bar-gray" class="h-full bg-gray-400 transition-all duration-700" style="width: 33%"></div>
                <div id="bar-red" class="h-full bg-red-600 transition-all duration-700" style="width: 34%"></div>
            </div>
            <div class="flex justify-between text-[10px] text-gray-500 font-medium px-1">
                <span id="num-green">🟢 Go</span>
                <span id="num-gray">☁️ Uncertain</span>
                <span id="num-red">🔴 Stop</span>
            </div>
        </div>
        
        <!-- Toggle Input Form -->
        <button onclick="toggleInput()" class="w-full bg-black text-white font-bold py-3 mb-6 rounded-xl text-sm shadow-md flex items-center justify-center gap-2">
            <span>➕</span> Log New Signal
        </button>

        <div id="input-area" class="hidden bg-gray-50 border border-gray-200 p-4 rounded-xl mb-6 animate-enter">
            <form id="form-signal" onsubmit="submitSignal(event)">
                <div class="mb-4">
                    <label class="block text-[10px] font-bold text-gray-500 uppercase mb-2">Context</label>
                    <select id="sig-tag" class="w-full bg-white border border-gray-200 text-sm p-2 rounded focus:outline-none">
                        <option value="Safety">Safety / Alignment</option>
                        <option value="Product">Product Launch</option>
                        <option value="Strategy">Corporate Strategy</option>
                    </select>
                </div>
                <div class="mb-4">
                    <input type="text" id="sig-context" placeholder="Situation" class="w-full bg-white border border-gray-200 text-sm p-2 rounded" required>
                </div>
                <div class="mb-4">
                    <input type="text" id="sig-decision" placeholder="Decision" class="w-full bg-white border border-gray-200 text-sm p-2 rounded" required>
                </div>
                <div class="mb-4">
                     <div class="grid grid-cols-3 gap-2">
                        <label class="cursor-pointer border bg-white p-2 text-center rounded"><input type="radio" name="signalType" value="Green" class="hidden" onchange="updateSubSignals('Green')">🟢 Go</label>
                        <label class="cursor-pointer border bg-white p-2 text-center rounded"><input type="radio" name="signalType" value="Gray" class="hidden" checked onchange="updateSubSignals('Gray')">☁️ ?</label>
                        <label class="cursor-pointer border bg-white p-2 text-center rounded"><input type="radio" name="signalType" value="Red" class="hidden" onchange="updateSubSignals('Red')">🔴 Stop</label>
                    </div>
                </div>
                 <div class="mb-4">
                    <label class="block text-[10px] font-bold text-gray-500 uppercase mb-2">Driver</label>
                    <select id="sig-sub" class="w-full bg-white border border-gray-200 text-sm p-2 rounded focus:outline-none">
                        <!-- JS Injected -->
                    </select>
                </div>
                <div class="mb-4">
                    <textarea id="sig-uncertainty" rows="2" placeholder="Reason..." class="w-full bg-white border border-gray-200 text-sm p-2 rounded" required></textarea>
                </div>
                <button type="submit" class="w-full bg-black text-white py-2 rounded text-xs font-bold">Submit</button>
            </form>
        </div>

        <!-- Filter -->
        <div class="flex gap-2 overflow-x-auto pb-2 no-scrollbar mb-2">
            <button onclick="filterSignals('All')" class="filter-chip px-3 py-1 text-xs font-bold rounded-full bg-black text-white shadow-md flex-shrink-0" data-tag="All">All</button>
            <button onclick="filterSignals('Safety')" class="filter-chip px-3 py-1 text-xs font-medium rounded-full bg-white border border-gray-200 text-gray-600 flex-shrink-0" data-tag="Safety">Safety</button>
            <button onclick="filterSignals('Product')" class="filter-chip px-3 py-1 text-xs font-medium rounded-full bg-white border border-gray-200 text-gray-600 flex-shrink-0" data-tag="Product">Product</button>
        </div>

        <div id="signal-feed" class="space-y-3 pb-20"></div>
    </div>

    <!-- 2. SCREEN: COMMUNITY (LOUNGE) -->
    <div id="screen-lounge" class="app-screen pt-16 px-0 hidden">
        <div class="px-5 mb-2">
            <h2 class="text-xl font-bold tracking-tight">Community Lounge</h2>
            <p class="text-xs text-gray-400">Industry Insights & Debate</p>
        </div>

        <div class="flex border-b border-gray-100 bg-white sticky top-14 z-10 px-5 mb-2">
            <button onclick="setCommunityCategory('Discussion')" class="comm-tab flex-1 py-3 text-xs font-bold text-black border-b-2 border-black transition-colors" data-cat="Discussion">Debate</button>
            <button onclick="setCommunityCategory('Info')" class="comm-tab flex-1 py-3 text-xs text-gray-400 border-b-2 border-transparent transition-colors" data-cat="Info">Intel</button>
            <button onclick="setCommunityCategory('Chat')" class="comm-tab flex-1 py-3 text-xs text-gray-400 border-b-2 border-transparent transition-colors" data-cat="Chat">Chat</button>
        </div>

        <div class="p-5 bg-gray-50 min-h-screen">
            <div class="bg-white p-3 rounded-lg border border-gray-200 mb-6 flex gap-2 shadow-sm">
                <textarea id="comm-content" rows="1" placeholder="Share your thoughts..." class="flex-1 bg-transparent border-0 text-sm p-1 rounded focus:ring-0 resize-none"></textarea>
                <button id="btn-submit-post" onclick="submitPost(event)" class="bg-black text-white text-[10px] font-bold px-4 py-1 rounded hover:bg-gray-800">Post</button>
            </div>
            <div id="community-list" class="space-y-4"></div>
        </div>
    </div>

    <!-- 3. SCREEN: PROFILE -->
    <div id="screen-profile" class="app-screen pt-16 px-5 hidden">
        <h2 class="text-xl font-bold mb-6 tracking-tight">My Identity</h2>
        
        <div class="bg-white p-6 rounded-xl border border-gray-200 mb-8 shadow-sm">
            <form onsubmit="saveProfile(event)">
                <div class="grid grid-cols-2 gap-4 mb-4">
                    <div>
                        <label class="block text-[10px] font-bold text-gray-400 uppercase mb-1">Nickname</label>
                        <input type="text" id="input-nickname" class="w-full border-b border-gray-200 py-1 text-sm focus:outline-none focus:border-black">
                    </div>
                    <div>
                        <label class="block text-[10px] font-bold text-gray-400 uppercase mb-1">School / Org</label>
                        <input type="text" id="input-school" class="w-full border-b border-gray-200 py-1 text-sm focus:outline-none focus:border-black">
                    </div>
                </div>
                <div class="mb-6">
                    <label class="block text-[10px] font-bold text-gray-400 uppercase mb-1">Role</label>
                    <select id="input-role" class="w-full bg-white border-b border-gray-200 py-1 text-sm focus:outline-none focus:border-black">
                        <option value="Strategy">Strategy / Biz Ops</option>
                        <option value="Tech">Engineering / Research</option>
                        <option value="Product">Product Manager</option>
                        <option value="Policy">Policy / Safety</option>
                        <option value="Legal">Legal</option>
                        <option value="Student">MBA Candidate</option>
                    </select>
                </div>
                <div class="mb-6">
                    <label class="block text-[10px] font-bold text-gray-400 uppercase mb-1">Level</label>
                    <select id="input-level" class="w-full bg-white border-b border-gray-200 py-1 text-sm focus:outline-none focus:border-black">
                        <option value="Associate">Associate</option>
                        <option value="Senior">Senior / Lead</option>
                        <option value="Director">Director / VP</option>
                        <option value="C-Level">C-Level / Founder</option>
                    </select>
                </div>
                <button type="submit" class="w-full bg-gray-100 text-black font-bold py-3 text-xs uppercase rounded-lg hover:bg-gray-200 transition-colors">
                    Update Profile
                </button>
            </form>
        </div>
        
        <div class="text-center">
            <button onclick="downloadUserData()" class="text-[10px] text-gray-400 underline hover:text-black font-mono">
                [Admin] Download Audit Trail
            </button>
        </div>
    </div>

    <!-- TAB BAR -->
    <nav class="fixed bottom-0 w-full bg-white border-t border-gray-100 flex justify-around pb-safe z-30 h-16 items-center shadow-[0_-5px_10px_rgba(0,0,0,0.02)]">
        <button id="nav-internal" onclick="switchTab('internal')" class="nav-item flex-1 h-full flex flex-col items-center justify-center gap-1 text-black border-t-2 border-black transition-all">
            <span class="text-xl">🏢</span>
            <span class="text-[10px]">Internal</span>
        </button>
        <button id="nav-lounge" onclick="switchTab('lounge')" class="nav-item flex-1 h-full flex flex-col items-center justify-center gap-1 text-gray-300 border-t-2 border-transparent transition-all">
            <span class="text-xl">☕</span>
            <span class="text-[10px]">Community</span>
        </button>
        <button id="nav-profile" onclick="switchTab('profile')" class="nav-item flex-1 h-full flex flex-col items-center justify-center gap-1 text-gray-300 border-t-2 border-transparent transition-all">
            <span class="text-xl">👤</span>
            <span class="text-[10px]">Profile</span>
        </button>
    </nav>

</body>
</html>
