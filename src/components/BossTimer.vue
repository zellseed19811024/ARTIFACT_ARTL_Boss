<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

// 導入 BOSS 配置
import {
  bossConfig,
  getLootsByBossId,
  getAllLoots,
} from "@/config/bossConfig.js";

// 導入 Heroicons
import {
  TrashIcon,
  PlusCircleIcon,
  PlusIcon,
  XMarkIcon,
  MagnifyingGlassIcon,
  ClockIcon,
  MapPinIcon,
  SparklesIcon,
  ShareIcon,
  DocumentTextIcon,
  CheckBadgeIcon,
  FireIcon,
  BoltIcon,
  EyeIcon,
  ArrowDownTrayIcon,
} from "@heroicons/vue/24/outline";

import { StarIcon as StarSolidIcon } from "@heroicons/vue/24/solid";
import LZString from "lz-string";

// ===== 清除記錄(本地端所有記錄) =====

const clearRecordsLocal = () => {
  if (confirm("確定要清除所有記錄嗎？")) {
    localStorage.removeItem("killRecords");
    killRecords.value = [];
  }
};

// ===== 擊殺記錄列表 =====
const killRecords = ref([]);

// 排序後的記錄列表（按最早重生時間排序）
const sortedKillRecords = computed(() => {
  return [...killRecords.value].sort((a, b) => {
    const timeA = new Date(a.respawnTimeMin).getTime();
    const timeB = new Date(b.respawnTimeMin).getTime();
    return timeA - timeB; // 由早到晚排序
  });
});

// 按狀態分組的記錄
const groupedRecords = computed(() => {
  const respawned = []; // 已重生
  const respawning = []; // 可能已重生
  const waiting = []; // 即將重生

  sortedKillRecords.value.forEach((record) => {
    const status = getRecordCountdown(record).status;
    if (status === "respawned") {
      respawned.push(record);
    } else if (status === "respawning") {
      respawning.push(record);
    } else if (status === "waiting") {
      waiting.push(record);
    }
  });

  return { respawned, respawning, waiting };
});

// 各群組排序：時間（最早重生先）或頻道（數字順序，同頻道再依重生時間）
const groupSortMode = ref({
  respawned: "time",
  respawning: "time",
  waiting: "time",
});

// 「已重生」區塊預設收合，可手動展開
const showRespawnedSection = ref(false);

const sortGroupRecords = (records, sortBy) => {
  const list = [...records];
  if (sortBy === "channel") {
    list.sort((a, b) => {
      const c = String(a.channel).localeCompare(String(b.channel), "zh-Hant", {
        numeric: true,
        sensitivity: "base",
      });
      if (c !== 0) return c;
      return (
        new Date(a.respawnTimeMin).getTime() -
        new Date(b.respawnTimeMin).getTime()
      );
    });
  } else {
    list.sort((a, b) => {
      const t =
        new Date(a.respawnTimeMin).getTime() -
        new Date(b.respawnTimeMin).getTime();
      if (t !== 0) return t;
      return String(a.channel).localeCompare(String(b.channel), "zh-Hant", {
        numeric: true,
        sensitivity: "base",
      });
    });
  }
  return list;
};

const sortedGroupedRecords = computed(() => ({
  respawned: sortGroupRecords(
    groupedRecords.value.respawned,
    groupSortMode.value.respawned
  ),
  respawning: sortGroupRecords(
    groupedRecords.value.respawning,
    groupSortMode.value.respawning
  ),
  waiting: sortGroupRecords(
    groupedRecords.value.waiting,
    groupSortMode.value.waiting
  ),
}));

// 頻道排序時：與上一筆或下一筆頻道號差距 ≤ N 則列背景霓虹綠（N 可於頁尾調整）
const channelProximityN = ref(5);

const parseChannelNumber = (ch) => {
  const n = parseInt(String(ch ?? "").trim(), 10);
  return Number.isFinite(n) ? n : null;
};

const showChannelProximityRowBg = (groupKey, index) => {
  if (groupSortMode.value[groupKey] !== "channel") return false;
  const n = Math.max(0, Number(channelProximityN.value) || 0);
  const records = sortedGroupedRecords.value[groupKey];
  if (!records?.length || index < 0 || index >= records.length) return false;
  const cur = parseChannelNumber(records[index].channel);
  if (cur === null) return false;
  if (index > 0) {
    const prev = parseChannelNumber(records[index - 1].channel);
    if (prev !== null && Math.abs(cur - prev) <= n) return true;
  }
  if (index < records.length - 1) {
    const next = parseChannelNumber(records[index + 1].channel);
    if (next !== null && Math.abs(cur - next) <= n) return true;
  }
  return false;
};

// 新增一筆紀錄的彈窗
const isAddRecordModalOpen = ref(false);
const addRecordForm = ref({
  boss: "",
  channel: "",
  killTime: "",
  killLocation: "",
  loot: "",
});

// 頻道輸入框的 ref
const channelInputRef = ref(null);

// 自定義下拉選單狀態
const isDropdownOpen = ref(false);
const dropdownSearch = ref("");

// ===== BOSS 最愛（收藏後置頂）=====
const favoriteBossStorageKey = "favoriteBossIds";
const favoriteBossIds = ref([]);
const favoriteBossIdSet = computed(() => new Set(favoriteBossIds.value));

const loadFavoriteBossIds = () => {
  try {
    const raw = localStorage.getItem(favoriteBossStorageKey);
    if (!raw) return;
    const parsed = JSON.parse(raw);
    if (Array.isArray(parsed)) {
      favoriteBossIds.value = parsed.filter(
        (id) => typeof id === "string" && id.length > 0
      );
    }
  } catch (_) {
    // ignore
  }
};

const saveFavoriteBossIds = () => {
  localStorage.setItem(
    favoriteBossStorageKey,
    JSON.stringify(favoriteBossIds.value)
  );
};

const toggleFavoriteBoss = (bossId) => {
  if (!bossId) return;
  const list = [...favoriteBossIds.value];
  const idx = list.indexOf(bossId);
  if (idx >= 0) {
    list.splice(idx, 1);
  } else {
    // 新增到最前面（最愛置頂）
    list.unshift(bossId);
  }
  favoriteBossIds.value = list;
  saveFavoriteBossIds();
};

const isBossFavorite = (bossId) => favoriteBossIdSet.value.has(bossId);

// 擊殺時間的「時」「分」拆開給 24 小時制輸入用
const killTimeHour = computed(() => {
  const t = addRecordForm.value.killTime;
  if (!t || !t.includes(":")) return "";
  return t.split(":")[0] || "";
});
const killTimeMinute = computed(() => {
  const t = addRecordForm.value.killTime;
  if (!t || !t.includes(":")) return "";
  return t.split(":")[1] || "";
});
const onKillTimeHourInput = (e) => {
  const v = e.target.value;
  const m = (addRecordForm.value.killTime && addRecordForm.value.killTime.split(":")[1]) || "00";
  if (v === "") {
    addRecordForm.value.killTime = `00:${m}`;
    return;
  }
  const n = parseInt(v, 10);
  if (!Number.isNaN(n)) {
    const h = String(Math.min(23, Math.max(0, n))).padStart(2, "0");
    addRecordForm.value.killTime = `${h}:${m}`;
  }
};
const onKillTimeMinuteInput = (e) => {
  const v = e.target.value;
  const h = (addRecordForm.value.killTime && addRecordForm.value.killTime.split(":")[0]) || "00";
  if (v === "") {
    addRecordForm.value.killTime = `${h}:00`;
    return;
  }
  const n = parseInt(v, 10);
  if (!Number.isNaN(n)) {
    const m = String(Math.min(59, Math.max(0, n))).padStart(2, "0");
    addRecordForm.value.killTime = `${h}:${m}`;
  }
};

// 根據選擇的 BOSS 判斷是否需要顯示地點欄位
const selectedBossConfig = computed(() => {
  if (!addRecordForm.value.boss) return null;
  return bossConfig.find((b) => b.id === addRecordForm.value.boss);
});

// 過濾後的 BOSS 列表（根據搜尋）
const filteredBosses = computed(() => {
  const base = !dropdownSearch.value
    ? bossConfig
    : bossConfig.filter((boss) => {
        const search = dropdownSearch.value.toLowerCase();
        return (
          boss.name.toLowerCase().includes(search) ||
          boss.description.toLowerCase().includes(search)
        );
      });

  // 收藏置頂：依 favoriteBossIds 的順序排在最前面
  const favorites = base.filter((boss) => favoriteBossIdSet.value.has(boss.id));
  favorites.sort(
    (a, b) =>
      favoriteBossIds.value.indexOf(a.id) - favoriteBossIds.value.indexOf(b.id)
  );
  const nonFavorites = base.filter(
    (boss) => !favoriteBossIdSet.value.has(boss.id)
  );
  return [...favorites, ...nonFavorites];
});

// 用於彈窗內切換 BOSS（numpad+ / numpad-）
const bossCycleList = computed(() => {
  const favorites = favoriteBossIds.value
    .map((id) => bossConfig.find((b) => b.id === id))
    .filter(Boolean);
  const favoriteSet = new Set(favorites.map((b) => b.id));
  const nonFavorites = bossConfig.filter((b) => !favoriteSet.has(b.id));
  return [...favorites, ...nonFavorites];
});

const cycleSelectedBoss = (delta) => {
  const list = bossCycleList.value;
  if (!list.length) return;
  const curId = addRecordForm.value.boss;
  let idx = list.findIndex((b) => b.id === curId);
  if (idx < 0) idx = 0;
  const nextIdx = (idx + delta + list.length) % list.length;
  addRecordForm.value.boss = list[nextIdx].id;
};

// 選擇 BOSS
const selectBoss = (bossId) => {
  addRecordForm.value.boss = bossId;
  isDropdownOpen.value = false;
  dropdownSearch.value = "";
};

// 取得 BOSS 圖片 URL
const getBossImage = (bossId) => {
  try {
    return new URL(`../assets/images/${bossId}.png`, import.meta.url).href;
  } catch (e) {
    console.error(`無法載入圖片: ${bossId}.png`, e);
    return ""; // 返回空字串，讓圖片載入失敗時不顯示
  }
};

const openAddRecordModal = () => {
  console.log("openAddRecordModal called, current boss:", addRecordForm.value.boss);
  
  if (!addRecordForm.value.boss) {
    alert("請先選擇一個 BOSS");
    return;
  }

  isAddRecordModalOpen.value = true;
  // 获取当前时间并格式化为 HH:mm 格式（input type="time" 需要的格式）
  const now = new Date();
  const hours = String(now.getHours()).padStart(2, "0");
  const minutes = String(now.getMinutes()).padStart(2, "0");
  const timeString = `${hours}:${minutes}`;

  // 保留已选择的 boss，只更新其他字段
  const currentBoss = addRecordForm.value.boss;
  addRecordForm.value = {
    boss: currentBoss, // 保留已选择的 boss
    channel: "",
    killTime: timeString,
    killLocation: "",
    loot: "",
  };

  console.log("After setting, boss is:", addRecordForm.value.boss);

  // 等待 DOM 更新後，自動 focus 到頻道輸入框
  setTimeout(() => {
    if (channelInputRef.value) {
      channelInputRef.value.focus();
    }
  }, 100);
};

const closeAddRecordModal = () => {
  isAddRecordModalOpen.value = false;
};

const addRecord = () => {
  if (
    !addRecordForm.value.boss ||
    !addRecordForm.value.channel ||
    !addRecordForm.value.killTime
  ) {
    alert("請填寫必填欄位");
    return;
  }

  const bossInfo = bossConfig.find((b) => b.id === addRecordForm.value.boss);

  // 將時間字串轉換為完整的日期時間（格式須為 HH:mm，與擊殺時間 24 小時制輸入一致）
  const timeParts = addRecordForm.value.killTime.split(":");
  const hours = parseInt(timeParts[0], 10);
  const minutes = parseInt(timeParts[1], 10);
  if (timeParts.length !== 2 || Number.isNaN(hours) || Number.isNaN(minutes) || hours < 0 || hours > 23 || minutes < 0 || minutes > 59) {
    alert("擊殺時間格式不正確，請使用 24 小時制（時:分）");
    return;
  }
  const killDateTime = new Date();
  killDateTime.setHours(hours, minutes, 0, 0);

  // 計算重生時間區間（最早和最晚）
  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );

  // 檢查是否已存在相同 BOSS 和頻道的記錄
  const existingRecordIndex = killRecords.value.findIndex(
    (record) =>
      record.bossId === addRecordForm.value.boss &&
      record.channel === addRecordForm.value.channel
  );

  const recordData = {
    bossId: addRecordForm.value.boss,
    bossName: bossInfo.name,
    channel: addRecordForm.value.channel,
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: addRecordForm.value.killLocation,
    loot: addRecordForm.value.loot,
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
    checkPoints: [], // 踩點記錄
  };

  if (existingRecordIndex !== -1) {
    // 如果存在，更新該記錄（保留原有的 id 和踩點記錄）
    killRecords.value[existingRecordIndex] = {
      ...recordData,
      id: killRecords.value[existingRecordIndex].id,
      checkPoints: killRecords.value[existingRecordIndex].checkPoints || [],
    };
  } else {
    // 如果不存在，新增記錄
    const newRecord = {
      ...recordData,
      id: Date.now(),
      checkPoints: [],
    };
    killRecords.value.unshift(newRecord); // 新記錄放在最前面
  }

  saveRecords();

  isAddRecordModalOpen.value = false;
  addRecordForm.value = {
    boss: addRecordForm.value.boss,
    channel: "",
    killTime: "",
    killLocation: "",
    loot: "",
  };
};

// 成功擊殺 新增紀錄
const reAddRecord = (record) => {
  const bossInfo = bossConfig.find((b) => b.id === record.bossId);

  if (!bossInfo) {
    alert("找不到 BOSS 資料");
    return;
  }

  // 獲取當前時間
  const killDateTime = new Date();

  // 計算重生時間區間（最早和最晚）
  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );

  // 檢查是否已存在相同 BOSS 和頻道的記錄
  const existingRecordIndex = killRecords.value.findIndex(
    (r) => r.bossId === record.bossId && r.channel === record.channel
  );

  const recordData = {
    bossId: record.bossId,
    bossName: record.bossName,
    channel: record.channel,
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: record.killLocation || "",
    loot: record.loot || "",
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
    checkPoints: [], // 重置踩點記錄
  };

  if (existingRecordIndex !== -1) {
    // 如果存在，更新該記錄（保留原有的 id）
    killRecords.value[existingRecordIndex] = {
      ...recordData,
      id: killRecords.value[existingRecordIndex].id,
      checkPoints: [], // 重置踩點記錄
    };
  } else {
    // 如果不存在，新增記錄
    const newRecord = {
      ...recordData,
      id: Date.now(),
      checkPoints: [],
    };
    killRecords.value.unshift(newRecord); // 新記錄放在最前面
  }

  saveRecords();
};

// ===== 踩點記錄功能 =====
// 添加踩點記錄
const addCheckPoint = (record) => {
  if (!record.checkPoints) {
    record.checkPoints = [];
  }

  const checkPoint = {
    time: new Date().toISOString(),
    note: "已查看，沒有 BOSS",
  };

  record.checkPoints.push(checkPoint);
  saveRecords();
};

// 刪除單個踩點記錄
const deleteCheckPoint = (record, index) => {
  if (!record.checkPoints) return;

  // 反轉索引（因為顯示時是反轉的）
  const actualIndex = record.checkPoints.length - 1 - index;
  record.checkPoints.splice(actualIndex, 1);
  saveRecords();
};

// 獲取最近的踩點時間
const getLatestCheckPoint = (record) => {
  if (!record.checkPoints || record.checkPoints.length === 0) {
    return null;
  }
  return record.checkPoints[record.checkPoints.length - 1];
};

// 格式化踩點時間顯示
const formatCheckPointTime = (checkPoint) => {
  if (!checkPoint) return "";
  const time = new Date(checkPoint.time);
  const now = new Date();

  // 獲取 HH:MM 格式
  const hours = String(time.getHours()).padStart(2, "0");
  const minutes = String(time.getMinutes()).padStart(2, "0");
  const timeStr = `${hours}:${minutes}`;

  // 計算距離現在多久
  const diffMs = now - time;
  const diffMins = Math.floor(diffMs / 60000);

  if (diffMins < 1) {
    return `剛剛踩點`;
  } else if (diffMins < 60) {
    return `${diffMins} 分鐘前踩點`;
  } else {
    const diffHours = Math.floor(diffMins / 60);
    return `${diffHours} 小時前踩點`;
  }
};

// BOSS 資料（保留原有的，暫時不刪除以免影響其他功能）
const bosses = ref([]);

const currentTime = ref(Date.now());
let intervalId = null;

// 儲存記錄
const saveRecords = () => {
  localStorage.setItem("killRecords", JSON.stringify(killRecords.value));
};

// 載入記錄
const loadRecords = () => {
  const saved = localStorage.getItem("killRecords");
  if (saved) {
    killRecords.value = JSON.parse(saved);
  }
};

// ===== 分享 / 匯入記錄 =====
const isImportModalOpen = ref(false);
const importPasteText = ref("");

// 將記錄編碼為可分享字串（LZ 壓縮）
// v3 payload：更精簡，時間改為 Unix 秒，踩點僅保留最近 1 筆
const SHARE_PAYLOAD_VERSION = 3;

const compactRecordsForShare = (records) => {
  const rows = records
    .map((r) => {
      if (!r || typeof r.bossId !== "string" || !r.killTime) return null;
      const killMs = new Date(r.killTime).getTime();
      if (Number.isNaN(killMs)) return null;

      const compact = [
        r.bossId, // b
        String(r.channel ?? ""), // c
        Math.floor(killMs / 1000), // t (unix seconds)
      ];

      const cpTimes =
        Array.isArray(r.checkPoints) && r.checkPoints.length > 0
          ? r.checkPoints
              .map((cp) => cp?.time)
              .filter((t) => typeof t === "string" && t.length > 0)
          : [];

      // 踩點分享只保留最近 1 筆
      if (cpTimes.length > 0) {
        const lastCpMs = new Date(cpTimes[cpTimes.length - 1]).getTime();
        if (!Number.isNaN(lastCpMs)) {
          compact.push(Math.floor(lastCpMs / 1000)); // p
        }
      }

      return compact;
    })
    .filter(Boolean);

  return {
    v: SHARE_PAYLOAD_VERSION,
    r: rows,
  };
};

const encodeShareData = (records) => {
  const json = JSON.stringify(compactRecordsForShare(records));
  return LZString.compressToEncodedURIComponent(json);
};

// 解碼分享字串：優先當作 LZ，失敗再嘗試 base64 與純 JSON（向下相容）
const decodeShareData = (raw) => {
  const trimmed = raw.trim();
  if (!trimmed) return null;

  // v3: LZString.compressToEncodedURIComponent
  try {
    const lz = LZString.decompressFromEncodedURIComponent(trimmed);
    if (lz) return JSON.parse(lz);
  } catch (_) {}

  let jsonStr = trimmed;
  // 舊版：base64
  try {
    if (/^[A-Za-z0-9+/=]+$/.test(trimmed)) {
      jsonStr = decodeURIComponent(escape(atob(trimmed)));
    }
  } catch (_) {}
  // 舊版：純 JSON
  try {
    return JSON.parse(jsonStr);
  } catch (_) {
    return null;
  }
};

const toAppRecordByBossId = (bossId, channel, killTime, checkPointTimes = []) => {
  const bossInfo = bossConfig.find((b) => b.id === bossId);
  if (!bossInfo) return null;
  const killDateTime = new Date(killTime);
  if (Number.isNaN(killDateTime.getTime())) return null;

  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );

  const checkPoints = Array.isArray(checkPointTimes)
    ? checkPointTimes
        .filter((t) => typeof t === "string" && t.length > 0)
        .map((t) => ({ time: t, note: "已查看，沒有 BOSS" }))
    : [];

  return {
    bossId: bossInfo.id,
    bossName: bossInfo.name,
    channel: String(channel ?? ""),
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: "",
    loot: "",
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
    checkPoints,
    id: Date.now() + Math.random(),
  };
};

/** 兼容舊格式（完整 record 陣列）與新瘦身格式（v2/v3） */
const convertSharePayloadToKillRecords = (parsed) => {
  const out = [];
  let skipped = 0;

  // v3：{ v:3, r:[[bossId, channel, killUnixSec, latestCheckPointUnixSec?], ...] }
  if (
    parsed &&
    typeof parsed === "object" &&
    parsed.v === 3 &&
    Array.isArray(parsed.r)
  ) {
    parsed.r.forEach((row) => {
      const bossId = row?.[0];
      const channel = row?.[1];
      const killUnixSec = row?.[2];
      const cpUnixSec = row?.[3];
      const killIso =
        typeof killUnixSec === "number" || /^\d+$/.test(String(killUnixSec))
          ? new Date(Number(killUnixSec) * 1000).toISOString()
          : null;
      const cpIso =
        cpUnixSec !== undefined &&
        cpUnixSec !== null &&
        (typeof cpUnixSec === "number" || /^\d+$/.test(String(cpUnixSec)))
          ? [new Date(Number(cpUnixSec) * 1000).toISOString()]
          : [];

      const record = killIso
        ? toAppRecordByBossId(bossId, channel, killIso, cpIso)
        : null;
      if (record) out.push(record);
      else skipped++;
    });
    return { records: out, skipped };
  }

  // v2：{ v:2, r:[{ b,c,t,p? }] }
  if (parsed && typeof parsed === "object" && Array.isArray(parsed.r)) {
    parsed.r.forEach((row) => {
      const record = toAppRecordByBossId(row?.b, row?.c, row?.t, row?.p ?? []);
      if (record) out.push(record);
      else skipped++;
    });
    return { records: out, skipped };
  }

  // 舊格式：直接為完整 killRecords 陣列
  if (Array.isArray(parsed)) {
    parsed.forEach((r) => {
      const record = toAppRecordByBossId(
        r?.bossId,
        r?.channel,
        r?.killTime,
        Array.isArray(r?.checkPoints)
          ? r.checkPoints.map((cp) => cp?.time).filter(Boolean)
          : []
      );
      if (record) out.push(record);
      else skipped++;
    });
    return { records: out, skipped };
  }

  return { records: [], skipped };
};

// 分享：複製到剪貼簿
const shareRecords = async () => {
  if (killRecords.value.length === 0) {
    alert("目前沒有記錄可分享");
    return;
  }
  const encoded = encodeShareData(killRecords.value);
  try {
    await navigator.clipboard.writeText(encoded);
    alert("已複製到剪貼簿！可貼上傳給他人，對方用「匯入分享資料」即可一鍵匯入。");
  } catch (e) {
    alert("複製失敗，請手動複製：\n\n" + encoded);
  }
};

// 匯入時合併：同一 BOSS + 頻道只保留較新的擊殺記錄，其餘加總
const mergeImportedRecords = (existing, imported) => {
  if (!Array.isArray(imported)) return existing;
  const byKey = {};
  const key = (r) => `${r.bossId}_${r.channel}`;
  existing.forEach((r) => {
    byKey[key(r)] = { ...r };
  });
  imported.forEach((r) => {
    const k = key(r);
    const existingRecord = byKey[k];
    if (!existingRecord) {
      byKey[k] = { ...r, id: r.id ?? Date.now() + Math.random() };
    } else {
      const existingTime = new Date(existingRecord.killTime).getTime();
      const importedTime = new Date(r.killTime).getTime();
      if (importedTime > existingTime) {
        byKey[k] = { ...r, id: existingRecord.id };
      }
    }
  });
  const list = Object.values(byKey);
  list.sort((a, b) => new Date(a.respawnTimeMin).getTime() - new Date(b.respawnTimeMin).getTime());
  return list;
};

const openImportModal = () => {
  importPasteText.value = "";
  isImportModalOpen.value = true;
};

const closeImportModal = () => {
  isImportModalOpen.value = false;
  importPasteText.value = "";
};

const doImport = () => {
  const parsed = decodeShareData(importPasteText.value);
  if (!parsed) {
    alert("無法辨識的資料，請貼上他人分享的完整內容。");
    return;
  }
  const { records, skipped } = convertSharePayloadToKillRecords(parsed);
  if (!records.length) {
    alert("沒有可匯入的資料（格式不符，或 BOSS 已不在目前設定）。");
    return;
  }
  const merged = mergeImportedRecords(killRecords.value, records);
  killRecords.value = merged;
  saveRecords();
  closeImportModal();
  let msg = `已匯入並合併，目前共 ${merged.length} 筆記錄。`;
  if (skipped > 0) msg += `（略過 ${skipped} 筆）`;
  alert(msg);
};

// ===== 匯入 LZString（compressToEncodedURIComponent）字串 =====
const isLzImportModalOpen = ref(false);
const lzImportPasteText = ref("");

const openLzImportModal = () => {
  lzImportPasteText.value = "";
  isLzImportModalOpen.value = true;
};

const closeLzImportModal = () => {
  isLzImportModalOpen.value = false;
  lzImportPasteText.value = "";
};

const findBossConfigByImportName = (name) => {
  if (name == null || String(name).trim() === "") return null;
  const n = String(name).trim();
  return bossConfig.find((b) => b.name === n) ?? null;
};

const buildKillRecordFromBossInfo = (bossInfo, channel, killDateTime) => {
  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );
  return {
    bossId: bossInfo.id,
    bossName: bossInfo.name,
    channel: String(channel),
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: "",
    loot: "",
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
    checkPoints: [],
    id: Date.now() + Math.random(),
  };
};

/** 將 LZ 解壓後的 JSON 轉成本程式 killRecords 陣列；支援本程式陣列格式或 { key: [{ name, channel, timestamp }] } */
const convertLzDecompressedToKillRecords = (parsed) => {
  const out = [];
  let skipped = 0;

  const pushAppRecord = (r) => {
    const record = toAppRecordByBossId(
      r?.bossId,
      r?.channel,
      r?.killTime,
      Array.isArray(r?.checkPoints)
        ? r.checkPoints.map((cp) => cp?.time).filter(Boolean)
        : []
    );
    if (!record) {
      skipped++;
      return;
    }
    out.push(record);
  };

  const pushLzItem = (item) => {
    if (!item || typeof item !== "object") {
      skipped++;
      return;
    }
    const name = item.name;
    const channel = item.channel;
    const ts = item.timestamp;
    if (name == null || channel === undefined || channel === null || !ts) {
      skipped++;
      return;
    }
    const bossInfo = findBossConfigByImportName(name);
    if (!bossInfo) {
      skipped++;
      return;
    }
    const killDateTime = new Date(ts);
    if (Number.isNaN(killDateTime.getTime())) {
      skipped++;
      return;
    }
    out.push(buildKillRecordFromBossInfo(bossInfo, channel, killDateTime));
  };

  if (Array.isArray(parsed)) {
    if (
      parsed.length > 0 &&
      parsed[0] &&
      typeof parsed[0].bossId === "string" &&
      parsed[0].killTime
    ) {
      parsed.forEach(pushAppRecord);
      return { records: out, skipped };
    }
    parsed.forEach(pushLzItem);
    return { records: out, skipped };
  }

  if (parsed && typeof parsed === "object") {
    for (const val of Object.values(parsed)) {
      const list = Array.isArray(val) ? val : val != null ? [val] : [];
      list.forEach(pushLzItem);
    }
    return { records: out, skipped };
  }

  return { records: [], skipped: 0 };
};

/** 若貼上完整 artale.app 連結，取出 ?data= 後的 LZ 字串；否則原樣使用 */
const normalizeLzImportPayload = (input) => {
  const trimmed = input.trim();
  if (!trimmed) return "";

  let href = trimmed;
  if (!/^https?:\/\//i.test(href) && /^artale\.app\//i.test(href)) {
    href = `https://${href}`;
  }

  try {
    const u = new URL(href);
    if (u.hostname === "artale.app" && u.pathname === "/boss") {
      const data = u.searchParams.get("data");
      if (data != null && data !== "") return data;
    }
  } catch (_) {
    /* 非完整 URL，改走字首剝除 */
  }

  const httpsPrefix = "https://artale.app/boss?data=";
  if (trimmed.startsWith(httpsPrefix)) {
    return trimmed.slice(httpsPrefix.length);
  }
  const httpPrefix = "http://artale.app/boss?data=";
  if (trimmed.toLowerCase().startsWith(httpPrefix)) {
    return trimmed.slice(httpPrefix.length);
  }

  return trimmed;
};

const doLzImport = () => {
  const raw = normalizeLzImportPayload(lzImportPasteText.value);
  if (!raw) {
    alert("請貼上 LZURL 字串。");
    return;
  }
  const decompressed = LZString.decompressFromEncodedURIComponent(raw);
  if (decompressed == null || decompressed === "") {
    alert("解壓縮失敗，請確認是否為 LZString.compressToEncodedURIComponent 產生的字串。");
    return;
  }
  let parsed;
  try {
    parsed = JSON.parse(decompressed);
  } catch {
    alert("解壓後內容不是合法 JSON。");
    return;
  }
  const { records, skipped } = convertLzDecompressedToKillRecords(parsed);
  if (!records.length) {
    alert("沒有可匯入的資料（格式不符，或 BOSS 名稱與本工具設定對不到）。");
    return;
  }
  const merged = mergeImportedRecords(killRecords.value, records);
  killRecords.value = merged;
  saveRecords();
  closeLzImportModal();
  let msg = `已匯入並合併，目前共 ${merged.length} 筆記錄。`;
  if (skipped > 0) msg += `（略過 ${skipped} 筆）`;
  alert(msg);
};

// 點擊外部關閉下拉選單
const closeDropdownOnClickOutside = (event) => {
  const dropdown = document.querySelector(".relative.w-96");
  if (dropdown && !dropdown.contains(event.target)) {
    isDropdownOpen.value = false;
  }
};

// 全局鍵盤快捷鍵
const handleKeyboardShortcut = (event) => {
  const target = event.target;

  // 彈窗開啟後：在「頻道輸入框」使用 numpad+ / numpad- 切換 BOSS
  const isModalOpen = isAddRecordModalOpen.value;
  const channelEl = channelInputRef.value;
  const isChannelFocused =
    isModalOpen &&
    channelEl &&
    (target === channelEl || document.activeElement === channelEl);

  if (isChannelFocused) {
    if (event.code === "NumpadAdd") {
      event.preventDefault();
      cycleSelectedBoss(1);
      return;
    }
    if (event.code === "NumpadSubtract") {
      event.preventDefault();
      cycleSelectedBoss(-1);
      return;
    }
    // 使用中括號也可切換 Boss（例如不方便用小鍵盤時）
    if (event.code === "BracketLeft" || event.key === "[") {
      event.preventDefault();
      cycleSelectedBoss(-1);
      return;
    }
    if (event.code === "BracketRight" || event.key === "]") {
      event.preventDefault();
      cycleSelectedBoss(1);
      return;
    }
  }

  // 其他輸入框情境：不觸發快捷鍵
  if (target?.tagName === "INPUT" || target?.tagName === "TEXTAREA")
    return;

  // 主鍵盤：+（常與 Shift 同按）或 =；或數字鍵盤：NumpadAdd
  if (event.key === "+" || event.key === "=" || event.code === "NumpadAdd") {
    event.preventDefault();
    if (!isAddRecordModalOpen.value) openAddRecordModal();
  }
};

// 載入儲存的資料
onMounted(() => {
  loadRecords();
  loadFavoriteBossIds();

  const saved = localStorage.getItem("bossTimers");
  if (saved) {
    const data = JSON.parse(saved);
    bosses.value = data.map((boss) => ({
      ...boss,
      lastKillTime: boss.lastKillTime ? new Date(boss.lastKillTime) : null,
    }));
  }

  // 每秒更新時間
  intervalId = setInterval(() => {
    currentTime.value = Date.now();
  }, 1000);

  // 監聽點擊事件來關閉下拉選單
  document.addEventListener("click", closeDropdownOnClickOutside);
  
  // 監聽鍵盤快捷鍵
  document.addEventListener("keydown", handleKeyboardShortcut);
});

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
  // 清除事件監聽器
  document.removeEventListener("click", closeDropdownOnClickOutside);
  document.removeEventListener("keydown", handleKeyboardShortcut);
});

// 儲存資料（保留原有的）
const saveData = () => {
  localStorage.setItem("bossTimers", JSON.stringify(bosses.value));
};

// 記錄擊殺時間
const recordKill = (boss) => {
  boss.lastKillTime = new Date();
  saveData();
};

// 重置計時器
const resetTimer = (boss) => {
  boss.lastKillTime = null;
  saveData();
};

// 計算剩餘時間
const getRemainingTime = (boss) => {
  if (!boss.lastKillTime) return null;

  const killTime = new Date(boss.lastKillTime).getTime();
  const respawnTime = killTime + boss.spawnTime * 1000;
  const remaining = respawnTime - currentTime.value;

  if (remaining <= 0) return 0;
  return Math.floor(remaining / 1000);
};

// 格式化時間顯示
const formatTime = (seconds) => {
  if (seconds === null) return "--:--:--";

  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;

  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(
    2,
    "0"
  )}:${String(secs).padStart(2, "0")}`;
};

// 格式化重生時間
const formatRespawnTime = (boss) => {
  if (!boss.lastKillTime) return "尚未擊殺";

  const killTime = new Date(boss.lastKillTime).getTime();
  const respawnTime = new Date(killTime + boss.spawnTime * 1000);

  return respawnTime.toLocaleString("zh-TW", {
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    hour12: false,
  });
};

// 判斷是否已重生
const isRespawned = (boss) => {
  const remaining = getRemainingTime(boss);
  return remaining !== null && remaining === 0;
};

// 新增 BOSS
const addBoss = () => {
  const newId = Math.max(...bosses.value.map((b) => b.id), 0) + 1;
  bosses.value.push({
    id: newId,
    name: `BOSS ${newId}`,
    spawnTime: 30 * 60,
    lastKillTime: null,
    note: "",
  });
  saveData();
};

// 刪除 BOSS
const removeBoss = (bossId) => {
  bosses.value = bosses.value.filter((b) => b.id !== bossId);
  saveData();
};

// 編輯模式
const editingBoss = ref(null);
const editForm = ref({
  name: "",
  spawnTime: 30,
  note: "",
});

const startEdit = (boss) => {
  editingBoss.value = boss.id;
  editForm.value = {
    name: boss.name,
    spawnTime: boss.spawnTime / 60,
    note: boss.note,
  };
};

const saveEdit = (boss) => {
  boss.name = editForm.value.name;
  boss.spawnTime = editForm.value.spawnTime * 60;
  boss.note = editForm.value.note;
  editingBoss.value = null;
  saveData();
};

const cancelEdit = () => {
  editingBoss.value = null;
};

// ===== 記錄列表相關功能 =====

const isDeleteConfirmModalOpen = ref(false);
const pendingDeleteRecord = ref(null);

const requestDeleteRecord = (record) => {
  pendingDeleteRecord.value = record;
  isDeleteConfirmModalOpen.value = true;
};

const closeDeleteConfirmModal = () => {
  isDeleteConfirmModalOpen.value = false;
  pendingDeleteRecord.value = null;
};

const confirmDeleteRecord = () => {
  const target = pendingDeleteRecord.value;
  if (!target) return;

  killRecords.value = killRecords.value.filter(
    (r) => !(r.bossId === target.bossId && String(r.channel) === String(target.channel))
  );
  saveRecords();
  closeDeleteConfirmModal();
};

// 格式化重生時間區間：若最早與最晚相同（固定重生時間）只顯示一個時間
const formatRespawnTimeRange = (record) => {
  const minT = new Date(record.respawnTimeMin).getTime();
  const maxT = new Date(record.respawnTimeMax).getTime();
  const str = formatDisplayTime(record.respawnTimeMin);
  if (minT === maxT) return str;
  return `${str} ~ ${formatDisplayTime(record.respawnTimeMax)}`;
};

// 格式化顯示時間
const formatDisplayTime = (isoString) => {
  const date = new Date(isoString);
  const now = new Date();

  // 判斷是否為當天
  const isToday =
    date.getFullYear() === now.getFullYear() &&
    date.getMonth() === now.getMonth() &&
    date.getDate() === now.getDate();

  // 如果是當天，只顯示時間
  if (isToday) {
    return date.toLocaleString("zh-TW", {
      hour: "2-digit",
      minute: "2-digit",
      hour12: false,
    });
  }

  // 如果不是當天，顯示日期和時間
  return date.toLocaleString("zh-TW", {
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });
};

// 計算記錄的倒數計時
const getRecordCountdown = (record) => {
  const now = currentTime.value;
  const respawnMin = new Date(record.respawnTimeMin).getTime();
  const respawnMax = new Date(record.respawnTimeMax).getTime();

  // 如果還沒到最早重生時間
  if (now < respawnMin) {
    const remaining = Math.floor((respawnMin - now) / 1000);
    return {
      status: "waiting",
      text: "即將重生",
      timeText: formatCountdownTime(remaining),
      remaining,
      percentage: 0,
    };
  }

  // 如果在重生時間範圍內
  if (now >= respawnMin && now < respawnMax) {
    const remaining = Math.floor((respawnMax - now) / 1000);
    const totalDuration = respawnMax - respawnMin;
    const elapsed = now - respawnMin;
    const percentage = Math.min(
      100,
      Math.max(0, (elapsed / totalDuration) * 100)
    );

    return {
      status: "respawning",
      text: "可能已重生",
      timeText: formatCountdownTime(remaining),
      remaining,
      percentage: Math.round(percentage),
    };
  }

  // 如果已過最晚重生時間
  return {
    status: "respawned",
    text: "已重生",
    timeText: "00:00:00",
    remaining: 0,
    percentage: 100,
  };
};

// 格式化倒數時間
const formatCountdownTime = (seconds) => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;

  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(
    2,
    "0"
  )}:${String(secs).padStart(2, "0")}`;
};
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl min-w-0">
    <!-- 標題 & 功能按鈕 (清除、分享、使用說明) -->
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-2xl font-bold text-white">ARTIFACT 野王計時器 v1.0</h1>
      <div class="flex flex-wrap gap-2">
        <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
          @click="shareRecords"
          title="複製目前所有記錄，可貼上分享給他人"
        >
          <ShareIcon class="w-4 h-4" />
          分享資料
        </button>
        <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
          @click="openImportModal"
          title="貼上他人分享的資料，一鍵匯入並與現有記錄合併"
        >
          <DocumentTextIcon class="w-4 h-4" />
          匯入分享資料
        </button>
        <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
          @click="openLzImportModal"
          title="貼上 LZString.compressToEncodedURIComponent 壓縮字串，解壓後轉成紀錄並合併"
        >
          <ArrowDownTrayIcon class="w-4 h-4" />
          匯入LZURLSTR
        </button>
        <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
          @click="clearRecordsLocal"
        >
          <TrashIcon class="w-4 h-4" />
          清除所有紀錄
        </button>
        <!-- <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
        >
          <ShareIcon class="w-4 h-4" />
          分享
        </button> -->
        <!-- <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
        >
          <DocumentTextIcon class="w-4 h-4" />
          使用說明
        </button> -->
      </div>
    </div>

    <!-- 
    * 新增一筆紀錄
      選擇一個BOSS 新增按鈕
      彈窗填寫資料：頻道、擊殺時間(預設當下 可調整)、擊殺地點(特定 BOSS 才要)、收穫的寶物(後續再加)
    -->
    <div
      class="flex flex-col items-end fixed right-0 bottom-0 gap-2 bg-gray-950 bg-opacity-50 backdrop-blur-sm p-3 w-full max-w-[480px] ml-auto z-20 rounded-tl-lg"
    >
      <div class="flex justify-end gap-2 w-full">
        <!-- 自定義下拉選單 -->
        <div class="relative w-full">
          <!-- 選單按鈕 -->
          <button
            @click="isDropdownOpen = !isDropdownOpen"
            class="w-full bg-gray-700 text-white px-4 py-2 rounded-md flex items-center justify-between hover:bg-gray-600 transition"
          >
            <span v-if="!selectedBossConfig" class="text-gray-400"
              >選擇一個BOSS</span
            >
            <span v-else class="flex items-center gap-2">
              <div class="w-8 h-8 flex-shrink-0">
                <img
                  :src="getBossImage(selectedBossConfig.id)"
                  :alt="selectedBossConfig.name"
                  class="w-full h-full object-contain"
                />
              </div>
              <span>{{ selectedBossConfig.name }}</span>
              <span class="text-xs text-gray-400"
                >({{ selectedBossConfig.description }})</span
              >
            </span>
            <svg
              class="w-5 h-5 transition-transform"
              :class="{ 'rotate-180': isDropdownOpen }"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M19 9l-7 7-7-7"
              ></path>
            </svg>
          </button>

          <!-- 下拉選單內容 -->
          <div
            v-show="isDropdownOpen"
            class="absolute z-10 w-full bottom-full mb-2 bg-gray-800 border border-gray-600 rounded-lg shadow-xl max-h-96 overflow-hidden"
          >
            <!-- 搜尋框 -->
            <div class="p-2 border-b border-gray-600">
              <div class="relative">
                <MagnifyingGlassIcon
                  class="w-4 h-4 absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400"
                />
                <input
                  v-model="dropdownSearch"
                  type="text"
                  placeholder="搜尋 BOSS 名稱..."
                  class="w-full bg-gray-700 text-white pl-9 pr-3 py-2 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
                  @click.stop
                />
              </div>
            </div>

            <!-- BOSS 列表 -->
            <!-- 讓下拉選單每次只顯示約 3 筆（其餘可捲動） -->
            <div class="overflow-y-auto max-h-[216px]">
              <button
                v-for="boss in filteredBosses"
                :key="boss.id"
                @click="selectBoss(boss.id)"
                class="w-full px-4 py-3 flex items-center gap-3 hover:bg-gray-700 transition text-left"
                :class="{ 'bg-gray-700': addRecordForm.boss === boss.id }"
              >
                <div class="w-12 h-12 flex-shrink-0">
                  <img
                    :src="getBossImage(boss.id)"
                    :alt="boss.name"
                    class="w-full h-full object-contain"
                  />
                </div>
                <div class="flex-1 min-w-0">
                  <div class="text-white font-medium">{{ boss.name }}</div>
                  <div class="text-xs text-gray-400">
                    {{ boss.description }}
                  </div>
                  <div
                    class="text-xs text-gray-500 truncate"
                    v-if="boss.location.length > 0"
                  >
                    📍 {{ boss.location[0] }}
                  </div>
                </div>
                <span
                  role="button"
                  :title="isBossFavorite(boss.id) ? '取消最愛' : '加入最愛'"
                  @click.stop="toggleFavoriteBoss(boss.id)"
                  class="flex-shrink-0"
                >
                  <StarSolidIcon
                    class="w-5 h-5 transition"
                    :class="
                      isBossFavorite(boss.id)
                        ? 'text-yellow-300 drop-shadow-[0_0_6px_rgba(250,204,21,0.8)]'
                        : 'text-gray-500 opacity-80 hover:opacity-100'
                    "
                  />
                </span>
              </button>
              <div
                v-if="filteredBosses.length === 0"
                class="px-4 py-8 text-center text-gray-500"
              >
                找不到符合的 BOSS
              </div>
            </div>
          </div>
        </div>

        <button
          class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md transition flex items-center gap-2"
          @click="openAddRecordModal"
        >
          <PlusIcon class="w-5 h-5" />
        </button>
      </div>

      <!-- 新增一筆紀錄的彈窗 -->
      <div
        class="w-full"
        v-if="isAddRecordModalOpen"
        @click.self="closeAddRecordModal"
      >
        <div class="bg-gray-800 p-6 rounded-lg w-full">
          <h3 class="text-xl font-bold text-white mb-4">
            新增擊殺記錄 - {{ selectedBossConfig?.name }}
          </h3>

          <div class="flex flex-col space-y-4">
            <div>
              <label for="channel" class="text-white block mb-2">
                頻道 <span class="text-red-600">*</span>
              </label>
              <input
                ref="channelInputRef"
                type="number"
                min="1"
                max="9999"
                id="channel"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 1111"
                v-model="addRecordForm.channel"
                @keyup.enter="addRecord"
              />
            </div>

            <div>
              <label class="text-white block mb-2">
                擊殺時間 <span class="text-red-600">*</span>
                <span class="text-gray-400 text-sm font-normal">（24 小時制）</span>
              </label>
              <div class="flex gap-2 items-center">
                <input
                  type="number"
                  min="0"
                  max="23"
                  class="bg-gray-700 text-white px-4 py-2 rounded-md flex-1 text-center"
                  placeholder="時"
                  :value="killTimeHour"
                  @input="onKillTimeHourInput"
                />
                <span class="text-gray-400">:</span>
                <input
                  type="number"
                  min="0"
                  max="59"
                  class="bg-gray-700 text-white px-4 py-2 rounded-md flex-1 text-center"
                  placeholder="分"
                  :value="killTimeMinute"
                  @input="onKillTimeMinuteInput"
                />
              </div>
            </div>

            <!-- <div v-if="selectedBossConfig?.needLocation">
              <label for="killLocation" class="text-white block mb-2">
                擊殺地點
              </label>
              <input
                type="text"
                id="killLocation"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 北方洞穴"
                v-model="addRecordForm.killLocation"
              />
            </div> -->

            <!-- <div>
              <label for="loot" class="text-white block mb-2">收穫的寶物</label>
              <input
                type="text"
                id="loot"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 紫裝、材料"
                v-model="addRecordForm.loot"
              />
            </div> -->

            <div class="flex gap-2 mt-4">
              <button
                class="flex-1 bg-gray-600 hover:bg-gray-700 text-white px-4 py-2 rounded-md transition flex items-center justify-center gap-2"
                @click="closeAddRecordModal"
              >
                取消
              </button>
              <button
                :disabled="
                  !(
                    addRecordForm.channel &&
                    addRecordForm.killTime &&
                    addRecordForm.boss
                  )
                "
                class="flex-1 bg-blue-600 hover:bg-blue-700 disabled:bg-gray-500 disabled:cursor-not-allowed disabled:opacity-50 text-white px-4 py-2 rounded-md transition flex items-center justify-center gap-2"
                @click="addRecord"
              >
                新增
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 匯入分享資料彈窗 -->
    <div
      v-if="isImportModalOpen"
      class="fixed inset-0 z-30 flex items-center justify-center bg-black/60 p-4"
      @click.self="closeImportModal"
    >
      <div class="bg-gray-800 rounded-xl p-6 w-full max-w-lg shadow-xl">
        <h3 class="text-lg font-bold text-white mb-2">匯入分享資料</h3>
        <p class="text-gray-400 text-sm mb-3">
          貼上他人分享的資料，將與您目前的記錄合併；同一 BOSS＋頻道會保留較新的擊殺時間。
        </p>
        <textarea
          v-model="importPasteText"
          class="w-full h-40 bg-gray-700 text-white rounded-lg px-3 py-2 text-sm font-mono resize-none focus:ring-2 focus:ring-blue-500 focus:outline-none"
          placeholder="請貼上他人分享的內容（整段複製）"
        />
        <div class="flex gap-2 mt-4">
          <button
            class="flex-1 bg-gray-600 hover:bg-gray-700 text-white px-4 py-2 rounded-md transition"
            @click="closeImportModal"
          >
            取消
          </button>
          <button
            class="flex-1 bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md transition"
            @click="doImport"
          >
            匯入並合併
          </button>
        </div>
      </div>
    </div>

    <!-- 匯入 LZURLSTR 彈窗 -->
    <div
      v-if="isLzImportModalOpen"
      class="fixed inset-0 z-30 flex items-center justify-center bg-black/60 p-4"
      @click.self="closeLzImportModal"
    >
      <div class="bg-gray-800 rounded-xl p-6 w-full max-w-lg shadow-xl">
        <h3 class="text-lg font-bold text-white mb-2">匯入 LZURLSTR</h3>
        <p class="text-gray-400 text-sm mb-3">
          使用
          <code class="text-gray-300">LZString.decompressFromEncodedURIComponent</code>
          解壓縮後解析 JSON，並轉成本工具擊殺紀錄格式，再與現有資料合併（同 BOSS＋頻道保留較新擊殺時間）。
          若貼上的是
          <code class="text-gray-300">https://artale.app/boss?data=...</code>
          完整連結，會自動取出
          <code class="text-gray-300">data</code>
          參數再解壓；純 LZ 字串亦可。
        </p>
        <textarea
          v-model="lzImportPasteText"
          class="w-full h-40 bg-gray-700 text-white rounded-lg px-3 py-2 text-sm font-mono resize-none focus:ring-2 focus:ring-blue-500 focus:outline-none"
          placeholder="可貼整段 https://artale.app/boss?data=... 或僅貼 data 後的 LZ 字串"
        />
        <div class="flex gap-2 mt-4">
          <button
            class="flex-1 bg-gray-600 hover:bg-gray-700 text-white px-4 py-2 rounded-md transition"
            @click="closeLzImportModal"
          >
            取消
          </button>
          <button
            class="flex-1 bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md transition"
            @click="doLzImport"
          >
            解壓並匯入合併
          </button>
        </div>
      </div>
    </div>

    <!-- 刪除確認彈窗 -->
    <div
      v-if="isDeleteConfirmModalOpen"
      class="fixed inset-0 z-40 flex items-end sm:items-center justify-center bg-black/60 p-3 sm:p-4"
      @click.self="closeDeleteConfirmModal"
    >
      <div
        class="w-full max-w-3xl rounded-2xl border border-[#373a40] bg-[#1f2128] p-5 sm:p-6 shadow-2xl"
      >
        <h3 class="text-3xl font-bold text-white leading-tight break-words">
          清除 {{ pendingDeleteRecord?.bossName }} 頻道 {{ pendingDeleteRecord?.channel }} 的記錄？
        </h3>
        <p class="mt-4 text-[#a1a1aa] text-xl leading-relaxed break-words">
          此操作無法復原，該 Boss 在此頻道的記錄都將被刪除。
        </p>

        <div class="mt-8 flex justify-end gap-3">
          <button
            type="button"
            class="px-8 py-3 rounded-2xl border border-[#4b4f5c] text-[#e5e7eb] bg-transparent hover:bg-white/5 transition"
            @click="closeDeleteConfirmModal"
          >
            取消
          </button>
          <button
            type="button"
            class="px-8 py-3 rounded-2xl bg-[#ff2d3d] hover:bg-[#ff4654] text-white font-semibold transition"
            @click="confirmDeleteRecord"
          >
            確定
          </button>
        </div>
      </div>
    </div>

    <!-- 記錄列表（版面參考：深底 #1a1b1e、卡片 #25262b、邊框 #373a40） -->
    <div
      class="rounded-xl p-4 bg-[#1a1b1e] border border-[#373a40] min-w-0 max-w-full"
    >
      <!-- 空狀態 -->
      <div
        v-if="killRecords.length === 0"
        class="rounded-[10px] border border-[#373a40] bg-[#25262b] p-8 text-center"
      >
        <p class="text-gray-400 text-lg">尚無記錄</p>
        <p class="text-gray-500 text-sm mt-2">選擇一個 BOSS 開始記錄吧！</p>
      </div>

      <!-- 分組顯示 -->
      <div v-else class="space-y-4 min-w-0 max-w-full">
        <!-- 已重生區塊（預設隱藏，可展開） -->
        <template v-if="groupedRecords.respawned.length > 0">
          <div
            v-if="!showRespawnedSection"
            class="border border-dashed border-[#373a40] rounded-[10px] px-4 py-3 flex flex-wrap items-center justify-between gap-2 bg-[#25262b]"
          >
            <div class="flex items-center gap-2 min-w-0">
              <div class="w-3 h-3 rounded-full bg-sky-500 shrink-0"></div>
              <span class="text-sky-400 font-semibold">已重生</span>
              <span class="text-gray-500 text-sm"
                >共 {{ groupedRecords.respawned.length }} 筆（預設隱藏）</span>
            </div>
            <button
              type="button"
              class="shrink-0 px-3 py-1.5 rounded-md bg-sky-700/90 hover:bg-sky-600 text-white text-sm transition"
              @click="showRespawnedSection = true"
            >
              展開
            </button>
          </div>
          <div
            v-else
            class="rounded-[10px] border border-[#373a40] bg-[#141517] p-4 min-w-0 max-w-full"
          >
          <div
            class="flex flex-wrap items-center justify-between gap-2 mb-3"
          >
            <div class="flex items-center gap-2">
              <div class="w-3 h-3 rounded-full bg-sky-500"></div>
              <h3 class="text-lg font-bold text-sky-400">
                已重生 ({{ groupedRecords.respawned.length }})
              </h3>
            </div>
            <div class="flex flex-wrap items-center justify-end gap-1.5 text-sm">
              <button
                type="button"
                class="px-2.5 py-1 rounded-md bg-gray-700 text-gray-300 hover:bg-gray-600 transition mr-1"
                @click="showRespawnedSection = false"
              >
                收合
              </button>
              <span class="text-gray-500 shrink-0">排序</span>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.respawned === 'time'
                    ? 'bg-sky-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.respawned = 'time'"
              >
                時間
              </button>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.respawned === 'channel'
                    ? 'bg-sky-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.respawned = 'channel'"
              >
                頻道
              </button>
            </div>
          </div>
          <div
            class="grid grid-cols-1 md:grid-cols-2 2xl:grid-cols-3 gap-2 min-w-0"
          >
            <div
              v-for="(record, index) in sortedGroupedRecords.respawned"
              :key="record.id"
              class="rounded-lg border px-2 py-2 transition border-[#373a40] bg-[#25262b] hover:border-[#4a4d55] min-w-0 max-w-full"
              :class="
                showChannelProximityRowBg('respawned', index)
                  ? 'bg-[rgba(57,255,20,0.12)] shadow-[0_0_18px_rgba(57,255,20,0.35)] ring-1 ring-[#39ff14]/45'
                  : ''
              "
            >
              <div class="flex gap-2 min-w-0 max-w-full items-start">
                <div
                  class="flex w-9 shrink-0 flex-col items-center gap-1"
                >
                  <div
                    class="flex h-9 w-9 items-center justify-center overflow-hidden rounded-md bg-[#1a1b1e]/40"
                  >
                    <img
                      :src="getBossImage(record.bossId)"
                      :alt="record.bossName"
                      class="h-full w-full max-h-full max-w-full object-contain"
                    />
                  </div>
                  <button
                    type="button"
                    @click="requestDeleteRecord(record)"
                    class="record-card-icon-btn p-1 text-[#909296] hover:text-white hover:bg-[#2c2e33] border border-[#373a40] hover:border-[#4a4d55]"
                    title="刪除記錄"
                  >
                    <TrashIcon class="h-3.5 w-3.5 shrink-0" />
                  </button>
                </div>

                <div
                  class="flex min-w-0 flex-1 flex-col gap-1"
                >
                  <div class="min-w-0 max-w-full space-y-0.5">
                    <div
                      class="flex min-w-0 flex-wrap items-baseline gap-x-1.5 gap-y-0"
                    >
                      <h3
                        class="min-w-0 max-w-full break-words text-xs font-bold leading-snug text-white [overflow-wrap:anywhere] sm:text-sm"
                        :title="record.bossName"
                      >
                        {{ record.bossName }}
                      </h3>
                      <span
                        class="record-card-channel-wrap shrink-0 text-[11px] sm:text-xs"
                      >
                        <span class="text-[#909296]">頻道</span>
                        <span class="record-card-channel-num">{{
                          record.channel
                        }}</span>
                      </span>
                    </div>
                    <div
                      class="flex min-w-0 max-w-full items-start gap-1 text-[11px] text-[#909296] sm:text-xs"
                    >
                      <ClockIcon
                        class="mt-px h-3 w-3 shrink-0 text-[#909296] sm:h-3.5 sm:w-3.5"
                      />
                      <span
                        class="min-w-0 flex-1 break-words leading-snug"
                        :title="formatRespawnTimeRange(record)"
                      >
                        {{ formatRespawnTimeRange(record) }}
                      </span>
                    </div>
                  </div>

                  <div
                    v-if="record.checkPoints && record.checkPoints.length > 0"
                    class="min-w-0 max-w-full text-[11px] text-[#909296]"
                  >
                    <div
                      class="flex min-w-0 max-w-full flex-wrap items-center gap-1"
                    >
                      <EyeIcon
                        class="h-3 w-3 shrink-0 text-[#909296]"
                        :title="
                          formatCheckPointTime(getLatestCheckPoint(record))
                        "
                      />
                      <span
                        v-for="(checkPoint, index) in [
                          ...record.checkPoints,
                        ].reverse()"
                        :key="index"
                        class="group relative max-w-full shrink-0 cursor-pointer rounded border border-[#373a40] bg-[#1a1b1e] px-1 py-px text-[10px] text-[#a6a7ab] transition-colors hover:border-[#4a4d55]"
                        @click.stop="deleteCheckPoint(record, index)"
                        title="點擊刪除此踩點記錄"
                      >
                        {{
                          new Date(checkPoint.time).toLocaleTimeString(
                            "zh-TW",
                            {
                              hour: "2-digit",
                              minute: "2-digit",
                              hour12: false,
                            }
                          )
                        }}
                        <XMarkIcon
                          class="absolute left-1/2 top-1/2 h-4 w-4 -translate-x-1/2 -translate-y-1/2 rounded-full bg-red-600 p-0.5 font-bold text-white opacity-0 transition-opacity group-hover:opacity-100"
                        />
                      </span>
                    </div>
                  </div>

                  <div class="record-card-actions">
                    <div
                      class="inline-flex max-w-full shrink-0 items-center gap-1 rounded border border-sky-600/50 bg-sky-950/60 px-1.5 py-0.5 leading-none"
                      title="已重生"
                    >
                      <ClockIcon class="h-3 w-3 shrink-0 text-sky-400" />
                      <span
                        class="break-words text-[11px] font-semibold tabular-nums text-sky-300"
                      >
                        {{ formatDisplayTime(record.respawnTimeMax) }}
                      </span>
                    </div>

                    <button
                      type="button"
                      @click="reAddRecord(record)"
                      class="record-card-action-icon record-card-action-icon--timer record-card-action-icon--sm"
                      title="重新計時"
                    >
                      <CheckBadgeIcon class="h-3.5 w-3.5 shrink-0" />
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </div>
          </div>
        </template>

        <!-- 可能已重生區塊 -->
        <div
          v-if="groupedRecords.respawning.length > 0"
          class="rounded-[10px] border border-[#373a40] bg-[#141517] p-4 min-w-0 max-w-full"
        >
          <div
            class="flex flex-wrap items-center justify-between gap-2 mb-3"
          >
            <div class="flex items-center gap-2 min-w-0" title="可能已重生">
              <BoltIcon
                class="w-5 h-5 text-green-400 animate-pulse shrink-0"
                aria-hidden="true"
              />
              <span class="text-lg font-bold text-green-400 tabular-nums">
                {{ groupedRecords.respawning.length }}
              </span>
            </div>
            <div class="flex items-center gap-1.5 text-sm">
              <span class="text-gray-500 shrink-0">排序</span>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.respawning === 'time'
                    ? 'bg-green-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.respawning = 'time'"
              >
                時間
              </button>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.respawning === 'channel'
                    ? 'bg-green-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.respawning = 'channel'"
              >
                頻道
              </button>
            </div>
          </div>
          <div
            class="grid grid-cols-1 md:grid-cols-2 2xl:grid-cols-3 gap-2 min-w-0"
          >
            <div
              v-for="(record, index) in sortedGroupedRecords.respawning"
              :key="record.id"
              class="rounded-lg border px-2 py-2 transition border-[#373a40] bg-[#25262b] hover:border-[#4a4d55] min-w-0 max-w-full"
              :class="
                showChannelProximityRowBg('respawning', index)
                  ? 'bg-[rgba(57,255,20,0.12)] shadow-[0_0_18px_rgba(57,255,20,0.35)] ring-1 ring-[#39ff14]/45'
                  : ''
              "
            >
              <div class="flex gap-2 min-w-0 max-w-full items-start">
                <div
                  class="flex w-9 shrink-0 flex-col items-center gap-1"
                >
                  <div
                    class="flex h-9 w-9 items-center justify-center overflow-hidden rounded-md bg-[#1a1b1e]/40"
                  >
                    <img
                      :src="getBossImage(record.bossId)"
                      :alt="record.bossName"
                      class="h-full w-full max-h-full max-w-full object-contain"
                    />
                  </div>
                  <button
                    type="button"
                    @click="requestDeleteRecord(record)"
                    class="record-card-icon-btn p-1 text-[#909296] hover:text-white hover:bg-[#2c2e33] border border-[#373a40] hover:border-[#4a4d55]"
                    title="刪除記錄"
                  >
                    <TrashIcon class="h-3.5 w-3.5 shrink-0" />
                  </button>
                </div>

                <div
                  class="flex min-w-0 flex-1 flex-col gap-1"
                >
                  <div class="min-w-0 max-w-full space-y-0.5">
                    <div
                      class="flex min-w-0 flex-wrap items-baseline gap-x-1.5 gap-y-0"
                    >
                      <h3
                        class="min-w-0 max-w-full break-words text-xs font-bold leading-snug text-white [overflow-wrap:anywhere] sm:text-sm"
                        :title="record.bossName"
                      >
                        {{ record.bossName }}
                      </h3>
                      <span
                        class="record-card-channel-wrap shrink-0 text-[11px] sm:text-xs"
                      >
                        <span class="text-[#909296]">頻道</span>
                        <span class="record-card-channel-num">{{
                          record.channel
                        }}</span>
                      </span>
                    </div>
                    <div
                      class="flex min-w-0 max-w-full items-start gap-1 text-[11px] text-[#909296] sm:text-xs"
                    >
                      <ClockIcon
                        class="mt-px h-3 w-3 shrink-0 text-[#909296] sm:h-3.5 sm:w-3.5"
                      />
                      <span
                        class="min-w-0 flex-1 break-words leading-snug"
                        :title="formatRespawnTimeRange(record)"
                      >
                        {{ formatRespawnTimeRange(record) }}
                      </span>
                    </div>
                  </div>

                  <div class="min-w-0 max-w-full">
                    <div
                      class="flex w-full min-w-0 max-w-full items-center gap-1"
                    >
                      <div
                        class="h-1.5 min-w-0 flex-1 overflow-hidden rounded-full border border-[#373a40] bg-[#1a1b1e]"
                      >
                        <div
                          class="h-full min-w-0 bg-gradient-to-r from-yellow-500 via-green-500 to-green-400 shadow-lg transition-all duration-1000 ease-out"
                          :style="{
                            width: getRecordCountdown(record).percentage + '%',
                          }"
                        ></div>
                      </div>
                      <span
                        class="shrink-0 tabular-nums text-[10px] font-semibold text-green-400"
                      >
                        {{ getRecordCountdown(record).percentage }}%
                      </span>
                    </div>
                  </div>

                  <div
                    v-if="record.checkPoints && record.checkPoints.length > 0"
                    class="min-w-0 max-w-full text-[11px] text-[#909296]"
                  >
                    <div
                      class="flex min-w-0 max-w-full flex-wrap items-center gap-1"
                    >
                      <EyeIcon
                        class="h-3 w-3 shrink-0 text-[#909296]"
                        :title="
                          formatCheckPointTime(getLatestCheckPoint(record))
                        "
                      />
                      <span
                        v-for="(checkPoint, index) in [
                          ...record.checkPoints,
                        ].reverse()"
                        :key="index"
                        class="group relative max-w-full shrink-0 cursor-pointer rounded border border-[#373a40] bg-[#1a1b1e] px-1 py-px text-[10px] text-[#a6a7ab] transition-colors hover:border-[#4a4d55]"
                        @click.stop="deleteCheckPoint(record, index)"
                        title="點擊刪除此踩點記錄"
                      >
                        {{
                          new Date(checkPoint.time).toLocaleTimeString(
                            "zh-TW",
                            {
                              hour: "2-digit",
                              minute: "2-digit",
                              hour12: false,
                            }
                          )
                        }}
                        <XMarkIcon
                          class="absolute left-1/2 top-1/2 h-4 w-4 -translate-x-1/2 -translate-y-1/2 rounded-full bg-red-600 p-0.5 font-bold text-white opacity-0 transition-opacity group-hover:opacity-100"
                        />
                      </span>
                    </div>
                  </div>

                  <div class="record-card-actions">
                    <div
                      class="inline-flex max-w-full shrink-0 animate-pulse items-center gap-1 rounded border border-green-600/50 bg-green-950/60 px-1.5 py-0.5 leading-none"
                      title="可能已重生"
                    >
                      <BoltIcon class="h-3.5 w-3.5 shrink-0 text-green-400" />
                      <span
                        class="min-w-0 max-w-full break-all font-mono text-[11px] font-bold tabular-nums text-green-300"
                      >
                        {{ getRecordCountdown(record).timeText }}
                      </span>
                    </div>

                    <button
                      type="button"
                      @click="addCheckPoint(record)"
                      class="record-card-action-icon record-card-action-icon--checkpoint record-card-action-icon--sm"
                      title="踩點記錄（已查看但沒有 BOSS）"
                    >
                      <EyeIcon class="h-3.5 w-3.5 shrink-0" />
                    </button>

                    <button
                      type="button"
                      @click="reAddRecord(record)"
                      class="record-card-action-icon record-card-action-icon--timer record-card-action-icon--sm"
                      title="重新計時"
                    >
                      <CheckBadgeIcon class="h-3.5 w-3.5 shrink-0" />
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 即將重生區塊 -->
        <div
          v-if="groupedRecords.waiting.length > 0"
          class="rounded-[10px] border border-[#373a40] bg-[#141517] p-4 min-w-0 max-w-full"
        >
          <div
            class="flex flex-wrap items-center justify-between gap-2 mb-3"
          >
            <div class="flex items-center gap-2 min-w-0" title="即將重生">
              <ClockIcon class="w-5 h-5 text-yellow-400 shrink-0" />
              <span class="text-lg font-bold text-yellow-400 tabular-nums">
                {{ groupedRecords.waiting.length }}
              </span>
            </div>
            <div class="flex items-center gap-1.5 text-sm">
              <span class="text-gray-500 shrink-0">排序</span>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.waiting === 'time'
                    ? 'bg-yellow-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.waiting = 'time'"
              >
                時間
              </button>
              <button
                type="button"
                class="px-2.5 py-1 rounded-md transition"
                :class="
                  groupSortMode.waiting === 'channel'
                    ? 'bg-yellow-600 text-white'
                    : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                "
                @click="groupSortMode.waiting = 'channel'"
              >
                頻道
              </button>
            </div>
          </div>
          <div
            class="grid grid-cols-1 md:grid-cols-2 2xl:grid-cols-3 gap-2 min-w-0"
          >
            <div
              v-for="(record, index) in sortedGroupedRecords.waiting"
              :key="record.id"
              class="rounded-lg border px-2 py-2 transition border-[#373a40] bg-[#25262b] hover:border-[#4a4d55] min-w-0 max-w-full"
              :class="
                showChannelProximityRowBg('waiting', index)
                  ? 'bg-[rgba(57,255,20,0.12)] shadow-[0_0_18px_rgba(57,255,20,0.35)] ring-1 ring-[#39ff14]/45'
                  : ''
              "
            >
              <div class="flex gap-2 min-w-0 max-w-full items-start">
                <div
                  class="flex w-9 shrink-0 flex-col items-center gap-1"
                >
                  <div
                    class="flex h-9 w-9 items-center justify-center overflow-hidden rounded-md bg-[#1a1b1e]/40"
                  >
                    <img
                      :src="getBossImage(record.bossId)"
                      :alt="record.bossName"
                      class="h-full w-full max-h-full max-w-full object-contain"
                    />
                  </div>
                  <button
                    type="button"
                    @click="requestDeleteRecord(record)"
                    class="record-card-icon-btn p-1 text-[#909296] hover:text-white hover:bg-[#2c2e33] border border-[#373a40] hover:border-[#4a4d55]"
                    title="刪除記錄"
                  >
                    <TrashIcon class="h-3.5 w-3.5 shrink-0" />
                  </button>
                </div>

                <div
                  class="flex min-w-0 flex-1 flex-col gap-1"
                >
                  <div class="min-w-0 max-w-full space-y-0.5">
                    <div
                      class="flex min-w-0 flex-wrap items-baseline gap-x-1.5 gap-y-0"
                    >
                      <h3
                        class="min-w-0 max-w-full break-words text-xs font-bold leading-snug text-white [overflow-wrap:anywhere] sm:text-sm"
                        :title="record.bossName"
                      >
                        {{ record.bossName }}
                      </h3>
                      <span
                        class="record-card-channel-wrap shrink-0 text-[11px] sm:text-xs"
                      >
                        <span class="text-[#909296]">頻道</span>
                        <span class="record-card-channel-num">{{
                          record.channel
                        }}</span>
                      </span>
                    </div>
                    <div
                      class="flex min-w-0 max-w-full items-start gap-1 text-[11px] text-[#909296] sm:text-xs"
                    >
                      <ClockIcon
                        class="mt-px h-3 w-3 shrink-0 text-[#909296] sm:h-3.5 sm:w-3.5"
                      />
                      <span
                        class="min-w-0 flex-1 break-words leading-snug"
                        :title="formatRespawnTimeRange(record)"
                      >
                        {{ formatRespawnTimeRange(record) }}
                      </span>
                    </div>
                  </div>

                  <div class="record-card-actions">
                    <div
                      class="inline-flex max-w-full shrink-0 items-center gap-1 rounded border border-yellow-600/50 bg-yellow-950/60 px-1.5 py-0.5 leading-none"
                      title="即將重生"
                    >
                      <ClockIcon class="h-3 w-3 shrink-0 text-yellow-400" />
                      <span
                        class="min-w-0 max-w-full break-all font-mono text-[11px] font-bold tabular-nums text-yellow-300"
                      >
                        {{ getRecordCountdown(record).timeText }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 頁尾 -->
    <div class="text-center text-white/50 text-sm min-h-[88px] py-4">
      <p>
        快捷鍵：主鍵盤 + 或 =、或數字鍵盤 + 打開新增彈窗；頻道輸入框開啟後可用 numpad+ / numpad- 或 [ ] 切換 Boss；按 Enter 確認新增
      </p>
      <p class="flex flex-wrap items-center justify-center gap-2">
        <span>頻道排序時，與上一筆或下一筆頻道號差距 ≤</span>
        <input
          v-model.number="channelProximityN"
          type="number"
          min="0"
          max="99999"
          class="w-16 bg-gray-800 border border-gray-600 rounded px-2 py-0.5 text-center text-white text-sm"
        />
        <span>時，該列以霓虹綠底標示</span>
      </p>
      <p>眼睛圖示用法：已查看但沒有 BOSS，點擊後會新增踩點記錄，踩點時間點擊可以刪除。</p>
      <br>
      <p>資料會自動儲存在瀏覽器本地</p>
    </div>
  </div>
</template>

<style scoped>
.record-card-channel-wrap {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
}

.record-card-channel-num {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 1.25rem;
  padding: 0.1rem 0.4rem;
  border-radius: 0.25rem;
  font-weight: 800;
  font-variant-numeric: tabular-nums;
  line-height: 1.2;
  color: #fde68a;
  background: rgba(251, 191, 36, 0.18);
  box-shadow: inset 0 0 0 1px rgba(251, 191, 36, 0.45);
}

.record-card-actions {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: flex-start;
  gap: 0.25rem;
  width: 100%;
  min-width: 0;
  max-width: 100%;
  box-sizing: border-box;
}

.record-card-action-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.375rem;
  border-radius: 0.375rem;
  border: 1px solid transparent;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
  flex: 0 0 auto;
  max-width: 100%;
  box-sizing: border-box;
  color: #fff;
  transition:
    background-color 0.15s ease,
    border-color 0.15s ease,
    box-shadow 0.15s ease;
}

.record-card-action-icon--checkpoint {
  background-color: #2563eb;
  border-color: #3b82f6;
}

.record-card-action-icon--checkpoint:hover {
  background-color: #3b82f6;
  border-color: #60a5fa;
  box-shadow: 0 2px 6px rgba(37, 99, 235, 0.35);
}

.record-card-action-icon--timer {
  background-color: #059669;
  border-color: #10b981;
}

.record-card-action-icon--timer:hover {
  background-color: #10b981;
  border-color: #34d399;
  box-shadow: 0 2px 6px rgba(5, 150, 105, 0.35);
}

.record-card-action-icon--sm {
  padding: 0.25rem;
  border-radius: 0.25rem;
  box-shadow: none;
}

.record-card-icon-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.375rem;
  border-radius: 0.375rem;
  flex: 0 0 auto;
  max-width: 100%;
  box-sizing: border-box;
  transition:
    color 0.15s ease,
    background-color 0.15s ease,
    border-color 0.15s ease;
}

.pic-auto {
  width: 100%;
  height: 100%;
  vertical-align: middle;
  object-fit: cover;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.8;
  }
}

.animate-pulse {
  animation: pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}
</style>
