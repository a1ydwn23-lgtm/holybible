# holybible
import React, { useEffect, useMemo, useRef, useState } from "react";
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from "recharts";
import {
  format,
  startOfWeek,
  addWeeks,
  subWeeks,
  eachDayOfInterval,
  startOfMonth,
  endOfMonth,
  addMonths,
  subMonths,
  getDate,
  getDay,
  isSameDay,
  parseISO,
} from "date-fns";

// =========================
// Cloud (Firebase) — Optional
// =========================
// 이 앱은 로컬 저장(localStorage)로도 완전 동작합니다.
// 어디서나 같은 데이터를 쓰려면 아래 FIREBASE_CONFIG 를 실제 값으로 채우세요.
// Firebase 콘솔에서 Web App 추가 → 구성 복사.
// 익명 로그인 + Cloud Firestore 사용.
// (설정이 비어 있으면 자동으로 로컬 저장으로 동작합니다.)

const FIREBASE_CONFIG = {
  apiKey: "",
  authDomain: "",
  projectId: "",
  appId: "",
};

let firebaseApp = null;
let firebaseAuth = null;
let firebaseDb = null;

async function ensureFirebase() {
  if (!FIREBASE_CONFIG.projectId) return false; // not configured
  if (firebaseApp) return true;
  const { initializeApp } = await import("firebase/app");
  const { getAuth, signInAnonymously } = await import("firebase/auth");
  const { getFirestore } = await import("firebase/firestore");
  firebaseApp = initializeApp(FIREBASE_CONFIG);
  firebaseAuth = getAuth(firebaseApp);
  await signInAnonymously(firebaseAuth);
  firebaseDb = getFirestore(firebaseApp);
  return true;
}

// Firestore helpers
async function loadFromCloud(groupId, userId) {
  const ok = await ensureFirebase();
  if (!ok) return null;
  const { doc, getDoc } = await import("firebase/firestore");
  const ref = doc(firebaseDb, "groups", groupId, "users", userId);
  const snap = await getDoc(ref);
  if (!snap.exists()) return null;
  return snap.data();
}

async function saveToCloud(groupId, userId, data) {
  const ok = await ensureFirebase();
  if (!ok) return false;
  const { doc, setDoc, serverTimestamp } = await import("firebase/firestore");
  const ref = doc(firebaseDb, "groups", groupId, "users", userId);
  await setDoc(ref, { ...data, updatedAt: serverTimestamp() }, { merge: true });
  return true;
}

// =========================
// Local constants
// =========================
const STORAGE_KEY = "bibleReadingDataV2"; // { users: { [groupId:userId]: {entries, baseline, name} }, current: {groupId,userId} }
const TARGET_TOTAL = 1189;

// =========================
// Helpers
// =========================
function toISODate(d) {
  return format(d, "yyyy-MM-dd");
}

function safeParseDate(value) {
  try {
    if (typeof value === "string") return parseISO(value);
  } catch {}
  return new Date();
}

function classNames(...arr) {
  return arr.filter(Boolean).join(" ");
}

function levelColor(count) {
  if (!count) return "bg-gray-100";
  if (count < 2) return "bg-green-100";
  if (count < 4) return "bg-green-200";
  if (count < 6) return "bg-green-300";
  if (count < 10) return "bg-green-400";
  return "bg-green-500";
}

// =========================
// Main Component
// =========================
export default function BibleReadingTracker() {
  // Multi-user state
  const [groupId, setGroupId] = useState(localStorage.getItem("brt_groupId") || "my-family");
  const [userId, setUserId] = useState(localStorage.getItem("brt_userId") || "me");
  const storageKeyForUser = `${groupId}:${userId}`;

  // Per-user data
  const [entries, setEntries] = useState({}); // { 'YYYY-MM-DD': number }
  const [baseline, setBaseline] = useState(0); // 기록 시작 전 이미 읽은 장수
  const [displayName, setDisplayName] = useState("");

  // UI state
  const [selectedDate, setSelectedDate] = useState(new Date());
  const [countInput, setCountInput] = useState(1);
  const [view, setView] = useState("WEEK"); // WEEK | MONTH
  const [weekStart, setWeekStart] = useState(startOfWeek(new Date(), { weekStartsOn: 1 }));
  const [monthCursor, setMonthCursor] = useState(startOfMonth(new Date()));
  const [cloudEnabled, setCloudEnabled] = useState(!!FIREBASE_CONFIG.projectId);
  const [statusMsg, setStatusMsg] = useState("");

  // ---------- Load persisted (local) ----------
  useEffect(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        const parsed = JSON.parse(raw);
        const u = parsed?.users?.[storageKeyForUser];
        if (u) {
          setEntries(u.entries || {});
          setBaseline(Number(u.baseline || 0));
          setDisplayName(u.name || "");
        }
      }
    } catch (e) {
      console.error("Failed to load storage", e);
    }
  }, [storageKeyForUser]);

  // ---------- Save to local ----------
  useEffect(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      const all = raw ? JSON.parse(raw) : { users: {} };
      all.users[storageKeyForUser] = {
        entries,
        baseline,
        name: displayName,
      };
      localStorage.setItem(STORAGE_KEY, JSON.stringify(all));
    } catch (e) {
      console.error("Failed to save storage", e);
    }
  }, [entries, baseline, displayName, storageKeyForUser]);

  // ---------- Cloud sync (optional) ----------
  useEffect(() => {
    localStorage.setItem("brt_groupId", groupId);
    localStorage.setItem("brt_userId", userId);
  }, [groupId, userId]);

  // Load from cloud when switching user/group
  useEffect(() => {
    (async () => {
      if (!cloudEnabled) return;
      setStatusMsg("클라우드에서 불러오는 중...");
      const cloud = await loadFromCloud(groupId, userId);
      if (cloud) {
        setEntries(cloud.entries || {});
        setBaseline(Number(cloud.baseline || 0));
        setDisplayName(cloud.name || "");
        setStatusMsg("클라우드에서 불러왔어요 ✅");
      } else {
        setStatusMsg("클라우드에 데이터가 없어 새로 시작합니다.");
      }
      setTimeout(() => setStatusMsg(""), 1500);
    })();
  }, [groupId, userId, cloudEnabled]);

  // Save to cloud on change (debounced)
  const saveTimer = useRef(null);
  useEffect(() => {
    if (!cloudEnabled) return;
    if (saveTimer.current) clearTimeout(saveTimer.current);
    saveTimer.current = setTimeout(async () => {
      setStatusMsg("클라우드 저장 중...");
      await saveToCloud(groupId, userId, {
        entries,
        baseline,
        name: displayName,
      });
      setStatusMsg("클라우드 저장 완료 ✅");
      setTimeout(() => setStatusMsg(""), 1200);
    }, 600);
    return () => saveTimer.current && clearTimeout(saveTimer.current);
  }, [entries, baseline, displayName, groupId, userId, cloudEnabled]);

  // ---------- Derived values ----------
  const sumEntries = useMemo(
    () => Object.values(entries).reduce((a, b) => a + (Number(b) || 0), 0),
    [entries]
  );
  const totalRead = baseline + sumEntries;
  const progressPct = Math.min(100, Math.round((totalRead / TARGET_TOTAL) * 100));

  // ---------- Actions ----------
  function addOrSetForDate(dateISO, delta, mode = "add") {
    setEntries((prev) => {
      const current = Number(prev[dateISO] || 0);
      const next = mode === "add" ? current + delta : Math.max(0, delta);
      return { ...prev, [dateISO]: next };
    });
  }
  function handleQuickAdd(n) {
    addOrSetForDate(toISODate(selectedDate), n, "add");
  }
  function handleSetExact() {
    const n = Number(countInput) || 0;
    addOrSetForDate(toISODate(selectedDate), n, "set");
  }
  function handleImport(jsonText) {
    try {
      const parsed = JSON.parse(jsonText);
      if (parsed.entries && typeof parsed.entries === "object") {
        setEntries(parsed.entries);
        if (typeof parsed.baseline === "number") setBaseline(parsed.baseline);
        alert("데이터를 불러왔어요.");
      } else {
        alert("형식이 올바르지 않아요. entries 객체가 필요해요.");
      }
    } catch (e) {
      alert("JSON 파싱 오류: " + e.message);
    }
  }
  function handleExport() {
    const blob = new Blob(
      [
        JSON.stringify(
          { entries, baseline, name: displayName, userId, groupId },
          null,
          2
        ),
      ],
      { type: "application/json" }
    );
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `bible-reading-backup-${groupId}-${userId}-${toISODate(
      new Date()
    )}.json`;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  }

  // ---------- Week data ----------
  const weekDays = useMemo(() => {
    const start = weekStart;
    // 7일 생성
    const end = addWeeks(start, 1);
    const days = eachDayOfInterval({ start, end });
    return days.slice(0, 7);
  }, [weekStart]);

  const weekChartData = weekDays.map((d) => ({
    name: format(d, "EEE"),
    value: Number(entries[toISODate(d)] || 0),
    date: toISODate(d),
  }));
  const weekTotal = weekChartData.reduce((a, b) => a + b.value, 0);

  // ---------- Month / Calendar data ----------
  const monthStart = monthCursor;
  const monthEnd = endOfMonth(monthStart);
  const monthDays = eachDayOfInterval({ start: monthStart, end: monthEnd });

  const monthChartData = monthDays.map((d) => ({
    name: String(getDate(d)),
    value: Number(entries[toISODate(d)] || 0),
    date: toISODate(d),
  }));
  const monthTotal = monthChartData.reduce((a, b) => a + b.value, 0);

  const leadingBlanks = (getDay(startOfMonth(monthStart)) + 6) % 7; // Mon=0

  // ---------- UI ----------
  return (
    <div className="min-h-screen w-full bg-gray-50 text-gray-900">
      {/* Top bar */}
      <header className="sticky top-0 z-10 bg-white border-b border-gray-200">
        <div className="max-w-6xl mx-auto px-4 py-3 flex items-center gap-3">
          <div className="text-xl font-bold tracking-tight">📊 성경 읽기 트래커</div>

          {/* User / Group selector */}
          <div className="ml-auto flex items-center gap-2">
            <input
              className="w-36 border rounded-full px-3 py-1 text-sm"
              placeholder="그룹ID(가족/셀)"
              value={groupId}
              onChange={(e) => setGroupId(e.target.value.trim())}
              title="같이 쓰는 사람들과 동일한 그룹ID를 쓰면 같은 공간을 공유합니다."
            />
            <input
              className="w-28 border rounded-full px-3 py-1 text-sm"
              placeholder="사용자ID"
              value={userId}
              onChange={(e) => setUserId(e.target.value.trim())}
              title="각자의 고유 아이디 (예: mj, hyun)"
            />
            <input
              className="w-32 border rounded-full px-3 py-1 text-sm"
              placeholder="이름(표시용)"
              value={displayName}
              onChange={(e) => setDisplayName(e.target.value)}
              title="카드에 표시되는 이름"
            />
            <button
              onClick={handleExport}
              className="px-3 py-1 rounded-full text-sm bg-gray-100"
            >
              내보내기
            </button>
            <ImportButton onImport={handleImport} />
          </div>
        </div>
        <div className="max-w-6xl mx-auto px-4 pb-3 flex items-center gap-3">
          <button
            onClick={() => setView("WEEK")}
            className={classNames(
              "px-3 py-1 rounded-full text-sm",
              view === "WEEK" ? "bg-gray-900 text-white" : "bg-gray-100"
            )}
          >
            주간
          </button>
          <button
            onClick={() => setView("MONTH")}
            className={classNames(
              "px-3 py-1 rounded-full text-sm",
              view === "MONTH" ? "bg-gray-900 text-white" : "bg-gray-100"
            )}
          >
            월간/캘린더
          </button>

          {/* Cloud toggle */}
          <div className="ml-auto flex items-center gap-2 text-sm">
            <label className="flex items-center gap-2">
              <input
                type="checkbox"
                checked={cloudEnabled}
                onChange={(e) => setCloudEnabled(e.target.checked)}
              />
              클라우드 동기화 (Firebase)
            </label>
            {statusMsg && <span className="text-gray-500">{statusMsg}</span>}
          </div>
        </div>
      </header>

      {/* Content */}
      <main className="max-w-6xl mx-auto px-4 py-6 space-y-8">
        {/* Overview cards */}
        <section className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
          <StatCard title={`총 읽은 장수 (${displayName || userId})`} value={totalRead} subtitle={`목표 ${TARGET_TOTAL}장 | 베이스라인 ${baseline}장`} />
          <StatCard title="진도율" value={`${progressPct}%`} subtitle={<ProgressBar pct={progressPct} />} />
          <StatCard title="이번 주 합계" value={weekTotal} subtitle={format(weekStart, "MM/dd")} />
          <StatCard title="이번 달 합계" value={monthTotal} subtitle={format(monthStart, "MM")} />
        </section>

        {/* Input panel */}
        <section className="bg-white rounded-2xl shadow p-4 md:p-6">
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div>
              <label className="block text-sm font-medium mb-1">날짜 선택</label>
              <input
                type="date"
                className="w-full border rounded-xl px-3 py-2"
                value={toISODate(selectedDate)}
                onChange={(e) => setSelectedDate(safeParseDate(e.target.value))}
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">읽은 장 수</label>
              <div className="flex gap-2">
                <input
                  type="number"
                  min={0}
                  className="w-36 border rounded-xl px-3 py-2"
                  value={countInput}
                  onChange={(e) => setCountInput(e.target.value)}
                />
                <button onClick={handleSetExact} className="px-4 py-2 rounded-xl bg-gray-900 text-white">정확히 설정</button>
                <button onClick={() => handleQuickAdd(Number(countInput) || 0)} className="px-4 py-2 rounded-xl bg-gray-100">+ 더하기</button>
              </div>
              <div className="flex flex-wrap gap-2 mt-2">
                {[1, 2, 3, 5, 10].map((n) => (
                  <button
                    key={n}
                    onClick={() => handleQuickAdd(n)}
                    className="px-3 py-2 rounded-xl bg-gray-100"
                  >
                    +{n}
                  </button>
                ))}
                <button
                  onClick={() => addOrSetForDate(toISODate(selectedDate), 0, "set")}
                  className="px-3 py-2 rounded-xl bg-red-50 text-red-600 border border-red-200"
                >
                  오늘 기록 리셋
                </button>
              </div>
            </div>
          </div>

          {/* Baseline */}
          <div className="mt-6">
            <label className="block text-sm font-medium mb-1">기록 시작 전 이미 읽은 장수(베이스라인)</label>
            <div className="flex items-center gap-2">
              <input
                type="number"
                min={0}
                className="w-48 border rounded-xl px-3 py-2"
                value={baseline}
                onChange={(e) => setBaseline(Math.max(0, Number(e.target.value) || 0))}
              />
              <span className="text-sm text-gray-500">* 진도율과 총합에 포함됩니다.</span>
            </div>
          </div>
          <p className="mt-3 text-sm text-gray-500">
            * 데이터는 브라우저에 자동 저장됩니다. Firebase 설정을 넣고 "클라우드 동기화"를 켜면 어디서든 같은 그룹ID/사용자ID로 접속해 데이터를 공유할 수 있습니다.
          </p>
        </section>

        {/* Views */}
        {view === "WEEK" ? (
          <section className="space-y-4">
            <div className="flex items-center justify-between">
              <div className="font-semibold text-lg">주간 차트</div>
              <div className="flex gap-2">
                <button onClick={() => setWeekStart(subWeeks(weekStart, 1))} className="px-3 py-1 rounded-lg bg-white border">
                  이전 주
                </button>
                <button onClick={() => setWeekStart(startOfWeek(new Date(), { weekStartsOn: 1 }))} className="px-3 py-1 rounded-lg bg-white border">
                  이번 주
                </button>
                <button onClick={() => setWeekStart(addWeeks(weekStart, 1))} className="px-3 py-1 rounded-lg bg-white border">
                  다음 주
                </button>
              </div>
            </div>
            <div className="bg-white rounded-2xl shadow p-4">
              <div className="h-72">
                <ResponsiveContainer width="100%" height="100%">
                  <BarChart data={weekChartData}>
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="name" />
                    <YAxis allowDecimals={false} />
                    <Tooltip formatter={(v) => [v + " 장", "읽음"]} labelFormatter={(l, p) => p?.[0]?.payload?.date || l} />
                    <Bar dataKey="value" radius={[6, 6, 0, 0]} />
                  </BarChart>
                </ResponsiveContainer>
              </div>
            </div>
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              {weekChartData.map((d) => (
                <div key={d.date} className="bg-white rounded-2xl shadow p-4 flex items-center justify-between">
                  <div>
                    <div className="text-sm text-gray-500">{d.date}</div>
                    <div className="text-lg font-semibold">{d.value} 장</div>
                  </div>
                  <div className="flex gap-2">
                    {[1, 2, 5].map((n) => (
                      <button
                        key={n}
                        onClick={() => addOrSetForDate(d.date, n, "add")}
                        className="px-3 py-2 rounded-xl bg-gray-100"
                      >
                        +{n}
                      </button>
                    ))}
                    <button
                      onClick={() => addOrSetForDate(d.date, 0, "set")}
                      className="px-3 py-2 rounded-xl bg-red-50 text-red-600 border border-red-200"
                    >
                      리셋
                    </button>
                  </div>
                </div>
              ))}
            </div>
          </section>
        ) : (
          <section className="space-y-4">
            <div className="flex items-center justify-between">
              <div className="font-semibold text-lg">월간 & 캘린더</div>
              <div className="flex gap-2">
                <button onClick={() => setMonthCursor(subMonths(monthStart, 1))} className="px-3 py-1 rounded-lg bg-white border">
                  이전 달
                </button>
                <button onClick={() => setMonthCursor(startOfMonth(new Date()))} className="px-3 py-1 rounded-lg bg-white border">
                  이번 달
                </button>
                <button onClick={() => setMonthCursor(addMonths(monthStart, 1))} className="px-3 py-1 rounded-lg bg-white border">
                  다음 달
                </button>
              </div>
            </div>

            {/* Calendar grid */}
            <div className="bg-white rounded-2xl shadow p-4">
              <div className="flex items-center justify-between mb-2">
                <div className="text-sm text-gray-600">{format(monthStart, "yyyy.MM")}</div>
                <div className="text-sm text-gray-600">합계: {monthTotal} 장</div>
              </div>

              <div className="grid grid-cols-7 gap-2 text-xs text-gray-500 mb-2">
                {["월", "화", "수", "목", "금", "토", "일"].map((d) => (
                  <div key={d} className="text-center">
                    {d}
                  </div>
                ))}
              </div>

              <div className="grid grid-cols-7 gap-2">
                {Array.from({ length: leadingBlanks }).map((_, i) => (
                  <div key={"b" + i} className="h-16" />
                ))}
                {monthDays.map((d) => {
                  const iso = toISODate(d);
                  const val = Number(entries[iso] || 0);
                  const isSel = isSameDay(d, selectedDate);
                  return (
                    <button
                      key={iso}
                      onClick={() => setSelectedDate(d)}
                      className={classNames(
                        "h-16 rounded-xl border p-1 text-left flex flex-col",
                        isSel ? "border-gray-900" : "border-gray-200",
                        "hover:shadow-sm"
                      )}
                    >
                      <div className="flex items-center justify-between">
                        <span className="text-xs text-gray-500">{getDate(d)}</span>
                        <span className={classNames("w-3 h-3 rounded", levelColor(val))} />
                      </div>
                      <div className="mt-auto text-sm font-semibold">{val} 장</div>
                    </button>
                  );
                })}
              </div>
            </div>

            {/* Monthly chart */}
            <div className="bg-white rounded-2xl shadow p-4">
              <div className="h-72">
                <ResponsiveContainer width="100%" height="100%">
                  <BarChart data={monthChartData}>
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="name" />
                    <YAxis allowDecimals={false} />
                    <Tooltip formatter={(v) => [v + " 장", "읽음"]} labelFormatter={(l, p) => p?.[0]?.payload?.date || l} />
                    <Bar dataKey="value" radius={[6, 6, 0, 0]} />
                  </BarChart>
                </ResponsiveContainer>
              </div>
            </div>
          </section>
        )}

        {/* Help */}
        <section className="bg-white rounded-2xl shadow p-4">
          <div className="font-semibold mb-2">도움말</div>
          <ul className="list-disc pl-5 text-sm text-gray-600 space-y-1">
            <li>여러 사람이 함께 쓰려면 동일한 <b>그룹ID</b>를 맞추고 각자 <b>사용자ID</b>를 다르게 입력하세요.</li>
            <li>Firebase 설정을 코드 상단에 넣고 "클라우드 동기화"를 켜면 인터넷만 있으면 어디서든 접속 가능합니다(익명 로그인).</li>
            <li>베이스라인은 기록 시작 전에 이미 읽은 장수를 의미하며, 총합 및 진도율 계산에 포함됩니다.</li>
            <li>JSON <b>내보내기/가져오기</b>로 백업/이사 가능합니다.</li>
          </ul>
        </section>
      </main>
    </div>
  );
}

function StatCard({ title, value, subtitle }) {
  return (
    <div className="bg-white rounded-2xl shadow p-4">
      <div className="text-sm text-gray-500">{title}</div>
      <div className="text-2xl font-bold mt-1">{value}</div>
      {subtitle && <div className="mt-2 text-sm text-gray-600">{subtitle}</div>}
    </div>
  );
}

function ProgressBar({ pct }) {
  return (
    <div className="mt-1 w-full bg-gray-100 rounded-full h-2">
      <div className="h-2 rounded-full bg-gray-900" style={{ width: `${pct}%` }} />
    </div>
  );
}

function ImportButton({ onImport }) {
  const fileRef = useRef(null);
  return (
    <>
      <input
        ref={fileRef}
        type="file"
        accept="application/json"
        className="hidden"
        onChange={async (e) => {
          const file = e.target.files?.[0];
          if (!file) return;
          const text = await file.text();
          onImport(text);
          e.target.value = "";
        }}
      />
      <button
        onClick={() => fileRef.current?.click()}
        className="px-3 py-1 rounded-full text-sm bg-gray-100"
      >
        가져오기
      </button>
    </>
  );
}
