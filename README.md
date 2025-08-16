naijaspendsimulator.html
# naijaspendsim
For earnings 
import React, { useEffect, useMemo, useRef, useState } from "react";
import { motion } from "framer-motion";
import { CheckCircle2, Clock, Coins, LogOut, Settings2, User, List } from "lucide-react";

// NaijaSpend — Naira Rewards Simulator (offline, for fun only)
// Single-file React component (Tailwind classes used for styling)
// IMPORTANT: This is a simulator. It does not handle real money, does not connect to banks, and cannot send or receive Naira.

const STORAGE_KEY = "naijaspend-sim";

type Store = {
  username: string | null;
  balanceNGN: number; // simulated Naira balance
  lastClaimISO: string | null;
  tasksDone: string[];
  claimTime: string; // HH:mm
  claimWindowMins: number;
  claimAmountNGN: number; // daily claim amount (NGN)
  transactions: { id: string; iso: string; amount: number; type: string; note?: string }[];
};

function loadStore(): Store {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) throw new Error("empty");
    const parsed = JSON.parse(raw) as Store;
    return {
      username: parsed.username ?? null,
      balanceNGN: Number(parsed.balanceNGN ?? 50000),
      lastClaimISO: parsed.lastClaimISO ?? null,
      tasksDone: Array.isArray(parsed.tasksDone) ? parsed.tasksDone : [],
      claimTime: parsed.claimTime || "14:30",
      claimWindowMins: Number(parsed.claimWindowMins ?? 5),
      claimAmountNGN: Number(parsed.claimAmountNGN ?? 5000),
      transactions: Array.isArray(parsed.transactions) ? parsed.transactions : [],
    };
  } catch {
    return {
      username: null,
      balanceNGN: 50000, // starts with ₦50,000 credited for maintenance (simulated)
      lastClaimISO: null,
      tasksDone: [],
      claimTime: "14:30",
      claimWindowMins: 5,
      claimAmountNGN: 5000, // daily claim amount (simulated)
      transactions: [
        { id: "init-credit", iso: new Date().toISOString(), amount: 50000, type: "credit", note: "Maintenance credit (simulated)" },
      ],
    };
  }
}

function saveStore(s: Store) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(s));
}

function todayKey(d = new Date()) {
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${y}-${m}-${day}`;
}

function parseHHMM(hhmm: string) {
  const [h, m] = hhmm.split(":").map((x) => Number(x));
  return { h: isNaN(h) ? 0 : h, m: isNaN(m) ? 0 : m };
}

function getWindowForDate(date: Date, hhmm: string, windowMins: number) {
  const { h, m } = parseHHMM(hhmm);
  const start = new Date(date);
  start.setHours(h, m, 0, 0);
  const end = new Date(start.getTime() + windowMins * 60_000);
  return { start, end };
}

function formatTime(d: Date) {
  const hh = String(d.getHours()).padStart(2, "0");
  const mm = String(d.getMinutes()).padStart(2, "0");
  const ss = String(d.getSeconds()).padStart(2, "0");
  return `${hh}:${mm}:${ss}`;
}

function humanCountdown(ms: number) {
  const s = Math.max(0, Math.floor(ms / 1000));
  const hh = Math.floor(s / 3600);
  const mm = Math.floor((s % 3600) / 60);
  const ss = s % 60;
  const parts = [
    hh > 0 ? `${hh}h` : null,
    mm > 0 ? `${mm}m` : null,
    `${ss}s`,
  ].filter(Boolean);
  return parts.join(" ");
}

export default function NaijaSpendSimulator() {
  const [store, setStore] = useState<Store>(() => loadStore());
  const [now, setNow] = useState<Date>(new Date());
  const [usernameInput, setUsernameInput] = useState("");
  const tickRef = useRef<number | null>(null);

  useEffect(() => {
    tickRef.current = window.setInterval(() => setNow(new Date()), 1000);
    return () => {
      if (tickRef.current) window.clearInterval(tickRef.current);
    };
  }, []);

  useEffect(() => {
    saveStore(store);
  }, [store]);

  const { inWindow, windowStart, windowEnd, millisToOpen, millisToClose, today, lastClaimWasToday } = useMemo(() => {
    const today = todayKey(now);
    const { start, end } = getWindowForDate(now, store.claimTime, store.claimWindowMins);
    const inWindow = now >= start && now <= end;
    const millisToOpen = Math.max(0, start.getTime() - now.getTime());
    const millisToClose = Math.max(0, end.getTime() - now.getTime());
    const lastClaimWasToday = store.lastClaimISO ? todayKey(new Date(store.lastClaimISO)) === today : false;
    return { inWindow, windowStart: start, windowEnd: end, millisToOpen, millisToClose, today, lastClaimWasToday };
  }, [now, store.claimTime, store.claimWindowMins, store.lastClaimISO]);

  // track day rollover and add transaction if initial credit exists already
  const previousDayRef = useRef<string>(todayKey());
  useEffect(() => {
    const cur = todayKey(now);
    if (cur !== previousDayRef.current) {
      previousDayRef.current = cur;
    }
  }, [now]);

  function handleLogin(e: React.FormEvent) {
    e.preventDefault();
    const name = usernameInput.trim();
    if (!name) return;
    setStore((s) => ({ ...s, username: name }));
  }

  function handleLogout() {
    setStore((s) => ({ ...s, username: null }));
  }

  function performClaim() {
    if (!inWindow || store.username === null) return;
    if (lastClaimWasToday) return;
    const amt = store.claimAmountNGN;
    const tx = { id: `claim-${Date.now()}`, iso: new Date().toISOString(), amount: amt, type: "credit", note: "Daily claim (simulated)" };
    setStore((s) => ({ ...s, balanceNGN: s.balanceNGN + amt, lastClaimISO: new Date().toISOString(), transactions: [tx, ...s.transactions] }));
  }

  // Tasks: two surveys + TikTok video + Instagram engagement mock
  const tasks = [
    { id: "survey-a", title: "Survey A (3–4 min)", reward: 100 },
    { id: "survey-b", title: "Survey B (2–3 min)", reward: 100 },
    { id: "tiktok-1", title: "Watch TikTok clip", reward: 100, kind: "video" },
    { id: "insta-1", title: "Visit Instagram profile & like/comment", reward: 100, kind: "social" },
  ];

  function completeTask(id: string, reward: number) {
    if (!store.username) return;
    if (store.tasksDone.includes(id)) return;
    const tx = { id: `task-${Date.now()}`, iso: new Date().toISOString(), amount: reward, type: "credit", note: `Task: ${id}` };
    setStore((s) => ({ ...s, tasksDone: [...s.tasksDone, id], balanceNGN: s.balanceNGN + reward, transactions: [tx, ...s.transactions] }));
  }

  // Withdraw simulation: enabled when balance >= 15000, withdraws 5000 when clicked
  function attemptWithdraw() {
    if (!store.username) return;
    if (store.balanceNGN < 15000) return;
    const withdrawAmount = 5000; // fixed withdrawal amount
    const tx = { id: `withdraw-${Date.now()}`, iso: new Date().toISOString(), amount: -withdrawAmount, type: "debit", note: "Withdrawal (simulated)" };
    setStore((s) => ({ ...s, balanceNGN: s.balanceNGN - withdrawAmount, transactions: [tx, ...s.transactions] }));
  }

  const nextOpenCountdown = !inWindow ? humanCountdown(millisToOpen) : "";
  const closeCountdown = inWindow ? humanCountdown(millisToClose) : "";

  return (
    <div className="min-h-screen w-full bg-gradient-to-br from-green-50 via-white to-green-100 dark:from-neutral-900 dark:via-neutral-900 dark:to-neutral-800 text-neutral-900 dark:text-neutral-100">
      <div className="max-w-3xl mx-auto p-6">
        <div className="flex items-center justify-between mb-6">
          <div className="flex items-center gap-3">
            <motion.div initial={{ scale: 0.9, opacity: 0 }} animate={{ scale: 1, opacity: 1 }} transition={{ type: "spring", stiffness: 200 }} className="h-12 w-12 rounded-2xl bg-green-600 shadow flex items-center justify-center text-white font-bold">
              ₦
            </motion.div>
            <div>
              <h1 className="text-2xl font-bold">NaijaSpend</h1>
              <p className="text-sm text-neutral-500">Naira rewards simulator — for testing and entertainment only.</p>
            </div>
          </div>
          {store.username && (
            <div className="flex items-center gap-2">
              <div className="text-right mr-2">
                <div className="text-xs text-neutral-500">Simulated balance</div>
                <div className="text-xl font-extrabold">₦{store.balanceNGN.toLocaleString()}</div>
              </div>
              <button onClick={handleLogout} className="px-3 py-2 rounded-2xl border border-neutral-200 dark:border-neutral-700 bg-white dark:bg-neutral-800">Log out</button>
            </div>
          )}
        </div>

        {!store.username ? (
          <motion.div initial={{ opacity: 0, y: 12 }} animate={{ opacity: 1, y: 0 }}>
            <div className="rounded-2xl shadow-xl bg-white/90 p-6 border border-neutral-200/60">
              <form onSubmit={handleLogin} className="space-y-4">
                <div className="flex items-center gap-3">
                  <div className="h-10 w-10 rounded-2xl bg-green-600 text-white flex items-center justify-center">N</div>
                  <div>
                    <h2 className="text-lg font-semibold">Create your NaijaSpend profile</h2>
                    <p className="text-sm text-neutral-500">Local simulator profile — no real money involved.</p>
                  </div>
                </div>
                <input value={usernameInput} onChange={(e) => setUsernameInput(e.target.value)} placeholder="Choose a username" className="w-full rounded-2xl border border-neutral-300 bg-white px-4 py-2" />
                <button type="submit" className="px-4 py-2 rounded-2xl bg-green-600 text-white">Enter NaijaSpend</button>
              </form>
            </div>
          </motion.div>
        ) : (
          <motion.div initial={{ opacity: 0, y: 12 }} animate={{ opacity: 1, y: 0 }} className="space-y-6">
            <div className="rounded-2xl shadow-xl bg-white/90 p-6 border border-neutral-200/60">
              <div className="flex items-center justify-between">
                <div>
                  <p className="text-sm text-neutral-500">Welcome,</p>
                  <h2 className="text-2xl font-bold">{store.username}</h2>
                </div>
                <div className="text-right">
                  <p className="text-sm text-neutral-500">Simulated balance</p>
                  <div className="text-3xl font-extrabold tracking-tight">₦{store.balanceNGN.toLocaleString()}</div>
                </div>
              </div>
              <div className="mt-4 text-sm text-neutral-500 flex items-center gap-2"><Clock className="h-4 w-4" /> Local time: {formatTime(now)}</div>
              <p className="mt-3 text-xs text-neutral-500">Statement: ₦50,000 was credited to your account for maintenance (simulated).</p>
            </div>

            <div className="rounded-2xl shadow p-6 bg-white/90 border border-neutral-200/60">
              <div className="flex items-center justify-between">
                <div>
                  <h3 className="text-lg font-semibold">Daily Withdrawal Claim</h3>
                  <p className="text-sm text-neutral-500">When you have at least ₦15,000, you can withdraw ₦5,000. Claim open at {store.claimTime} for {store.claimWindowMins} minutes.</p>
                </div>
                <div className="text-sm text-right">
                  <div>Window: <span className="font-mono">{store.claimTime}</span></div>
                  <div className="text-neutral-500">{todayKey(now)}</div>
                </div>
              </div>

              <div className="mt-4">
                {inWindow ? (
                  <div className="flex items-center justify-between">
                    <div className="text-sm text-neutral-600">Closes in {closeCountdown}</div>
                    <div className="flex gap-2">
                      <button onClick={performClaim} disabled={lastClaimWasToday} className={`px-4 py-2 rounded-2xl ${lastClaimWasToday ? "border" : "bg-green-600 text-white"}`}>
                        {lastClaimWasToday ? "Already claimed" : `Claim ₦${store.claimAmountNGN.toLocaleString()}`}
                      </button>
                      <button onClick={attemptWithdraw} disabled={store.balanceNGN < 15000} className={`px-4 py-2 rounded-2xl ${store.balanceNGN < 15000 ? "border" : "bg-yellow-500"}`}>
                        Withdraw ₦5,000
                      </button>
                    </div>
                  </div>
                ) : (
                  <div className="flex items-center justify-between">
                    <div className="text-sm text-neutral-600">Opens in {nextOpenCountdown}</div>
                    <button disabled className="px-4 py-2 rounded-2xl border">Waiting…</button>
                  </div>
                )}
              </div>

            </div>

            <div className="rounded-2xl shadow p-6 bg-white/90 border border-neutral-200/60">
              <div className="flex items-center justify-between mb-4">
                <div className="flex items-center gap-2"><List className="h-5 w-5" /><h3 className="text-lg font-semibold">Tasks & Surveys</h3></div>
                <div className="text-sm text-neutral-500">Complete tasks to earn simulated ₦100 per task</div>
              </div>

              <ul className="space-y-3">
                {tasks.map((t) => (
                  <li key={t.id} className="flex items-center justify-between bg-white p-3 rounded-2xl border">
                    <div>
                      <div className="font-medium">{t.title}</div>
                      <div className="text-xs text-neutral-500">Reward: ₦{t.reward.toLocaleString()}</div>
                      {t.kind === "video" && <div className="text-xs text-neutral-400">(Opens simulated video — does not play external content)</div>}
                      {t.kind === "social" && <div className="text-xs text-neutral-400">(Mock action — does not open Instagram)</div>}
                    </div>
                    <div>
                      <button onClick={() => completeTask(t.id, t.reward)} disabled={store.tasksDone.includes(t.id)} className={`px-3 py-2 rounded-2xl ${store.tasksDone.includes(t.id) ? "border" : "bg-green-600 text-white"}`}>
                        {store.tasksDone.includes(t.id) ? "Completed" : "Complete"}
                      </button>
                    </div>
                  </li>
                ))}
              </ul>
            </div>

            <div className="rounded-2xl shadow p-6 bg-white/90 border border-neutral-200/60">
              <h3 className="text-lg font-semibold mb-2">Transactions</h3>
              {store.transactions.length === 0 ? (
                <p className="text-sm text-neutral-500">No transactions yet.</p>
              ) : (
                <ul className="text-sm space-y-2">
                  {store.transactions.slice(0, 20).map((tx) => (
                    <li key={tx.id} className="flex items-center justify-between bg-white p-2 rounded-lg border">
                      <div>
                        <div className="font-medium">{tx.type === "credit" ? "Credit" : "Debit"}</div>
                        <div className="text-xs text-neutral-500">{new Date(tx.iso).toLocaleString()} — {tx.note}</div>
                      </div>
                      <div className={`font-semibold ${tx.type === "debit" ? "text-red-600" : "text-green-600"}`}>₦{Math.abs(tx.amount).toLocaleString()}</div>
                    </li>
                  ))}
                </ul>
              )}
            </div>

            <div className="rounded-2xl shadow p-6 bg-white/90 border border-neutral-200/60">
              <h3 className="text-lg font-semibold mb-2">Settings</h3>
              <div className="grid md:grid-cols-4 grid-cols-2 gap-3 items-end">
                <div>
                  <label className="text-xs text-neutral-500">Claim time (HH:mm)</label>
                  <input value={store.claimTime} onChange={(e) => setStore({ ...store, claimTime: e.target.value })} className="w-full rounded-2xl border px-3 py-2" />
                </div>
                <div>
                  <label className="text-xs text-neutral-500">Window (mins)</label>
                  <input type="number" value={String(store.claimWindowMins)} onChange={(e) => setStore({ ...store, claimWindowMins: Math.max(1, Number(e.target.value || 0)) })} className="w-full rounded-2xl border px-3 py-2" />
                </div>
                <div>
                  <label className="text-xs text-neutral-500">Claim amount (NGN)</label>
                  <input type="number" value={String(store.claimAmountNGN)} onChange={(e) => setStore({ ...store, claimAmountNGN: Number(e.target.value || 0) })} className="w-full rounded-2xl border px-3 py-2" />
                </div>
                <div>
                  <label className="text-xs text-neutral-500">Reset simulator</label>
                  <button onClick={() => setStore({ ...store, balanceNGN: 50000, tasksDone: [], lastClaimISO: null, transactions: [{ id: "init-credit", iso: new Date().toISOString(), amount: 50000, type: "credit", note: "Maintenance credit (simulated)" }] })} className="px-3 py-2 rounded-2xl border">Reset</button>
                </div>
              </div>
              <p className="mt-3 text-xs text-neutral-500">This is a simulator for entertainment and testing — it does not use real Naira and cannot send or receive funds.</p>
            </div>

          </motion.div>
        )}

        <div className="mt-8 text-xs text-neutral-500 text-center">NaijaSpend (simulator) — for fun and testing only.</div>
      </div>
    </div>
  );
}
