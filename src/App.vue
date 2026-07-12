<script setup lang="ts">
import { ref, computed, watch, onMounted } from "vue";
import JSZip from "jszip";
import { openUrl as openBrowser } from "@tauri-apps/plugin-opener";
import { marked } from "marked";

// Define Interfaces
interface KPI {
  label: string;
  value: string;
  trend: string;
  icon: string;
  up: boolean;
}

interface Opportunity {
  type: "Striking Distance" | "Low CTR" | "Internal Link";
  query: string;
  page: string;
  impr: number;
  clicks: number;
  ctr: number; // percentage, e.g. 2.1
  gain: number;
  prio: "high" | "medium" | "low";
  pos: number;
  why: string;
  action: string;
  source: string;
}

interface CannibalPage {
  url: string;
  pos: string;
  clicks: number;
  tag: string;
  primary: boolean;
}

interface CannibalGroup {
  query: string;
  totalImpr: string;
  count: number;
  pages: CannibalPage[];
}

interface DecayRow {
  query: string;
  page: string;
  prev: number;
  curr: number;
  change: string;
  posDiff: string;
  up: boolean;
}

interface GscRow {
  query: string;
  page: string;
  clicks: number;
  impressions: number;
  ctr: number; // as ratio, e.g. 0.021
  position: number;
}

// Sidebar & Active Views
const activeView = ref("dashboard");
const hasData = ref(false);
const uploading = ref(false);
const search = ref("");
const filter = ref("all");
const selectedOpp = ref<Opportunity | null>(null);

// Sorting & Source Filtering State for Fırsat Kaşifi
const sortBy = ref<string>("prio");
const sortOrder = ref<"asc" | "desc">("desc");

const showSourceQueries = ref(true);
const showSourcePages = ref(true);
const showSourceJoint = ref(true);

function toggleSort(col: string) {
  if (sortBy.value === col) {
    sortOrder.value = sortOrder.value === "asc" ? "desc" : "asc";
  } else {
    sortBy.value = col;
    sortOrder.value = "desc";
  }
}

// Settings
const minImpr = ref(100);
const sdLow = ref(4);
const sdHigh = ref(20);
const gscConnected = ref(false);
const gscAccessToken = ref("");
const gscClientId = ref(localStorage.getItem("gsc_client_id") || "");
const gscSites = ref<string[]>([]);
const selectedGscSite = ref("");
const loadingGscData = ref(false);
const geminiOn = ref(false);
const apiKey = ref(localStorage.getItem("gemini_api_key") || "");
const model = ref("flash");

// Auto-Updater State
const currentVersion = "0.1.0";
const githubRepo = ref("mehmetakarim/QueryGSC");
const showUpdateModal = ref(false);
const latestReleaseData = ref<{ version: string; body: string; url: string; assets: any[] } | null>(null);

// File Metadata
const currentFileName = ref("");
const previousFileName = ref("");
const rowCount = ref(0);

interface DailyMetric {
  date: string;
  clicks: number;
  impressions: number;
  ctr: number;
  position: number;
}

// GSC Data Rows
const parsedCurrentQueries = ref<GscRow[]>([]);
const parsedCurrentPages = ref<GscRow[]>([]);
const parsedPreviousQueries = ref<GscRow[]>([]);
const parsedPreviousPages = ref<GscRow[]>([]);
const parsedDailyMetrics = ref<DailyMetric[]>([]);

const isDataJoint = computed(() => {
  const currentQueries = parsedCurrentQueries.value;
  const currentPages = parsedCurrentPages.value;
  return (currentQueries.length > 0 && currentQueries.some(r => r.query && r.page)) ||
         (currentPages.length > 0 && currentPages.some(r => r.query && r.page));
});

// Dynamic Collections
const calculatedOpps = ref<Opportunity[]>([]);
const calculatedCannibal = ref<CannibalGroup[]>([]);
const calculatedDecay = ref<DecayRow[]>([]);
const decayAvgDecline = ref("-0.0%");
const decayAvgGrowth = ref("+0.0%");
const kpis = ref<KPI[]>([]);

// Chat State
const prompt = ref("");
const chat = ref<Array<{ role: string; text: string }>>([
  { role: "ai", text: 'Merhaba! Ben QueryGSC Yapay Zeka SEO Asistanıyım. Google Search Console verilerinizi yükledikten veya API ile bağlandıktan sonra, verilerinizi otomatik olarak okuyabilir, en kritik SEO fırsatlarını tespit edebilir ve size özel eylem planları hazırlayabilirim. Fırsat Kaşifi üzerinden bir kelime seçip "Yapay Zeka Asistanı ile Raporla" diyerek veya buraya doğrudan sorunuzu yazarak başlayabilirsiniz. Nasıl yardımcı olabilirim?' }
]);

// Titles dictionary
const titles: Record<string, [string, string]> = {
  dashboard: ["Panel", "Verilerinizi yükleyin ve genel görünümü inceleyin"],
  explorer: ["Fırsat Kaşifi", "Önceliklendirilmiş GSC SEO fırsatlarınız"],
  cannibal: ["Yamyamlık Denetçisi", "Aynı sorgu için birbiriyle rekabet eden sayfalarınız"],
  decay: ["İçerik Eskimesi", "Dönemsel performans ve sıralama değişimleri"],
  ai: ["Yapay Zeka Asistanı", "Gemini destekli akıllı SEO raporları"],
  settings: ["Ayarlar", "Analiz parametreleri ve API entegrasyonları"],
};

const currentTitle = computed(() => titles[activeView.value]?.[0] || "");
const currentSub = computed(() => titles[activeView.value]?.[1] || "");

const userName = computed(() => {
  if (gscConnected.value && selectedGscSite.value) {
    const domain = selectedGscSite.value.replace("https://", "").replace("http://", "").split("/")[0];
    return domain || "Google Kullanıcısı";
  } else if (geminiOn.value) {
    return "Gemini SEO Analisti";
  } else {
    return "Yerel Kullanıcı";
  }
});

const userInitials = computed(() => {
  const name = userName.value;
  if (name.includes(".")) {
    const parts = name.split(".");
    return ((parts[0]?.[0] || "Y") + (parts[0]?.[1] || "K")).toUpperCase();
  }
  const parts = name.split(" ");
  const first = parts[0]?.[0] || "Y";
  const last = parts[1]?.[0] || "K";
  return (first + last).toUpperCase();
});

const userTier = computed(() => {
  return (gscConnected.value || geminiOn.value) ? "Premium Plan" : "Ücretsiz Sürüm";
});

// Navigation menu
const navItems = [
  { id: "dashboard", label: "Panel & Veri Girişi", icon: "grid" },
  { id: "explorer", label: "Fırsat Kaşifi", icon: "target" },
  { id: "cannibal", label: "Yamyamlık Denetçisi", icon: "split" },
  { id: "decay", label: "İçerik Eskimesi", icon: "trend" },
  { id: "ai", label: "Yapay Zeka Asistanı", icon: "spark", badge: "AI" },
  { id: "settings", label: "Ayarlar", icon: "gear" },
];

// Expected GSC CTR values by position
function expectedCtr(pos: number) {
  if (pos <= 1) return 0.30;
  if (pos <= 2) return 0.15;
  if (pos <= 3) return 0.10;
  if (pos <= 4) return 0.08;
  if (pos <= 5) return 0.06;
  if (pos <= 6) return 0.04;
  if (pos <= 7) return 0.03;
  if (pos <= 8) return 0.025;
  if (pos <= 9) return 0.02;
  if (pos <= 10) return 0.015;
  return 0.01;
}

// Process GSC CSV Data
function processGscData() {
  const currentQueries = parsedCurrentQueries.value;
  const currentPages = parsedCurrentPages.value;
  const previousQueries = parsedPreviousQueries.value;
  const previousPages = parsedPreviousPages.value;

  const hasCurrentQueries = currentQueries.length > 0;
  const hasCurrentPages = currentPages.length > 0;

  if (!hasCurrentQueries && !hasCurrentPages) {
    hasData.value = false;
    return;
  }

  // 1. Calculate KPIs (Use queries if available, otherwise pages to avoid double-counting)
  const kpiSource = hasCurrentQueries ? currentQueries : currentPages;
  const totalImpr = kpiSource.reduce((sum, r) => sum + r.impressions, 0);
  const totalClicks = kpiSource.reduce((sum, r) => sum + r.clicks, 0);
  const avgCtr = totalImpr > 0 ? (totalClicks / totalImpr) * 100 : 0;
  const avgPos = kpiSource.length > 0 ? kpiSource.reduce((sum, r) => sum + r.position, 0) / kpiSource.length : 0;

  let prevImpr = 0;
  let prevClicks = 0;
  let prevAvgCtr = 0;
  let prevAvgPos = 0;

  const prevSource = previousQueries.length > 0 ? previousQueries : previousPages;
  if (prevSource.length > 0) {
    prevImpr = prevSource.reduce((sum, r) => sum + r.impressions, 0);
    prevClicks = prevSource.reduce((sum, r) => sum + r.clicks, 0);
    prevAvgCtr = prevImpr > 0 ? (prevClicks / prevImpr) * 100 : 0;
    prevAvgPos = prevSource.reduce((sum, r) => sum + r.position, 0) / prevSource.length;
  }

  const formatKpiVal = (val: number, isPercent = false, isFloat = false) => {
    if (isPercent) return "%" + val.toFixed(2);
    if (isFloat) return val.toFixed(1);
    if (val >= 1000000) return (val / 1000000).toFixed(1) + "M";
    if (val >= 1000) return (val / 100).toFixed(1) + "K";
    return val.toString();
  };

  const getTrendLabel = (curr: number, prev: number) => {
    if (prev === 0) return "+0.0%";
    const diff = ((curr - prev) / prev) * 100;
    const sign = diff >= 0 ? "+" : "";
    return `${sign}${diff.toFixed(1)}%`;
  };

  kpis.value = [
    { 
      label: "Toplam Gösterim", 
      value: formatKpiVal(totalImpr), 
      trend: getTrendLabel(totalImpr, prevImpr), 
      icon: "eye", 
      up: totalImpr >= prevImpr 
    },
    { 
      label: "Toplam Tıklama", 
      value: formatKpiVal(totalClicks), 
      trend: getTrendLabel(totalClicks, prevClicks), 
      icon: "click", 
      up: totalClicks >= prevClicks 
    },
    { 
      label: "Ortalama CTR", 
      value: formatKpiVal(avgCtr, true), 
      trend: getTrendLabel(avgCtr, prevAvgCtr), 
      icon: "percent", 
      up: avgCtr >= prevAvgCtr 
    },
    { 
      label: "Ortalama Pozisyon", 
      value: formatKpiVal(avgPos, false, true), 
      trend: getTrendLabel(avgPos, prevAvgPos), 
      icon: "rank", 
      up: avgPos <= prevAvgPos
    },
  ];

  // 2. Opportunities Mining
  const opps: Opportunity[] = [];

  const isQueriesJoint = hasCurrentQueries && currentQueries.some(r => r.query && r.page);
  const isPagesJoint = hasCurrentPages && currentPages.some(r => r.query && r.page);
  const isJoint = isQueriesJoint || isPagesJoint;

  if (isJoint) {
    const jointRows = isQueriesJoint ? currentQueries : currentPages;
    jointRows.forEach(row => {
      if (row.impressions < minImpr.value) return;

      const exp = expectedCtr(row.position);
      const isSd = row.position >= sdLow.value && row.position <= sdHigh.value;
      const isLowCtr = row.position < 10 && row.ctr < exp * 0.6;

      if (isSd) {
        if (isLowCtr) {
          // Combined Striking Distance + Low CTR
          opps.push({
            type: "Striking Distance",
            query: row.query,
            page: row.page,
            impr: row.impressions,
            clicks: row.clicks,
            ctr: row.ctr * 100,
            gain: Math.round(row.impressions * (exp - row.ctr)),
            prio: "high",
            pos: row.position,
            why: `Bu sorgu hem ${row.position.toFixed(1)}. pozisyonda (çıkış potansiyeli yüksek) yer alıyor, hem de arama tıklama oranı (%${(row.ctr * 100).toFixed(1)}) bu sıra için beklenen %${(exp * 100).toFixed(1)} oranının çok altında. Hem sıralama hem de tıklama optimizasyonu gerekiyor.`,
            action: `1. Başlık ve meta açıklamasını arama niyetine göre daha cazip hale getirip SERP görünümünü iyileştirin (Low CTR çözümü). 2. Sayfa içeriğine "${row.query}" alt başlıklarını ekleyin ve güçlü sayfalarınızdan iç link verin (Striking Distance çözümü).`,
            source: "Birleşik Veri"
          });
        } else {
          // Pure Striking Distance
          const ctrGap = Math.max(0, exp - row.ctr);
          const isHigh = row.position <= 10 && row.impressions > 1000;
          opps.push({
            type: "Striking Distance",
            query: row.query,
            page: row.page,
            impr: row.impressions,
            clicks: row.clicks,
            ctr: row.ctr * 100,
            gain: ctrGap > 0 ? Math.round(row.impressions * ctrGap) : Math.round(row.impressions * 0.02),
            prio: isHigh ? "high" : (row.position <= 15 ? "medium" : "low"),
            pos: row.position,
            why: `Bu sorgu ${row.position.toFixed(1)}. pozisyonda yer alıyor. Birkaç iç link ve içerik güncellemesi ile ilk 5 sıraya yükselebilir. Gösterim değeri (${row.impressions}) yüksek olduğu için küçük bir pozisyon artışı doğrudan trafik getirir.`,
            action: `Hedef sayfada "${row.query}" sorgusunu içeren H2/H3 başlıkları açın ve kapsamlı açıklamalar ekleyin. Sitenizin diğer sayfalarından bu sayfaya "${row.query}" çapa metniyle iç link verin.`,
            source: "Birleşik Veri"
          });
        }
      } else if (isLowCtr) {
        opps.push({
          type: "Low CTR",
          query: row.query,
          page: row.page,
          impr: row.impressions,
          clicks: row.clicks,
          ctr: row.ctr * 100,
          gain: Math.round(row.impressions * (exp - row.ctr)),
          prio: row.impressions > 1000 ? "high" : "medium",
          pos: row.position,
          why: `Sayfanız ${row.position.toFixed(1)}. sırada yer almasına rağmen ortalama tıklama oranı (%${(row.ctr * 100).toFixed(1)}), bu pozisyon için beklenen oran olan %${(exp * 100).toFixed(1)} değerinin çok altında kalıyor. Arama görünümünüz (SERP) kullanıcı çekemiyor.`,
          action: `Sayfanın başlık (title) etiketine dikkat çekici kelimeler veya sayılar ekleyin. Meta açıklamasını arama niyetine göre zenginleştirin ve harekete geçirici mesaj (Call to Action) ekleyin. FAQ Schema markup entegre edin.`,
          source: "Birleşik Veri"
        });
      } else if (row.position >= sdLow.value && row.position <= sdHigh.value && row.impressions > 2000) {
        opps.push({
          type: "Internal Link",
          query: row.query,
          page: row.page,
          impr: row.impressions,
          clicks: row.clicks,
          ctr: row.ctr * 100,
          gain: Math.round(row.impressions * 0.015),
          prio: "high",
          pos: row.position,
          why: `Bu sayfa ${row.position.toFixed(1)}. pozisyonda ve potansiyeli yüksek. Güçlü otoriteli sayfalarınızdan yeterince iç link desteği almıyor olması sıralama yükselişini engelliyor olabilir.`,
          action: `Sitenizin en çok dış bağlantı alan ve trafik çeken sayfalarından, bu sayfaya "${row.query}" kelimesi üzerinden alakalı kontekstüel link verin.`,
          source: "Birleşik Veri"
        });
      }
    });
  } else {
    // Separate files
    if (hasCurrentQueries) {
      currentQueries.forEach(row => {
        if (row.impressions < minImpr.value) return;

        const exp = expectedCtr(row.position);
        const isSd = row.position >= sdLow.value && row.position <= sdHigh.value;
        const isLowCtr = row.position < 10 && row.ctr < exp * 0.6;

        if (isSd) {
          if (isLowCtr) {
            // Combined Striking Distance + Low CTR
            opps.push({
              type: "Striking Distance",
              query: row.query,
              page: "(Sorgu Temelli / URL Eksik)",
              impr: row.impressions,
              clicks: row.clicks,
              ctr: row.ctr * 100,
              gain: Math.round(row.impressions * (exp - row.ctr)),
              prio: "high",
              pos: row.position,
              why: `Bu sorgu hem ${row.position.toFixed(1)}. pozisyonda (çıkış potansiyeli yüksek) yer alıyor, hem de tıklama oranı (%${(row.ctr * 100).toFixed(1)}) beklenen %${(exp * 100).toFixed(1)} oranının altında. Çift taraflı optimizasyon gerekiyor.`,
              action: `1. SERP başlığını ve meta açıklamasını dikkat çekici hale getirin (Low CTR çözümü). 2. Sitenizde bu arama sorgusunu en iyi karşılayan sayfayı belirleyip diğer sayfalardan iç bağlantı verin (Striking Distance çözümü).`,
              source: "Sorgular.csv"
            });
          } else {
            // Pure Striking Distance
            const ctrGap = Math.max(0, exp - row.ctr);
            const isHigh = row.position <= 10 && row.impressions > 1000;
            opps.push({
              type: "Striking Distance",
              query: row.query,
              page: "(Sorgu Temelli / URL Eksik)",
              impr: row.impressions,
              clicks: row.clicks,
              ctr: row.ctr * 100,
              gain: ctrGap > 0 ? Math.round(row.impressions * ctrGap) : Math.round(row.impressions * 0.02),
              prio: isHigh ? "high" : (row.position <= 15 ? "medium" : "low"),
              pos: row.position,
              why: `Bu sorgu ${row.position.toFixed(1)}. pozisyonda (2. sayfanın başında veya 1. sayfanın sonunda) yer alıyor. Hangi sayfanın sıralandığını belirlemek için bu kelimeyi web sitenizde arayın veya GSC API bağlayın.`,
              action: `Sitenizde bu arama sorgusunu en iyi karşılayan sayfayı belirleyin. O sayfada "${row.query}" kelimesini içeren başlıklar açın ve diğer sayfalarınızdan iç bağlantı verin.`,
              source: "Sorgular.csv"
            });
          }
        } else if (isLowCtr) {
          opps.push({
            type: "Low CTR",
            query: row.query,
            page: "(Sorgu Temelli / URL Eksik)",
            impr: row.impressions,
            clicks: row.clicks,
            ctr: row.ctr * 100,
            gain: Math.round(row.impressions * (exp - row.ctr)),
            prio: row.impressions > 1000 ? "high" : "medium",
            pos: row.position,
            why: `"${row.query}" sorgusu ${row.position.toFixed(1)}. sırada yer almasına rağmen tıklama oranı (%${(row.ctr * 100).toFixed(1)}), bu pozisyon için beklenen oran olan %${(exp * 100).toFixed(1)} değerinin çok altında kalıyor.`,
            action: `Bu sorgu için Google arama sonuçlarındaki (SERP) başlığınızı ve meta açıklamanızı daha dikkat çekici hale getirin.`,
            source: "Sorgular.csv"
          });
        }
      });
    }

    if (hasCurrentPages) {
      currentPages.forEach(row => {
        if (row.impressions < minImpr.value) return;

        const exp = expectedCtr(row.position);
        const isSd = row.position >= sdLow.value && row.position <= sdHigh.value;
        const isLowCtr = row.position < 10 && row.ctr < exp * 0.6;

        if (isSd) {
          // In pages, Striking Distance maps to an Internal Link audit
          opps.push({
            type: "Internal Link",
            query: "(Sayfa Temelli / Sorgu Eksik)",
            page: row.page,
            impr: row.impressions,
            clicks: row.clicks,
            ctr: row.ctr * 100,
            gain: Math.round(row.impressions * 0.015),
            prio: row.impressions > 2000 ? "high" : "medium",
            pos: row.position,
            why: `Bu sayfa ortalama ${row.position.toFixed(1)}. pozisyonda ve potansiyeli yüksek. Güçlü otoriteli sayfalarınızdan yeterince iç link desteği almıyor olması sıralama yükselişini engelliyor olabilir.`,
            action: `Sitenizin en çok dış bağlantı alan ve trafik çeken sayfalarından, bu sayfaya alakalı kontekstüel link verin.`,
            source: "Sayfalar.csv"
          });
        } else if (isLowCtr) {
          opps.push({
            type: "Low CTR",
            query: "(Sayfa Temelli / Sorgu Eksik)",
            page: row.page,
            impr: row.impressions,
            clicks: row.clicks,
            ctr: row.ctr * 100,
            gain: Math.round(row.impressions * (exp - row.ctr)),
            prio: row.impressions > 1000 ? "high" : "medium",
            pos: row.position,
            why: `Sayfanız ${row.position.toFixed(1)}. sırada yer almasına rağmen tıklama oranı beklenen oranın çok altında kalıyor. Arama görünümünüz (SERP) kullanıcı çekemiyor.`,
            action: `Sayfanın başlık (title) etiketine dikkat çekici kelimeler ekleyin. Meta açıklamasını harekete geçirici mesaj ile zenginleştirin.`,
            source: "Sayfalar.csv"
          });
        }
      });
    }
  }

  const prioWeight = { high: 3, medium: 2, low: 1 };
  opps.sort((a, b) => {
    if (prioWeight[a.prio] !== prioWeight[b.prio]) {
      return prioWeight[b.prio] - prioWeight[a.prio];
    }
    return b.gain - a.gain;
  });
  calculatedOpps.value = opps;

  // 3. Cannibalization (Sorgulara göre gruplama - requires joint rows)
  const cannibal: CannibalGroup[] = [];
  if (isJoint) {
    const jointRows = isQueriesJoint ? currentQueries : currentPages;
    const queryToPages: Record<string, GscRow[]> = {};
    jointRows.forEach(row => {
      if (!row.query || !row.page) return;
      if (!queryToPages[row.query]) queryToPages[row.query] = [];
      queryToPages[row.query].push(row);
    });

    Object.keys(queryToPages).forEach(query => {
      const pages = queryToPages[query];
      if (pages.length >= 2) {
        pages.sort((a, b) => b.clicks - a.clicks);
        const totalImpr = pages.reduce((sum, p) => sum + p.impressions, 0);

        cannibal.push({
          query: query,
          totalImpr: totalImpr.toLocaleString("tr-TR"),
          count: pages.length,
          pages: pages.map((p, i) => ({
            url: p.page,
            pos: p.position.toFixed(1),
            clicks: p.clicks,
            tag: i === 0 ? "Ana" : "Rakip",
            primary: i === 0
          }))
        });
      }
    });
    cannibal.sort((a, b) => parseInt(b.totalImpr.replace(/[^\d]/g, "")) - parseInt(a.totalImpr.replace(/[^\d]/g, "")));
  }
  calculatedCannibal.value = cannibal.slice(0, 15);

  // 4. Content Decay (Geçmiş dönemi kıyasla)
  const decay: DecayRow[] = [];
  let declineSum = 0;
  let declineCount = 0;
  let growthSum = 0;
  let growthCount = 0;

  if (previousQueries.length > 0 && currentQueries.length > 0) {
    const prevMap: Record<string, GscRow> = {};
    previousQueries.forEach(row => {
      if (row.query) {
        prevMap[row.query.trim().toLowerCase()] = row;
      }
    });

    currentQueries.forEach(row => {
      if (!row.query) return;
      const key = row.query.trim().toLowerCase();
      const prevRow = prevMap[key];
      if (prevRow && prevRow.clicks > 0) {
        const clickDiff = row.clicks - prevRow.clicks;
        const posDiff = row.position - prevRow.position;
        const pct = (clickDiff / prevRow.clicks) * 100;

        if (clickDiff < 0) {
          declineSum += pct;
          declineCount++;

          decay.push({
            query: row.query,
            page: row.page || "(Sorgu Temelli)",
            prev: prevRow.clicks,
            curr: row.clicks,
            change: pct.toFixed(1) + "%",
            posDiff: posDiff > 0 ? `▼ ${posDiff.toFixed(1)}` : (posDiff < 0 ? `▲ ${Math.abs(posDiff).toFixed(1)}` : "0"),
            up: posDiff < 0
          });
        } else if (clickDiff > 0) {
          growthSum += pct;
          growthCount++;
        }
      }
    });
  } else if (previousPages.length > 0 && currentPages.length > 0) {
    const prevMap: Record<string, GscRow> = {};
    previousPages.forEach(row => {
      if (row.page) {
        prevMap[row.page.trim().toLowerCase()] = row;
      }
    });

    currentPages.forEach(row => {
      if (!row.page) return;
      const key = row.page.trim().toLowerCase();
      const prevRow = prevMap[key];
      if (prevRow && prevRow.clicks > 0) {
        const clickDiff = row.clicks - prevRow.clicks;
        const posDiff = row.position - prevRow.position;
        const pct = (clickDiff / prevRow.clicks) * 100;

        if (clickDiff < 0) {
          declineSum += pct;
          declineCount++;

          decay.push({
            query: "(Sayfa Temelli)",
            page: row.page,
            prev: prevRow.clicks,
            curr: row.clicks,
            change: pct.toFixed(1) + "%",
            posDiff: posDiff > 0 ? `▼ ${posDiff.toFixed(1)}` : (posDiff < 0 ? `▲ ${Math.abs(posDiff).toFixed(1)}` : "0"),
            up: posDiff < 0
          });
        } else if (clickDiff > 0) {
          growthSum += pct;
          growthCount++;
        }
      }
    });
  }

  decayAvgDecline.value = declineCount > 0 ? (declineSum / declineCount).toFixed(1) + "%" : "0.0%";
  decayAvgGrowth.value = growthCount > 0 ? "+" + (growthSum / growthCount).toFixed(1) + "%" : "0.0%";
  decay.sort((a, b) => (a.curr - a.prev) - (b.curr - b.prev));
  calculatedDecay.value = decay.slice(0, 20);

  // Set total row count
  rowCount.value = currentQueries.length || currentPages.length;
  hasData.value = true;
}

// CSV Parser Helper
function parseCSV(text: string): GscRow[] {
  const lines = text.split(/\r?\n/);
  if (lines.length < 2) return [];

  // Find the header line dynamically
  let headerIndex = -1;
  let delimiter = ",";

  for (let i = 0; i < Math.min(20, lines.length); i++) {
    const line = lines[i];
    let testDelim = ",";
    if (line.includes(";")) testDelim = ";";
    else if (line.includes("\t")) testDelim = "\t";

    const parts = line.split(testDelim).map(p => p.trim().replace(/^["']|["']$/g, "").toLowerCase());

    // Check if this line looks like a header row
    const hasQuery = parts.some(p => p.includes("query") || p.includes("sorgu") || p.includes("keyword"));
    const hasPage = parts.some(p => p.includes("page") || p.includes("sayfa") || p.includes("url"));
    const hasClicks = parts.some(p => p.includes("clicks") || p.includes("tıklama") || p.includes("tıklamalar") || p.includes("tik"));
    const hasImpr = parts.some(p => p.includes("impressions") || p.includes("gösterim") || p.includes("gösterimler") || p.includes("gosterim"));

    if ((hasQuery || hasPage) && (hasClicks || hasImpr)) {
      headerIndex = i;
      delimiter = testDelim;
      break;
    }
  }

  // If no header found, default to first line
  if (headerIndex === -1) {
    headerIndex = 0;
    const firstLine = lines[0];
    if (firstLine.includes(";")) delimiter = ";";
    else if (firstLine.includes("\t")) delimiter = "\t";
  }

  const headers = lines[headerIndex].split(delimiter).map(h => h.trim().replace(/^["']|["']$/g, "").toLowerCase());

  const idxQuery = headers.findIndex(h => h.includes("query") || h.includes("sorgu") || h.includes("keyword") || h.includes("date") || h.includes("tarih"));
  const idxPage = headers.findIndex(h => h.includes("page") || h.includes("sayfa") || h.includes("url") || h.includes("landing page"));
  const idxClicks = headers.findIndex(h => h.includes("clicks") || h.includes("tıklama") || h.includes("tıklamalar") || h.includes("tik"));
  const idxImpressions = headers.findIndex(h => h.includes("impressions") || h.includes("gösterim") || h.includes("gösterimler") || h.includes("gosterim"));
  const idxCtr = headers.findIndex(h => h.includes("ctr") || h.includes("to") || h.includes("tıklama oranı") || h.includes("tıklama orani"));
  const idxPosition = headers.findIndex(h => h.includes("position") || h.includes("pozisyon") || h.includes("pos") || h.includes("konum") || h.includes("ortalama konum") || h.includes("sira") || h.includes("sıra"));

  const rows: GscRow[] = [];

  for (let i = headerIndex + 1; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line) continue;

    let parts: string[] = [];
    let insideQuote = false;
    let currentPart = "";
    for (let j = 0; j < line.length; j++) {
      const char = line[j];
      if (char === '"' || char === "'") {
        insideQuote = !insideQuote;
      } else if (char === delimiter && !insideQuote) {
        parts.push(currentPart.trim().replace(/^["']|["']$/g, ""));
        currentPart = "";
      } else {
        currentPart += char;
      }
    }
    parts.push(currentPart.trim().replace(/^["']|["']$/g, ""));

    if (parts.length < headers.length) continue;

    const query = idxQuery !== -1 ? parts[idxQuery] : "";
    const page = idxPage !== -1 ? parts[idxPage] : "";
    const clicks = idxClicks !== -1 ? parseInt(parts[idxClicks].replace(/[^\d]/g, "")) || 0 : 0;
    const impressions = idxImpressions !== -1 ? parseInt(parts[idxImpressions].replace(/[^\d]/g, "")) || 0 : 0;

    let ctrVal = 0;
    if (idxCtr !== -1) {
      let rawCtr = parts[idxCtr].replace("%", "").trim();
      rawCtr = rawCtr.replace(",", ".");
      const parsed = parseFloat(rawCtr);
      if (!isNaN(parsed)) {
        if (parts[idxCtr].includes("%") || parsed > 1.0) {
          ctrVal = parsed / 100;
        } else {
          ctrVal = parsed;
        }
      }
    } else {
      ctrVal = impressions > 0 ? clicks / impressions : 0;
    }

    let position = 0;
    if (idxPosition !== -1) {
      let rawPos = parts[idxPosition].trim().replace(",", ".");
      position = parseFloat(rawPos) || 0;
    }

    if (query || page) {
      rows.push({
        query,
        page,
        clicks,
        impressions,
        ctr: ctrVal,
        position
      });
    }
  }

  return rows;
}

// File Inputs Refs
const currentFileInput = ref<HTMLInputElement | null>(null);
const prevFileInput = ref<HTMLInputElement | null>(null);

function triggerCurrentUpload() {
  currentFileInput.value?.click();
}

function triggerPreviousUpload() {
  prevFileInput.value?.click();
}

function parseDailyChart(text: string): DailyMetric[] {
  const parsed = parseCSV(text);
  const list: DailyMetric[] = [];
  parsed.forEach(row => {
    if (row.query && /^\d{4}-\d{2}-\d{2}$/.test(row.query)) {
      list.push({
        date: row.query,
        clicks: row.clicks,
        impressions: row.impressions,
        ctr: row.ctr,
        position: row.position
      });
    }
  });
  list.sort((a, b) => a.date.localeCompare(b.date));
  return list;
}

function handleCurrentFile(file: File) {
  currentFileName.value = file.name;
  uploading.value = true;

  if (file.name.endsWith(".zip")) {
    const reader = new FileReader();
    reader.onload = async (e) => {
      try {
        const arrayBuffer = e.target?.result as ArrayBuffer;
        const zip = await JSZip.loadAsync(arrayBuffer);
        
        let queriesText = "";
        let pagesText = "";
        let chartText = "";
        
        const fileNames = Object.keys(zip.files);
        for (const name of fileNames) {
          const lowerName = name.toLowerCase();
          if (lowerName.includes("sorgu") || lowerName.includes("query")) {
            queriesText = await zip.files[name].async("text");
          } else if (lowerName.includes("sayfa") || lowerName.includes("page")) {
            pagesText = await zip.files[name].async("text");
          } else if (lowerName.includes("grafik") || lowerName.includes("date")) {
            chartText = await zip.files[name].async("text");
          }
        }
        
        if (queriesText) parsedCurrentQueries.value = parseCSV(queriesText);
        if (pagesText) parsedCurrentPages.value = parseCSV(pagesText);
        if (chartText) parsedDailyMetrics.value = parseDailyChart(chartText);
        
        if (queriesText || pagesText) {
          processGscData();
        } else {
          alert("ZIP dosyası içinde 'Sorgular' veya 'Sayfalar' CSV dosyası bulunamadı.");
        }
      } catch (err: any) {
        alert("ZIP dosyası açılırken bir hata oluştu: " + err.message);
      } finally {
        uploading.value = false;
      }
    };
    reader.readAsArrayBuffer(file);
  } else {
    const reader = new FileReader();
    reader.onload = (e) => {
      const text = e.target?.result as string;
      const rows = parseCSV(text);
      
      const hasQueries = rows.some(r => r.query);
      const hasPages = rows.some(r => r.page);
      
      if (hasQueries && hasPages) {
        parsedCurrentQueries.value = rows;
        parsedCurrentPages.value = rows;
      } else if (hasQueries) {
        parsedCurrentQueries.value = rows;
        parsedCurrentPages.value = [];
      } else if (hasPages) {
        parsedCurrentPages.value = rows;
        parsedCurrentQueries.value = [];
      }
      
      processGscData();
      uploading.value = false;
    };
    reader.readAsText(file);
  }
}

function handlePreviousFile(file: File) {
  previousFileName.value = file.name;

  if (file.name.endsWith(".zip")) {
    const reader = new FileReader();
    reader.onload = async (e) => {
      try {
        const arrayBuffer = e.target?.result as ArrayBuffer;
        const zip = await JSZip.loadAsync(arrayBuffer);
        
        let queriesText = "";
        let pagesText = "";
        
        const fileNames = Object.keys(zip.files);
        for (const name of fileNames) {
          const lowerName = name.toLowerCase();
          if (lowerName.includes("sorgu") || lowerName.includes("query")) {
            queriesText = await zip.files[name].async("text");
          } else if (lowerName.includes("sayfa") || lowerName.includes("page")) {
            pagesText = await zip.files[name].async("text");
          }
        }
        
        if (queriesText) parsedPreviousQueries.value = parseCSV(queriesText);
        if (pagesText) parsedPreviousPages.value = parseCSV(pagesText);
        
        processGscData();
      } catch (err: any) {
        alert("Geçmiş ZIP dosyası açılırken hata: " + err.message);
      }
    };
    reader.readAsArrayBuffer(file);
  } else {
    const reader = new FileReader();
    reader.onload = (e) => {
      const text = e.target?.result as string;
      const rows = parseCSV(text);
      
      const hasQueries = rows.some(r => r.query);
      const hasPages = rows.some(r => r.page);
      
      if (hasQueries && hasPages) {
        parsedPreviousQueries.value = rows;
        parsedPreviousPages.value = rows;
      } else if (hasQueries) {
        parsedPreviousQueries.value = rows;
        parsedPreviousPages.value = [];
      } else if (hasPages) {
        parsedPreviousPages.value = rows;
        parsedPreviousQueries.value = [];
      }
      
      processGscData();
    };
    reader.readAsText(file);
  }
}

function onCurrentFileChange(e: Event) {
  const files = (e.target as HTMLInputElement).files;
  if (files && files.length > 0) {
    handleCurrentFile(files[0]);
  }
}

function onPreviousFileChange(e: Event) {
  const files = (e.target as HTMLInputElement).files;
  if (files && files.length > 0) {
    handlePreviousFile(files[0]);
  }
}

// Drag & Drop Handlers
function onDragOver(e: DragEvent) {
  e.preventDefault();
}

function onCurrentDrop(e: DragEvent) {
  e.preventDefault();
  const files = e.dataTransfer?.files;
  if (files && files.length > 0) {
    handleCurrentFile(files[0]);
  }
}

// Drag & Drop Handlers
function onPreviousDrop(e: DragEvent) {
  e.preventDefault();
  const files = e.dataTransfer?.files;
  if (files && files.length > 0) {
    handlePreviousFile(files[0]);
  }
}

// Mock upload simulation is disabled per user request
// function simulateUpload() {
//   uploading.value = true;
//   setTimeout(() => {
//     loadMockData();
//     uploading.value = false;
//   }, 600);
// }

// Load Mock Data
function loadMockData() {
  currentFileName.value = "query_data.csv";
  previousFileName.value = "query_data_previous.csv";

  kpis.value = [
    { label: "Toplam Gösterim", value: "847.2K", trend: "+12.4%", icon: "eye", up: true },
    { label: "Toplam Tıklama", value: "23.8K", trend: "+8.1%", icon: "click", up: true },
    { label: "Ortalama CTR", value: "%2.81", trend: "-0.3%", icon: "percent", up: false },
    { label: "Ortalama Pozisyon", value: "14.2", trend: "+2.1", icon: "rank", up: true },
  ];

  calculatedOpps.value = [
    { type: "Striking Distance", query: "içerik pazarlama araçları", page: "/blog/icerik-araclari", impr: 14200, clicks: 298, ctr: 2.1, gain: 640, prio: "high", pos: 11.4, why: "Bu sorgu 11.4 pozisyonunda — 2. sayfanın hemen başında yer alıyor. İlk 5'e taşınırsa tıklama oranı 3-4 katına çıkabilir. Zaten yüksek gösterim aldığı için düşük çabayla yüksek getiri sağlar.", action: 'Sayfaya "2026 güncel" ifadesini ve 3 yeni araç ekleyin. Ardından yüksek otoriteli /rehber sayfanızdan bu sayfaya alakalı anchor metniyle iç link verin.', source: "Birleşik Veri" },
    { type: "Low CTR", query: "seo denetimi nasıl yapılır", page: "/rehber/seo-denetimi", impr: 9800, clicks: 88, ctr: 0.9, gain: 420, prio: "high", pos: 4.2, why: "4. pozisyonda olmasına rağmen CTR yalnızca %0,9. Bu, başlık ve açıklamanızın tıklamayı teşvik etmediğini gösterir. Sıralama iyi ama sunum zayıf.", action: 'FAQ şeması ekleyin ve başlığı soru formatına çevirin. Meta açıklamaya adım sayısı ve süre bilgisi ("7 adımda, 15 dk") ekleyin.', source: "Birleşik Veri" },
    { type: "Internal Link", query: "anahtar kelime araştırması", page: "/blog/kelime-arastirmasi", impr: 6400, clicks: 210, ctr: 3.3, gain: 280, prio: "medium", pos: 8.1, why: "Güçlü bir sayfa ama iç link desteği zayıf. Sitenizin en çok bağlantı alan 3 sayfasından hiçbiri buraya link vermiyor.", action: "/seo-rehberi ve /baslangic sayfalarından bu içeriğe \"anahtar kelime araştırması\" anchor metniyle iç link ekleyin.", source: "Birleşik Veri" },
    { type: "Striking Distance", query: "backlink analizi", page: "/arac/backlink", impr: 5100, clicks: 142, ctr: 2.8, gain: 230, prio: "medium", pos: 12.7, why: "12.7 pozisyonu — biraz destekle ilk sayfaya taşınabilir. Rakip analizinde bu sorgu için içerik boşluğu tespit edildi.", action: "İçeriğe karşılaştırma tablosu ve güncel istatistikler ekleyip kelime sayısını 800'den 1500'e çıkarın.", source: "Birleşik Veri" },
    { type: "Low CTR", query: "ücretsiz seo araçları", page: "/liste/ucretsiz-araclar", impr: 11300, clicks: 135, ctr: 1.2, gain: 390, prio: "high", pos: 6.8, why: "Yüksek gösterim, düşük CTR. \"Ücretsiz\" arayan kullanıcılar için başlık yeterince cazip değil.", action: 'Başlığa araç sayısını ve "kayıt gerektirmez" ifadesini ekleyin. Liste öğelerini yıldız derecelendirmesiyle zenginleştirin.', source: "Birleşik Veri" },
    { type: "Internal Link", query: "site hızı optimizasyonu", page: "/rehber/site-hizi", impr: 4200, clicks: 96, ctr: 2.3, gain: 180, prio: "low", pos: 9.4, why: "Orta seviye bir fırsat. Teknik SEO kümesindeki diğer sayfalarla bağlantısı kopuk.", action: "Teknik SEO küme sayfanızdan (pillar page) bu içeriğe kontekstüel iç link ekleyin.", source: "Birleşik Veri" },
    { type: "Striking Distance", query: "schema markup örnekleri", page: "/blog/schema-ornekleri", impr: 3800, clicks: 71, ctr: 1.9, gain: 160, prio: "low", pos: 13.2, why: "13.2 pozisyonunda, geliştirme potansiyeli var. Kod örnekleri arayan kullanıcılar için değerli.", action: "Kopyalanabilir kod blokları ve her şema tipi için canlı örnek ekleyin.", source: "Birleşik Veri" },
    { type: "Low CTR", query: "google analytics kurulumu", page: "/rehber/ga4-kurulum", impr: 7600, clicks: 152, ctr: 2.0, gain: 250, prio: "medium", pos: 5.1, why: "5.1 pozisyonunda ancak CTR düşük. GA4 geçişiyle arama hacmi arttı, fırsat büyüyor.", action: 'Başlığa "GA4" ve "2026" ekleyin, ekran görüntülü adım adım rehber vurgusu yapın.', source: "Birleşik Veri" },
  ];

  calculatedCannibal.value = [
    { query: "seo aracı", totalImpr: "21.400", count: 3, pages: [
      { url: "/arac/seo-analiz", pos: "4.0", clicks: 180, tag: "Ana", primary: true },
      { url: "/blog/en-iyi-seo-araclari", pos: "7.0", clicks: 95, tag: "Rakip", primary: false },
      { url: "/liste/seo-araclari-2026", pos: "11.0", clicks: 34, tag: "Rakip", primary: false },
    ]},
    { query: "anahtar kelime bulma", totalImpr: "13.800", count: 2, pages: [
      { url: "/arac/kelime-bulucu", pos: "5.0", clicks: 120, tag: "Ana", primary: true },
      { url: "/blog/kelime-nasil-bulunur", pos: "9.0", clicks: 62, tag: "Rakip", primary: false },
    ]},
    { query: "içerik planlama", totalImpr: "8.900", count: 3, pages: [
      { url: "/rehber/icerik-plani", pos: "6.0", clicks: 88, tag: "Ana", primary: true },
      { url: "/sablon/icerik-takvimi", pos: "8.0", clicks: 54, tag: "Rakip", primary: false },
      { url: "/blog/icerik-stratejisi", pos: "14.0", clicks: 19, tag: "Rakip", primary: false },
    ]},
  ];

  calculatedDecay.value = [
    { query: "seo nedir", page: "/rehber/seo-nedir", prev: 1240, curr: 820, change: "-33.9%", posDiff: "▼ 3.2", up: false },
    { query: "meta açıklama", page: "/blog/meta-aciklama", prev: 680, curr: 512, change: "-24.7%", posDiff: "▼ 1.8", up: false },
    { query: "sitemap oluşturma", page: "/rehber/sitemap", prev: 420, curr: 290, change: "-31.0%", posDiff: "▼ 2.5", up: false },
    { query: "core web vitals", page: "/blog/web-vitals", prev: 310, curr: 540, change: "+74.2%", posDiff: "▲ 4.1", up: true },
    { query: "robots.txt", page: "/rehber/robots-txt", prev: 520, curr: 380, change: "-26.9%", posDiff: "▼ 1.2", up: false },
    { query: "e-e-a-t nedir", page: "/blog/eeat", prev: 180, curr: 410, change: "+127.8%", posDiff: "▲ 6.3", up: true },
    { query: "canonical etiket", page: "/rehber/canonical", prev: 290, curr: 198, change: "-31.7%", posDiff: "▼ 2.1", up: false },
    { query: "yapay zeka seo", page: "/blog/ai-seo", prev: 95, curr: 380, change: "+300.0%", posDiff: "▲ 9.7", up: true },
  ];

  rowCount.value = 2847;
  hasData.value = true;
}

function clearData() {
  hasData.value = false;
  parsedCurrentQueries.value = [];
  parsedCurrentPages.value = [];
  parsedPreviousQueries.value = [];
  parsedPreviousPages.value = [];
  parsedDailyMetrics.value = [];
  calculatedOpps.value = [];
  calculatedCannibal.value = [];
  calculatedDecay.value = [];
  currentFileName.value = "";
  previousFileName.value = "";
  rowCount.value = 0;
}

function startGscOAuth() {
  const clientId = gscClientId.value.trim();
  if (!clientId) {
    alert("Lütfen Google OAuth ile giriş yapabilmek için önce geçerli bir Client ID girin.");
    return;
  }
  const redirectUri = window.location.origin + "/";
  const scopes = "https://www.googleapis.com/auth/webmasters.readonly";
  const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?client_id=${clientId}&redirect_uri=${encodeURIComponent(redirectUri)}&response_type=token&scope=${encodeURIComponent(scopes)}`;
  
  try {
    openBrowser(authUrl);
  } catch (e) {
    console.error("Tauri opener failed, falling back to window.open", e);
    window.open(authUrl, "_blank");
  }
}

watch(gscClientId, (newVal) => {
  localStorage.setItem("gsc_client_id", newVal);
});

function isNewerVersion(current: string, latest: string): boolean {
  const currParts = current.split(".").map(Number);
  const lateParts = latest.split(".").map(Number);
  for (let i = 0; i < Math.max(currParts.length, lateParts.length); i++) {
    const currVal = currParts[i] || 0;
    const lateVal = lateParts[i] || 0;
    if (lateVal > currVal) return true;
    if (lateVal < currVal) return false;
  }
  return false;
}

async function checkForUpdates() {
  try {
    const res = await fetch(`https://api.github.com/repos/${githubRepo.value}/releases/latest`);
    if (!res.ok) return;
    const data = await res.json();
    if (!data.tag_name) return;
    const latestVersion = data.tag_name.replace("v", "").trim();
    
    if (isNewerVersion(currentVersion, latestVersion)) {
      latestReleaseData.value = {
        version: data.tag_name,
        body: data.body || "",
        url: data.html_url,
        assets: data.assets || []
      };
      showUpdateModal.value = true;
    }
  } catch (err) {
    console.error("Güncelleme kontrolü başarısız:", err);
  }
}



function triggerUpdateInstall() {
  if (latestReleaseData.value) {
    try {
      openBrowser(latestReleaseData.value.url);
    } catch (e) {
      window.open(latestReleaseData.value.url, "_blank");
    }
    showUpdateModal.value = false;
  }
}

async function connectWithToken() {
  const token = gscAccessToken.value.trim();
  if (!token) {
    alert("Lütfen geçerli bir Access Token yapıştırın.");
    return;
  }
  
  try {
    const res = await fetch("https://www.googleapis.com/webmasters/v3/sites", {
      headers: {
        Authorization: `Bearer ${token}`
      }
    });
    if (!res.ok) {
      throw new Error("Token geçersiz veya süresi dolmuş. (HTTP " + res.status + ")");
    }
    const data = await res.json();
    gscSites.value = (data.siteEntry || []).map((s: any) => s.siteUrl);
    
    gscConnected.value = true;
    localStorage.setItem("gsc_access_token", token);
    
    if (gscSites.value.length > 0) {
      selectedGscSite.value = gscSites.value[0];
      localStorage.setItem("selected_gsc_site", selectedGscSite.value);
    }
    
    alert("Google Search Console API bağlantısı başarıyla kuruldu!");
  } catch (err: any) {
    alert("Bağlantı Hatası: " + err.message);
  }
}

function reportWithAi(opp: Opportunity) {
  setView("ai");
  prompt.value = `Lütfen şu fırsat için detaylı bir SEO eylem planı, başlık/meta açıklaması önerileri ve içerik geliştirme stratejisi hazırlar mısın?

- Fırsat Tipi: ${opp.type}
- Sorgu: "${opp.query}"
- Hedef URL: ${opp.page}
- Mevcut Ortalama Pozisyon: ${opp.pos.toFixed(1)}
- Mevcut Tıklama Oranı: %${opp.ctr.toFixed(1)}
- Son 28 Günlük Gösterim: ${opp.impr.toLocaleString('tr-TR')}
- Tahmini Potansiyel Aylık Kazanç: +${opp.gain.toLocaleString('tr-TR')} tıklama/ay
- Öncelik Derecesi: ${opp.prio === 'high' ? 'Yüksek' : (opp.prio === 'medium' ? 'Orta' : 'Düşük')}

Lütfen neden önemli olduğunu ve tam olarak hangi adımların atılması gerektiğini detaylıca açıklar mısın?`;

  sendPrompt();
}

async function fetchGscSites() {
  const token = gscAccessToken.value;
  if (!token) return;
  
  try {
    const res = await fetch("https://www.googleapis.com/webmasters/v3/sites", {
      headers: {
        Authorization: `Bearer ${token}`
      }
    });
    if (!res.ok) throw new Error("GSC siteleri alınamadı.");
    const data = await res.json();
    gscSites.value = (data.siteEntry || []).map((s: any) => s.siteUrl);
    if (gscSites.value.length > 0) {
      if (!selectedGscSite.value || !gscSites.value.includes(selectedGscSite.value)) {
        selectedGscSite.value = gscSites.value[0];
        localStorage.setItem("selected_gsc_site", selectedGscSite.value);
      }
    }
  } catch (err: any) {
    console.error("GSC siteleri çekilirken hata:", err);
  }
}

async function loadDataFromGscApi() {
  const token = gscAccessToken.value;
  const site = selectedGscSite.value;
  if (!token || !site) {
    alert("Lütfen önce Google Search Console bağlantısını yapın ve sitenizi seçin.");
    return;
  }
  
  loadingGscData.value = true;
  
  try {
    const today = new Date();
    const formatGscDate = (d: Date) => d.toISOString().split("T")[0];
    
    const end = new Date(today.getTime() - 3 * 24 * 60 * 60 * 1000);
    const start = new Date(end.getTime() - 27 * 24 * 60 * 60 * 1000);
    
    const prevEnd = new Date(start.getTime() - 1 * 24 * 60 * 60 * 1000);
    const prevStart = new Date(prevEnd.getTime() - 27 * 24 * 60 * 60 * 1000);
    
    const siteEncoded = encodeURIComponent(site);
    const queryUrl = `https://www.googleapis.com/webmasters/v3/sites/${siteEncoded}/searchAnalytics/query`;
    
    const resCurr = await fetch(queryUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        startDate: formatGscDate(start),
        endDate: formatGscDate(end),
        dimensions: ["query", "page"],
        rowLimit: 2500
      })
    });
    
    const resPrev = await fetch(queryUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        startDate: formatGscDate(prevStart),
        endDate: formatGscDate(prevEnd),
        dimensions: ["query", "page"],
        rowLimit: 2500
      })
    });
    
    const resDaily = await fetch(queryUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        startDate: formatGscDate(start),
        endDate: formatGscDate(end),
        dimensions: ["date"],
        rowLimit: 100
      })
    });
    
    if (!resCurr.ok || !resPrev.ok || !resDaily.ok) {
      throw new Error("GSC API'den veri alınırken hata oluştu.");
    }
    
    const dataCurr = await resCurr.json();
    const dataPrev = await resPrev.json();
    const dataDaily = await resDaily.json();
    
    const mapGscApiRows = (rows: any[]): GscRow[] => {
      return (rows || []).map(r => ({
        query: r.keys?.[0] || "",
        page: r.keys?.[1] || "",
        clicks: r.clicks || 0,
        impressions: r.impressions || 0,
        ctr: r.ctr || 0,
        position: r.position || 0
      }));
    };
    
    const currRows = mapGscApiRows(dataCurr.rows);
    const prevRows = mapGscApiRows(dataPrev.rows);
    
    parsedCurrentQueries.value = currRows;
    parsedCurrentPages.value = currRows;
    
    parsedPreviousQueries.value = prevRows;
    parsedPreviousPages.value = prevRows;
    
    parsedDailyMetrics.value = (dataDaily.rows || []).map((r: any) => ({
      date: r.keys?.[0] || "",
      clicks: r.clicks || 0,
      impressions: r.impressions || 0,
      ctr: r.ctr || 0,
      position: r.position || 0
    })).sort((a: any, b: any) => a.date.localeCompare(b.date));
    
    currentFileName.value = `GSC API: ${site}`;
    previousFileName.value = `GSC API (Önceki Dönem)`;
    rowCount.value = currRows.length;
    hasData.value = true;
    
    processGscData();
    
  } catch (err: any) {
    alert("GSC API veri yükleme hatası: " + err.message);
  } finally {
    loadingGscData.value = false;
  }
}

const filteredOpps = computed(() => {
  const q = search.value.trim().toLowerCase();
  return calculatedOpps.value.filter(o => {
    const matchesFilter = filter.value === "all" || o.type === filter.value;
    const matchesSearch = !q || o.query.toLowerCase().includes(q) || o.page.toLowerCase().includes(q);
    
    let matchesSource = true;
    if (o.source === "Sorgular.csv") matchesSource = showSourceQueries.value;
    else if (o.source === "Sayfalar.csv") matchesSource = showSourcePages.value;
    else if (o.source === "Birleşik Veri") matchesSource = showSourceJoint.value;
    
    return matchesFilter && matchesSearch && matchesSource;
  });
});

const sortedFilteredOpps = computed(() => {
  const list = [...filteredOpps.value];
  const col = sortBy.value;
  const order = sortOrder.value === "asc" ? 1 : -1;

  const prioWeight = { high: 3, medium: 2, low: 1 };

  list.sort((a, b) => {
    let valA: string | number = a[col as keyof Opportunity] as any;
    let valB: string | number = b[col as keyof Opportunity] as any;

    if (col === "prio") {
      valA = prioWeight[a.prio] || 0;
      valB = prioWeight[b.prio] || 0;
    }

    if (typeof valA === "string") {
      const strA = valA;
      const strB = String(valB);
      return strA.localeCompare(strB) * order;
    } else {
      const numA = Number(valA) || 0;
      const numB = Number(valB) || 0;
      return (numA - numB) * order;
    }
  });

  return list;
});

const chartBars = computed(() => {
  if (parsedDailyMetrics.value.length > 0) {
    const maxClicks = Math.max(...parsedDailyMetrics.value.map(d => d.clicks)) || 1;
    return parsedDailyMetrics.value.map((d, idx) => {
      const h = Math.max(12, Math.min(95, (d.clicks / maxClicks) * 95));
      const color = idx >= parsedDailyMetrics.value.length - 3 ? "#E1251A" : "rgba(225, 37, 26, 0.35)";
      return {
        h: h + "%",
        color,
        label: `${d.date}: ${d.clicks} tık`
      };
    });
  }
  return Array.from({ length: 28 }, (_, i) => {
    const base = 40 + Math.sin(i * 0.6) * 22 + (i > 20 ? (i - 20) * 3 : 0) + (i % 4 === 0 ? 12 : 0);
    const h = Math.max(18, Math.min(95, base + (i % 3) * 6));
    return {
      h: h + "%",
      color: i >= 25 ? "#E1251A" : "rgba(225, 37, 26, 0.35)",
      label: `Gün ${i + 1}`
    };
  });
});

// Opportunity Distribution percentages
const oppDist = computed(() => {
  const counts = { "Striking Distance": 0, "Low CTR": 0, "Internal Link": 0 };
  calculatedOpps.value.forEach(o => {
    if (counts[o.type] !== undefined) counts[o.type]++;
  });

  const total = calculatedOpps.value.length || 1;
  return [
    { label: "Striking Distance", count: counts["Striking Distance"], pct: ((counts["Striking Distance"] / total) * 100) + "%", color: "#E1251A" },
    { label: "Low CTR", count: counts["Low CTR"], pct: ((counts["Low CTR"] / total) * 100) + "%", color: "#34d399" },
    { label: "Internal Link", count: counts["Internal Link"], pct: ((counts["Internal Link"] / total) * 100) + "%", color: "#f59e0b" },
  ];
});

// Gemini Chat functionality
async function sendPrompt() {
  const p = prompt.value.trim();
  if (!p) return;

  chat.value.push({ role: "user", text: p });
  prompt.value = "";

  const loadingIndex = chat.value.length;
  chat.value.push({ role: "ai", text: "Düşünüyor..." });

  try {
    const activeApiKey = apiKey.value.trim();
    if (!activeApiKey) {
      chat.value[loadingIndex] = { role: "ai", text: "Hata: Gemini API Anahtarı bulunamadı. Lütfen Ayarlar sayfasından geçerli bir API Anahtarı girin." };
      return;
    }
    const selectedModel = model.value === "pro" ? "gemini-2.5-pro" : (model.value === "flash2" ? "gemini-2.0-flash" : "gemini-2.5-flash");
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${selectedModel}:generateContent?key=${activeApiKey}`;

    let context = "Sen GSC Performance analizi yapan uzman bir SEO asistanısın. Kullanıcı şu GSC verilerini yükledi:\n";
    if (kpis.value.length > 0) {
      context += `KPIs: Toplam Gösterim: ${kpis.value[0]?.value}, Toplam Tıklama: ${kpis.value[1]?.value}, Ortalama CTR: ${kpis.value[2]?.value}, Ortalama Pozisyon: ${kpis.value[3]?.value}\n`;
    }
    context += `En önemli fırsatlar:\n`;
    calculatedOpps.value.slice(0, 5).forEach((o, i) => {
      context += `${i + 1}. [${o.type}] Sorgu: "${o.query}", Sayfa: "${o.page}", Gösterim: ${o.impr}, CTR: %${o.ctr.toFixed(1)}, Pozisyon: ${o.pos.toFixed(1)}, Tahmini Kazanç: +${o.gain} tıklama/ay.\n`;
    });
    context += `Kullanıcının sorusu: ${p}\nLütfen bu verilere dayanarak Türkçe, son derece net, profesyonel ve aksiyon odaklı bir yanıt raporu ver.`;

    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        contents: [{ parts: [{ text: context }] }]
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP Hatası: ${response.status}`);
    }

    const data = await response.json();
    const replyText = data.candidates?.[0]?.content?.parts?.[0]?.text || "Yanıt alınamadı.";
    chat.value[loadingIndex] = { role: "ai", text: replyText };
  } catch (error: any) {
    chat.value[loadingIndex] = { role: "ai", text: `Yapay zeka motoruna bağlanırken bir hata oluştu. Lütfen API anahtarınızı veya internet bağlantınızı kontrol edin. Hata: ${error.message || error}` };
  }
}

// Close Opp details drawer
function closeDrawer() {
  selectedOpp.value = null;
}

const chatContainer = ref<HTMLElement | null>(null);

function scrollToBottom() {
  setTimeout(() => {
    if (chatContainer.value) {
      chatContainer.value.scrollTop = chatContainer.value.scrollHeight;
    }
  }, 100);
}

function clearChatHistory() {
  if (confirm("Sohbet geçmişini sıfırlamak istediğinize emin misiniz?")) {
    chat.value = [
      { role: "ai", text: 'Merhaba! Ben QueryGSC Yapay Zeka SEO Asistanıyım. Google Search Console verilerinizi yükledikten veya API ile bağlandıktan sonra, verilerinizi otomatik olarak okuyabilir, en kritik SEO fırsatlarını tespit edebilir ve size özel eylem planları hazırlayabilirim. Fırsat Kaşifi üzerinden bir kelime seçip "Yapay Zeka Asistanı ile Raporla" diyerek veya buraya doğrudan sorunuzu yazarak başlayabilirsiniz. Nasıl yardımcı olabilirim?' }
    ];
    localStorage.setItem("gsc_chat_history", JSON.stringify(chat.value));
  }
}

function renderMarkdown(text: string): string {
  try {
    return marked.parse(text) as string;
  } catch (err) {
    console.error("Markdown parse error:", err);
    return text;
  }
}

// Select view from settings or widgets
const setView = (view: string) => {
  activeView.value = view;
  closeDrawer();
  if (view === "ai") {
    scrollToBottom();
  }
};

watch(chat, (newVal) => {
  localStorage.setItem("gsc_chat_history", JSON.stringify(newVal));
  scrollToBottom();
}, { deep: true });

// API Key changes storage handling
function onApiKeyInput(e: Event) {
  const value = (e.target as HTMLInputElement).value;
  apiKey.value = value;
  localStorage.setItem("gemini_api_key", value);
}

function handleGscDisconnect() {
  gscConnected.value = false;
  gscAccessToken.value = "";
  gscSites.value = [];
  selectedGscSite.value = "";
  localStorage.removeItem("gsc_access_token");
  localStorage.removeItem("selected_gsc_site");
}

function onGscTokenInput(e: Event) {
  const value = (e.target as HTMLInputElement).value.trim();
  gscAccessToken.value = value;
}

function saveSelectedGscSite() {
  localStorage.setItem("selected_gsc_site", selectedGscSite.value);
}

// Save threshold settings to localStorage
function saveThresholds() {
  localStorage.setItem("gsc_min_impr", minImpr.value.toString());
  localStorage.setItem("gsc_sd_low", sdLow.value.toString());
  localStorage.setItem("gsc_sd_high", sdHigh.value.toString());
  
  if (parsedCurrentQueries.value.length > 0 || parsedCurrentPages.value.length > 0) {
    processGscData();
  } else if (hasData.value) {
    // If using mock data, reload mock data to reflect settings
    loadMockData();
  }
}

watch(geminiOn, (newVal) => {
  localStorage.setItem("gemini_api_on", newVal.toString());
});

onMounted(() => {
  // Check if we have oauth hash parameters in URL
  if (window.location.hash) {
    const hash = window.location.hash.substring(1);
    const params = new URLSearchParams(hash);
    const accessToken = params.get("access_token");
    if (accessToken) {
      gscAccessToken.value = accessToken;
      localStorage.setItem("gsc_access_token", accessToken);
      gscConnected.value = true;
      // Clear hash from address bar
      window.history.replaceState(null, "", window.location.pathname);
      fetchGscSites();
    }
  }

  // Load configuration from LocalStorage
  const storedKey = localStorage.getItem("gemini_api_key");
  if (storedKey) apiKey.value = storedKey;

  const storedMinImpr = localStorage.getItem("gsc_min_impr");
  if (storedMinImpr) minImpr.value = parseInt(storedMinImpr) || 100;

  const storedSdLow = localStorage.getItem("gsc_sd_low");
  if (storedSdLow) sdLow.value = parseInt(storedSdLow) || 4;

  const storedSdHigh = localStorage.getItem("gsc_sd_high");
  if (storedSdHigh) sdHigh.value = parseInt(storedSdHigh) || 20;

  const storedGeminiOn = localStorage.getItem("gemini_api_on");
  if (storedGeminiOn) geminiOn.value = storedGeminiOn === "true";

  const storedChat = localStorage.getItem("gsc_chat_history");
  if (storedChat) {
    try {
      chat.value = JSON.parse(storedChat);
    } catch (e) {
      console.error("Sohbet geçmişi yüklenirken hata:", e);
    }
  }

  const storedGscToken = localStorage.getItem("gsc_access_token");
  if (storedGscToken) {
    gscAccessToken.value = storedGscToken;
    gscConnected.value = true;
    fetchGscSites().then(() => {
      const storedSite = localStorage.getItem("selected_gsc_site");
      if (storedSite && gscSites.value.includes(storedSite)) {
        selectedGscSite.value = storedSite;
      }
    });
  }

  checkForUpdates();
});
</script>

<template>
  <div class="root-layout">
    <!-- SIDEBAR -->
    <aside class="sidebar">
      <!-- App Identity -->
      <div class="identity">
        <div class="logo-box">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M3 3v18h18" />
            <path d="m7 14 4-4 3 3 5-6" />
          </svg>
        </div>
        <div>
          <div class="app-name">QueryGSC</div>
          <div class="app-sub">SEO Analiz Motoru</div>
        </div>
      </div>

      <!-- Navigation Menu -->
      <nav class="nav-menu">
        <div class="nav-section-title">ÇALIŞMA ALANI</div>
        <button 
          v-for="item in navItems" 
          :key="item.id" 
          @click="setView(item.id)" 
          :class="['nav-item', { active: activeView === item.id }]"
        >
          <span 
            class="nav-icon" 
            :style="{ color: activeView === item.id ? '#ffb3ab' : '#7e776c' }"
          >
            <!-- Custom Inline SVG Icons based on design mockup -->
            <svg 
              v-if="item.icon === 'grid'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <rect x="3" y="3" width="7" height="7" rx="1.5"/><rect x="14" y="3" width="7" height="7" rx="1.5"/><rect x="3" y="14" width="7" height="7" rx="1.5"/><rect x="14" y="14" width="7" height="7" rx="1.5"/>
            </svg>
            <svg 
              v-else-if="item.icon === 'target'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <circle cx="12" cy="12" r="9"/><circle cx="12" cy="12" r="5"/><circle cx="12" cy="12" r="1"/>
            </svg>
            <svg 
              v-else-if="item.icon === 'split'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <path d="M16 3h5v5"/><path d="M8 3H3v5"/><path d="M21 3l-7.5 7.5a4 4 0 0 0-1.5 3.1V21"/><path d="M3 3l7.5 7.5"/>
            </svg>
            <svg 
              v-else-if="item.icon === 'trend'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <path d="M22 7 13.5 15.5 8.5 10.5 2 17"/><path d="M16 7h6v6"/>
            </svg>
            <svg 
              v-else-if="item.icon === 'spark'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <path d="M12 3l1.9 5.8a2 2 0 0 0 1.3 1.3L21 12l-5.8 1.9a2 2 0 0 0-1.3 1.3L12 21l-1.9-5.8a2 2 0 0 0-1.3-1.3L3 12l5.8-1.9a2 2 0 0 0 1.3-1.3z"/>
            </svg>
            <svg 
              v-else-if="item.icon === 'gear'" 
              width="100%" height="100%" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"
            >
              <circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 1 1-2.83 2.83l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-4 0v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 1 1-2.83-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1 0-4h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 1 1 2.83-2.83l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 4 0v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 1 1 2.83 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 0 4h-.09a1.65 1.65 0 0 0-1.51 1z"/>
            </svg>
          </span>
          <span class="nav-label">{{ item.label }}</span>
          <span v-if="item.badge" class="nav-badge">{{ item.badge }}</span>
        </button>
      </nav>

      <!-- User Account Info (Dynamic connection profile) -->
      <div class="user-box">
        <div class="account-card">
          <div class="avatar" :style="{ background: userTier === 'Premium Plan' ? 'linear-gradient(135deg, #E1251A, #a0100a)' : 'rgba(255,255,255,0.08)' }">
            {{ userInitials }}
          </div>
          <div class="account-details">
            <div class="username" :title="userName">{{ userName }}</div>
            <div class="plan-label" :style="{ color: userTier === 'Premium Plan' ? '#34d399' : '#a8a29e' }">
              {{ userTier }}
            </div>
          </div>
        </div>
      </div>
    </aside>

    <!-- MAIN APP WRAPPER -->
    <div class="main-wrapper">
      <!-- TOPBAR -->
      <header class="topbar">
        <div>
          <h1 class="view-title">{{ currentTitle }}</h1>
          <div class="view-subtitle">{{ currentSub }}</div>
        </div>
        <div class="topbar-actions">
          <div v-if="hasData" class="data-badge">
            <span class="pulse-dot"></span>
            <span class="badge-text">{{ currentFileName }} · {{ rowCount }} satır</span>
            <button class="clear-data-btn" @click="clearData" title="Verileri Temizle">
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 6 6 18M6 6l12 12"/></svg>
            </button>
          </div>
          <button class="icon-btn" @click="setView('settings')" title="Ayarlar">
            <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 1 1-2.83 2.83l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-4 0v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 1 1-2.83-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1 0-4h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 1 1 2.83-2.83l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 4 0v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 1 1 2.83 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 0 4h-.09a1.65 1.65 0 0 0-1.51 1z"/>
            </svg>
          </button>
        </div>
      </header>

      <!-- DYNAMIC CONTENT -->
      <main class="content-viewport">
        <div class="view-container">
          <!-- ============ 1. DASHBOARD VIEW ============ -->
          <div v-if="activeView === 'dashboard'" class="dashboard-view animate-fade">
            <!-- Upload Forms (Shown when no data is loaded) -->
            <div v-if="!hasData" class="upload-screen">
              <div class="upload-grid">
                <!-- Current Period GSC Upload -->
                <div 
                  class="dropzone primary-dropzone"
                  @click="triggerCurrentUpload" 
                  @dragover="onDragOver" 
                  @drop="onCurrentDrop"
                >
                  <input 
                    type="file" 
                    ref="currentFileInput" 
                    class="hidden-input" 
                    accept=".csv,.zip" 
                    @change="onCurrentFileChange" 
                  />
                  <div class="upload-icon-box">
                    <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="#ff6b5e" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
                      <polyline points="17 8 12 3 7 8" />
                      <line x1="12" y1="3" x2="12" y2="15" />
                    </svg>
                  </div>
                  <div class="drop-title">Güncel Dönem CSV'si</div>
                  <p class="drop-desc">Google Search Console'dan indirdiğiniz performans dosyasını buraya sürükleyin veya tıklayın</p>
                  <button class="upload-action-btn">Dosya Seç</button>
                </div>

                <!-- Previous Period GSC Upload (Optional) -->
                <div 
                  class="dropzone secondary-dropzone"
                  @click="triggerPreviousUpload" 
                  @dragover="onDragOver" 
                  @drop="onPreviousDrop"
                >
                  <input 
                    type="file" 
                    ref="prevFileInput" 
                    class="hidden-input" 
                    accept=".csv,.zip" 
                    @change="onPreviousFileChange" 
                  />
                  <div class="upload-icon-box grayscale">
                    <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="#7e776c" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M3 3v18h18" />
                      <path d="m19 9-5 5-4-4-3 3" />
                    </svg>
                  </div>
                  <div class="drop-title secondary-title">Geçmiş Dönem CSV'si</div>
                  <p class="drop-desc">İçerik eskimesi ve karşılaştırma analizi yapmak için opsiyonel geçmiş dönem dosyası</p>
                  <button class="upload-action-btn secondary-btn">Opsiyonel Seç</button>
                </div>
              </div>

              <!-- GSC API Quick Connect Card (Shown when connected) -->
              <div v-if="gscConnected" class="gsc-api-quick-card animate-fade">
                <div class="api-card-header">
                  <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#60a5fa" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <circle cx="12" cy="12" r="10" />
                    <line x1="22" y1="22" x2="16.65" y2="16.65" />
                  </svg>
                  <span class="api-card-title">Google Search Console Canlı Bağlantısı</span>
                </div>
                
                <div class="api-card-body-row">
                  <div class="site-select-group">
                    <label class="site-select-label">Mülk (Web Sitesi) Seçin</label>
                    <select v-model="selectedGscSite" @change="saveSelectedGscSite" class="site-dropdown">
                      <option v-for="site in gscSites" :key="site" :value="site">{{ site }}</option>
                      <option v-if="gscSites.length === 0" value="" disabled>Mülk yüklenemedi...</option>
                    </select>
                  </div>
                  
                  <button 
                    class="gsc-load-btn" 
                    @click="loadDataFromGscApi"
                    :disabled="loadingGscData || !selectedGscSite"
                  >
                    <span v-if="loadingGscData" class="spinner-small" style="display: inline-block; width: 12px; height: 12px; border: 2px solid white; border-top-color: transparent; border-radius: 50%; animation: spin 0.8s linear infinite; margin-right: 6px; vertical-align: middle;"></span>
                    {{ loadingGscData ? 'Veriler Çekiliyor...' : 'GSC API\'den Canlı Veri Yükle' }}
                  </button>
                </div>
              </div>

              <!-- Upload Helpers -->
              <div class="upload-help-bar">
                <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="#E1251A" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                  <circle cx="12" cy="12" r="10" />
                  <path d="M12 16v-4" />
                  <path d="M12 8h.01" />
                </svg>
                <span>Verileriniz tamamen yerel bilgisayarınızda işlenir, sunucuya yüklenmez. <span v-if="!gscConnected">Dilerseniz Google Search Console API entegrasyonuyla verileri canlı çekebilirsiniz.</span></span>
              </div>
            </div>

            <!-- Dashboard Content (Shown when GSC data loaded) -->
            <div v-else class="dashboard-content">
              <!-- KPI Cards Grid -->
              <div class="kpi-grid">
                <div v-for="k in kpis" :key="k.label" class="kpi-card">
                  <div class="kpi-top">
                    <span 
                      class="kpi-icon-box"
                      :style="{ 
                        background: k.up ? 'rgba(52,211,153,0.12)' : 'rgba(225,37,26,0.12)',
                        color: k.up ? '#34d399' : '#ff6b5e' 
                      }"
                    >
                      <!-- Render Icons -->
                      <svg v-if="k.icon === 'eye'" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9"><path d="M2 12s3-7 10-7 10 7 10 7-3 7-10 7-10-7-10-7z"/><circle cx="12" cy="12" r="3"/></svg>
                      <svg v-else-if="k.icon === 'click'" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9"><path d="M9 9l5 12 1.8-5.2L21 14z"/><path d="M7.2 2.2 8 5.1"/><path d="m5.1 8-2.9-.8"/><path d="M14 4.1 12 6"/><path d="m6 12-1.9 2"/></svg>
                      <svg v-else-if="k.icon === 'percent'" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9"><line x1="19" y1="5" x2="5" y2="19"/><circle cx="6.5" cy="6.5" r="2.5"/><circle cx="17.5" cy="17.5" r="2.5"/></svg>
                      <svg v-else-if="k.icon === 'rank'" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9"><path d="M3 3v18h18"/><rect x="7" y="10" width="3" height="8"/><rect x="12" y="6" width="3" height="12"/><rect x="17" y="13" width="3" height="5"/></svg>
                    </span>
                    <span 
                      :class="['kpi-trend', { down: !k.up }]"
                      :style="{ 
                        background: k.up ? 'rgba(52,211,153,0.12)' : 'rgba(239,68,68,0.12)',
                        color: k.up ? '#34d399' : '#f87171' 
                      }"
                    >
                      {{ k.trend }}
                    </span>
                  </div>
                  <div class="kpi-value">{{ k.value }}</div>
                  <div class="kpi-label">{{ k.label }}</div>
                </div>
              </div>

              <!-- Main Widgets Section -->
              <div class="dashboard-widgets">
                <!-- Trend Chart -->
                <div class="widget-panel chart-panel">
                  <div class="panel-header">
                    <h3 class="panel-title">Tıklama Eğilimi · Son 28 Gün</h3>
                    <span class="panel-badge">Günlük</span>
                  </div>
                  <div class="bar-chart">
                    <div 
                      v-for="(b, idx) in chartBars" 
                      :key="idx" 
                      class="bar-wrapper"
                    >
                      <div 
                        class="bar" 
                        :style="{ height: b.h, backgroundColor: b.color }"
                        :title="`Gün ${idx+1}: ${b.h}`"
                      ></div>
                    </div>
                  </div>
                </div>

                <!-- Distribution and Call to Action -->
                <div class="widget-panel distribution-panel">
                  <h3 class="panel-title spacer-bottom">Fırsat Dağılımı</h3>
                  <div class="distribution-list">
                    <div v-for="o in oppDist" :key="o.label" class="dist-item">
                      <div class="dist-header">
                        <span class="dist-name">{{ o.label }}</span>
                        <span class="dist-count">{{ o.count }} fırsat</span>
                      </div>
                      <div class="progress-track">
                        <div 
                          class="progress-fill" 
                          :style="{ width: o.pct, backgroundColor: o.color }"
                        ></div>
                      </div>
                    </div>
                  </div>
                  <button class="primary-btn-neon" @click="setView('explorer')">
                    Tüm Fırsatları Gör →
                  </button>
                </div>
              </div>
            </div>
          </div>

          <!-- ============ 2. OPPORTUNITY EXPLORER VIEW ============ -->
          <div v-if="activeView === 'explorer'" class="explorer-view animate-fade">
            <!-- Search & Filtering Bar -->
            <div class="filter-bar">
              <div class="search-input-box">
                <span class="search-icon">
                  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <circle cx="11" cy="11" r="8" />
                    <path d="m21 21-4.3-4.3" />
                  </svg>
                </span>
                <input 
                  type="text" 
                  v-model="search" 
                  placeholder="Sorgu veya sayfa ara..." 
                  class="search-input"
                />
              </div>

              <!-- Quick Filter Tags -->
              <div class="filter-tags">
                <button 
                  v-for="opt in [['all', 'Tümü'], ['Striking Distance', 'Striking Distance'], ['Low CTR', 'Low CTR'], ['Internal Link', 'Internal Link']]"
                  :key="opt[0]"
                  @click="filter = opt[0]"
                  :class="['filter-tag-btn', { active: filter === opt[0] }]"
                >
                  {{ opt[1] }}
                </button>
              </div>
            </div>

            <!-- Source Filter Toggles -->
            <div class="source-filter-row" v-if="hasData">
              <span class="source-filter-label">Kaynak Filtresi:</span>
              <label class="source-checkbox-label">
                <input type="checkbox" v-model="showSourceQueries" />
                <span>Sorgular.csv</span>
              </label>
              <label class="source-checkbox-label">
                <input type="checkbox" v-model="showSourcePages" />
                <span>Sayfalar.csv</span>
              </label>
              <label class="source-checkbox-label">
                <input type="checkbox" v-model="showSourceJoint" />
                <span>Birleşik Veri</span>
              </label>
            </div>

            <!-- Opportunities Datatable -->
            <div class="table-container">
              <div class="table-scroll">
                <table class="datatable">
                  <thead>
                    <tr>
                      <th class="text-left sortable" style="width: 12%" @click="toggleSort('type')">
                        FIRSAT TİPİ
                        <span class="sort-indicator">{{ sortBy === 'type' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-left sortable" style="width: 20%" @click="toggleSort('query')">
                        SORGU
                        <span class="sort-indicator">{{ sortBy === 'query' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-left sortable" style="width: 20%" @click="toggleSort('page')">
                        HEDEF SAYFA
                        <span class="sort-indicator">{{ sortBy === 'page' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-right sortable" style="width: 8%" @click="toggleSort('impr')">
                        GÖSTERİM
                        <span class="sort-indicator">{{ sortBy === 'impr' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-right sortable" style="width: 8%" @click="toggleSort('clicks')">
                        TIKLAMA
                        <span class="sort-indicator">{{ sortBy === 'clicks' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-right sortable" style="width: 6%" @click="toggleSort('ctr')">
                        CTR
                        <span class="sort-indicator">{{ sortBy === 'ctr' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-right sortable" style="width: 8%" @click="toggleSort('gain')">
                        POT. KAZANÇ
                        <span class="sort-indicator">{{ sortBy === 'gain' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-left sortable" style="width: 10%" @click="toggleSort('source')">
                        KAYNAK
                        <span class="sort-indicator">{{ sortBy === 'source' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                      <th class="text-center sortable" style="width: 8%" @click="toggleSort('prio')">
                        ÖNCELİK
                        <span class="sort-indicator">{{ sortBy === 'prio' ? (sortOrder === 'asc' ? ' ▲' : ' ▼') : '' }}</span>
                      </th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr 
                      v-for="(row, idx) in sortedFilteredOpps" 
                      :key="idx" 
                      @click="selectedOpp = row"
                      :class="['table-row', { selected: selectedOpp?.query === row.query && selectedOpp?.page === row.page }]"
                    >
                      <td>
                        <span 
                          :class="['badge-tag', row.type.toLowerCase().replace(' ', '-')]"
                        >
                          {{ row.type }}
                        </span>
                      </td>
                      <td class="query-cell font-semibold">{{ row.query }}</td>
                      <td class="page-cell text-muted">{{ row.page }}</td>
                      <td class="text-right font-medium">{{ row.impr.toLocaleString('tr-TR') }}</td>
                      <td class="text-right font-medium">{{ row.clicks.toLocaleString('tr-TR') }}</td>
                      <td class="text-right font-medium">%{{ row.ctr.toFixed(1) }}</td>
                      <td class="text-right text-success font-bold">+{{ row.gain.toLocaleString('tr-TR') }}</td>
                      <td>
                        <span class="source-tag">{{ row.source }}</span>
                      </td>
                      <td class="text-center">
                        <span :class="['priority-badge', row.prio]">
                          {{ row.prio === 'high' ? 'Yüksek' : (row.prio === 'medium' ? 'Orta' : 'Düşük') }}
                        </span>
                      </td>
                    </tr>
                    <tr v-if="sortedFilteredOpps.length === 0">
                      <td colspan="9" class="text-center empty-state-row">
                        Eşleşen fırsat bulunamadı.
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
              
              <!-- Footer Row Counter -->
              <div class="table-footer">
                <span>{{ filteredOpps.length }} fırsat gösteriliyor</span>
                <span>Öncelik skoruna ve tahmini kazanca göre sıralandı</span>
              </div>
            </div>

            <!-- OPPORTUNITY DETAIL DRAWER -->
            <transition name="slide">
              <div v-if="selectedOpp" class="drawer-overlay">
                <div class="drawer-backdrop" @click="closeDrawer"></div>
                <div class="drawer-panel animate-slide">
                  <!-- Header -->
                  <div class="drawer-header">
                    <div>
                      <span :class="['badge-tag', selectedOpp.type.toLowerCase().replace(' ', '-')]">
                        {{ selectedOpp.type }}
                      </span>
                      <h2 class="drawer-title">{{ selectedOpp.query }}</h2>
                      <div class="drawer-url">{{ selectedOpp.page }} <span class="source-tag" style="margin-left: 8px; vertical-align: middle;">{{ selectedOpp.source }}</span></div>
                    </div>
                    <button class="close-drawer-btn" @click="closeDrawer">
                      <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
                        <path d="M18 6 6 18M6 6l12 12" />
                      </svg>
                    </button>
                  </div>

                  <!-- Drawer Stats Grid -->
                  <div class="drawer-content">
                    <div class="drawer-stats">
                      <div class="mini-stat-card">
                        <div class="stat-num">{{ selectedOpp.impr.toLocaleString('tr-TR') }}</div>
                        <div class="stat-lbl">Gösterim</div>
                      </div>
                      <div class="mini-stat-card">
                        <div class="stat-num">%{{ selectedOpp.ctr.toFixed(1) }}</div>
                        <div class="stat-lbl">Mevcut CTR</div>
                      </div>
                      <div class="mini-stat-card highlight-border">
                        <div class="stat-num text-accent">{{ selectedOpp.pos.toFixed(1) }}</div>
                        <div class="stat-lbl">Pozisyon</div>
                      </div>
                    </div>

                    <!-- Details Section -->
                    <div class="drawer-section">
                      <div class="section-title">
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#E1251A" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                          <circle cx="12" cy="12" r="10" />
                          <path d="M12 16v-4" />
                          <path d="M12 8h.01" />
                        </svg>
                        <span>Neden Önemli?</span>
                      </div>
                      <p class="section-body">{{ selectedOpp.why }}</p>
                    </div>

                    <!-- Actions Section -->
                    <div class="drawer-section">
                      <div class="section-title text-success-title">
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#34d399" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                          <path d="m9 11 3 3L22 4" />
                          <path d="M21 12v7a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11" />
                        </svg>
                        <span>Önerilen Eylemler</span>
                      </div>
                      <div class="action-box">
                        <p class="action-body">{{ selectedOpp.action }}</p>
                      </div>
                    </div>

                    <!-- Potential Traffic gain Banner -->
                    <div class="gain-banner">
                      <div class="gain-lbl">Eğer bu optimizasyon yapılırsa tahmini kazanç:</div>
                      <div class="gain-val">+{{ selectedOpp.gain.toLocaleString('tr-TR') }} tıklama/ay</div>
                    </div>

                    <!-- Ask AI Integration Button -->
                    <button class="drawer-ai-btn" @click="reportWithAi(selectedOpp)">
                      <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M12 3l1.9 5.8a2 2 0 0 0 1.3 1.3L21 12l-5.8 1.9a2 2 0 0 0-1.3 1.3L12 21l-1.9-5.8a2 2 0 0 0-1.3-1.3L3 12l5.8-1.9a2 2 0 0 0 1.3-1.3z" />
                      </svg>
                      Yapay Zeka Asistanı ile Raporla
                    </button>
                  </div>
                </div>
              </div>
            </transition>
          </div>

          <!-- ============ 3. CANNIBALIZATION VIEW ============ -->
          <div v-if="activeView === 'cannibal'" class="cannibal-view animate-fade">
            <div class="cannibal-content">
              <!-- Warning Banner -->
              <div v-if="isDataJoint" class="warning-banner">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#fb923c" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                  <path d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3z" />
                  <path d="M12 9v4" />
                  <path d="M12 17h.01" />
                </svg>
                <div class="warning-text">
                  <div class="warning-title">{{ calculatedCannibal.length }} sorgu için yamyamlık tespit edildi</div>
                  <div class="warning-desc">Aynı anahtar kelime için sitenizde birden fazla URL sıralama alıyor. Bu durum, sayfalarınızın birbiriyle rekabet edip sıralama değerini düşürmesine sebep olur.</div>
                </div>
              </div>
              <div v-else class="warning-banner info-blue">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#60a5fa" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                  <circle cx="12" cy="12" r="10" />
                  <path d="M12 16v-4" />
                  <path d="M12 8h.01" />
                </svg>
                <div class="warning-text">
                  <div class="warning-title" style="color: #93c5fd;">Birleşik Sorgu-Sayfa Verisi Gerekiyor</div>
                  <div class="warning-desc">
                    Yamyamlık (Cannibalization) analizi, her bir arama sorgusunun hangi hedef sayfalarla eşleştiğini bilmeyi gerektirir. 
                    Şu anda yüklediğiniz standart GSC ZIP dosyasında sorgular ve sayfalar ayrı tablolar halindedir. 
                    Bu analizi etkinleştirmek için <strong>Ayarlar'dan GSC API yeteneğini bağlayabilir</strong> veya hem sorguyu hem sayfayı barındıran birleşik bir CSV yükleyebilirsiniz.
                  </div>
                </div>
              </div>

              <!-- Cannibalization Groups -->
              <div class="cannibal-list">
                <div 
                  v-for="(group, idx) in calculatedCannibal" 
                  :key="idx" 
                  class="cannibal-card"
                >
                  <div class="cannibal-card-header">
                    <div class="cannibal-left">
                      <span class="cannibal-count-badge">{{ group.count }} URL</span>
                      <span class="cannibal-query-title">"{{ group.query }}"</span>
                    </div>
                    <span class="cannibal-total-impr">{{ group.totalImpr }} toplam gösterim</span>
                  </div>

                  <!-- Competing Pages Table list -->
                  <div class="cannibal-pages">
                    <div 
                      v-for="(page, pIdx) in group.pages" 
                      :key="pIdx" 
                      class="cannibal-page-row"
                    >
                      <span 
                        class="status-dot" 
                        :style="{ backgroundColor: page.primary ? '#34d399' : '#fb923c' }"
                      ></span>
                      <span class="cannibal-url font-medium">{{ page.url }}</span>
                      <span class="cannibal-pos text-muted">Pozisyon: {{ page.pos }}</span>
                      <span class="cannibal-clicks text-muted">{{ page.clicks }} tık</span>
                      <span 
                        :class="['cannibal-tag', { primary: page.primary }]"
                      >
                        {{ page.tag }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- ============ 4. CONTENT DECAY VIEW ============ -->
          <div v-if="activeView === 'decay'" class="decay-view animate-fade">
            <div class="decay-content">
              <!-- Grid Summary info -->
              <div class="decay-stats-grid">
                <div class="decay-stat-card red-bg">
                  <div class="stat-title-num">{{ decayAvgDecline }}</div>
                  <div class="stat-desc">Düşüşteki Sorgu/Sayfalar (Ort.)</div>
                </div>
                <div class="decay-stat-card green-bg">
                  <div class="stat-title-num">{{ decayAvgGrowth }}</div>
                  <div class="stat-desc">Yükselişteki Sorgu/Sayfalar (Ort.)</div>
                </div>
                <div class="decay-stat-card dark-bg">
                  <div class="stat-title-num">{{ calculatedDecay.filter(d => !d.up).length }}</div>
                  <div class="stat-desc">Müdahale Gereken Sorgu</div>
                </div>
              </div>

              <!-- Content Decay table -->
              <div class="table-container">
                <div class="table-scroll">
                  <table class="datatable">
                    <thead>
                      <tr>
                        <th class="text-left" style="width: 45%">SORGU / SAYFA</th>
                        <th class="text-right" style="width: 15%">GEÇMİŞ TIKLAMA</th>
                        <th class="text-right" style="width: 15%">GÜNCEL TIKLAMA</th>
                        <th class="text-right" style="width: 12%">DEĞİŞİM</th>
                        <th class="text-right" style="width: 13%">POZİSYON FARKI</th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr 
                        v-for="(row, idx) in calculatedDecay" 
                        :key="idx" 
                        class="table-row"
                      >
                        <td style="padding: 13px 18px">
                          <div class="query-cell font-semibold">{{ row.query }}</div>
                          <div class="page-cell text-muted size-small">{{ row.page }}</div>
                        </td>
                        <td class="text-right text-muted font-medium">{{ row.prev.toLocaleString('tr-TR') }}</td>
                        <td class="text-right font-medium">{{ row.curr.toLocaleString('tr-TR') }}</td>
                        <td class="text-right">
                          <span 
                            class="change-tag font-bold"
                            :style="{ 
                              background: row.curr >= row.prev ? 'rgba(52,211,153,0.13)' : 'rgba(239,68,68,0.13)',
                              color: row.curr >= row.prev ? '#34d399' : '#f87171'
                            }"
                          >
                            {{ row.change }}
                          </span>
                        </td>
                        <td 
                          class="text-right font-semibold"
                          :style="{ color: row.up ? '#34d399' : '#f87171' }"
                        >
                          {{ row.posDiff }}
                        </td>
                      </tr>
                      <tr v-if="calculatedDecay.length === 0">
                        <td colspan="5" class="text-center empty-state-row">
                          Lütfen analiz paneline hem Güncel hem de Geçmiş dönem CSV'sini yükleyin.
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          </div>

          <!-- ============ 5. AI WRITER / ASSISTANT VIEW ============ -->
          <div v-if="activeView === 'ai'" class="ai-view animate-fade">
            <!-- AI Disabled Alert -->
            <div v-if="!geminiOn" class="ai-disabled-box">
              <div class="disabled-inner">
                <div class="spark-icon-box">
                  <svg width="30" height="30" viewBox="0 0 24 24" fill="none" stroke="#ff6b5e" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M12 3l1.9 5.8a2 2 0 0 0 1.3 1.3L21 12l-5.8 1.9a2 2 0 0 0-1.3 1.3L12 21l-1.9-5.8a2 2 0 0 0-1.3-1.3L3 12l5.8-1.9a2 2 0 0 0 1.3-1.3z" />
                  </svg>
                </div>
                <h3 class="disabled-title">Yapay Zeka Asistanı Pasif</h3>
                <p class="disabled-desc">Gemini ile entegre akıllı SEO fırsat analizleri ve aksiyon raporları oluşturmak için Ayarlar ekranından Gemini API'yi etkinleştirin.</p>
                <button class="primary-btn-neon inline-fit" @click="setView('settings')">Ayarlar'a Git</button>
              </div>
            </div>

            <!-- Active AI Chat Interface -->
            <div v-else class="ai-chat-interface">
              <!-- Chat Header -->
              <div class="chat-header-row" style="display: flex; justify-content: space-between; align-items: center; padding: 12px 18px; border-bottom: 1px solid var(--border-color); background: rgba(0, 0, 0, 0.15);">
                <div style="display: flex; align-items: center; gap: 8px;">
                  <span class="pulse-dot"></span>
                  <span class="chat-header-title" style="font-weight: 600; font-size: 13.5px; color: var(--text-primary);">SEO Yapay Zeka Asistanı</span>
                </div>
                <button class="clear-chat-btn" @click="clearChatHistory" style="background: rgba(225, 37, 26, 0.12); color: #ff6b5e; border: 1px solid rgba(225, 37, 26, 0.25); padding: 4px 10px; border-radius: 6px; font-size: 11px; font-weight: 600; cursor: pointer; transition: background 0.2s, border-color 0.2s;">
                  Sohbeti Temizle
                </button>
              </div>

              <div ref="chatContainer" class="chat-messages-container" style="flex: 1; overflow-y: auto; padding: 18px; display: flex; flex-direction: column; gap: 16px;">
                <div 
                  v-for="(msg, idx) in chat" 
                  :key="idx" 
                  :class="['chat-row', { user: msg.role === 'user' }]"
                >
                  <div v-if="msg.role === 'ai'" class="ai-avatar-badge">
                    <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M12 3l1.9 5.8a2 2 0 0 0 1.3 1.3L21 12l-5.8 1.9a2 2 0 0 0-1.3 1.3L12 21l-1.9-5.8a2 2 0 0 0-1.3-1.3L3 12l5.8-1.9a2 2 0 0 0 1.3-1.3z" />
                    </svg>
                  </div>
                  <div class="chat-bubble">
                    <!-- Render Markdown parsed HTML for AI replies, simple text for User -->
                    <div v-if="msg.role === 'ai'" class="markdown-body" v-html="renderMarkdown(msg.text)"></div>
                    <p v-else style="white-space: pre-wrap; margin: 0;">{{ msg.text }}</p>
                  </div>
                </div>
              </div>

              <!-- Prompt Input area -->
              <div class="chat-input-container">
                <div class="chat-input-wrapper">
                  <input 
                    type="text" 
                    v-model="prompt" 
                    placeholder="Sorgu performansları veya fırsat detayları hakkında soru sorun..." 
                    class="chat-input"
                    @keydown.enter="sendPrompt"
                  />
                  <button class="send-prompt-btn" @click="sendPrompt">
                    <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="m22 2-7 20-4-9-9-4z" />
                      <path d="M22 2 11 13" />
                    </svg>
                  </button>
                </div>
                <div class="model-info-footer">
                  Aktif Model: {{ model === 'pro' ? 'Gemini 2.5 Pro' : (model === 'flash2' ? 'Gemini 2.0 Flash' : 'Gemini 2.5 Flash') }}
                </div>
              </div>
            </div>
          </div>

          <!-- ============ 6. SETTINGS VIEW ============ -->
          <div v-if="activeView === 'settings'" class="settings-view animate-fade">
            <div class="settings-content">
              <!-- Section A: Analysis Thresholds -->
              <div class="settings-card">
                <h3 class="card-section-title">Dosya Analiz Parametreleri</h3>
                <p class="card-section-desc">Fırsat tespit motorunun tetikleneceği kriter sınırlarını belirleyin</p>

                <!-- Impressions slider -->
                <div class="settings-control spacer-bottom">
                  <div class="control-header">
                    <span class="control-label">Minimum Gösterim Filtresi</span>
                    <span class="control-val">{{ minImpr }} gösterim</span>
                  </div>
                  <input 
                    type="range" 
                    min="0" 
                    max="1000" 
                    step="50" 
                    v-model.number="minImpr" 
                    @change="saveThresholds"
                    class="accent-slider"
                  />
                  <div class="slider-boundaries"><span>0</span><span>1000</span></div>
                </div>

                <!-- Position boundaries double slider emulation -->
                <div class="settings-control">
                  <div class="control-header">
                    <span class="control-label">Striking-Distance Pozisyon Sınırları</span>
                    <span class="control-val">{{ sdLow }} ile {{ sdHigh }} arası</span>
                  </div>
                  <div class="flex-control-row">
                    <div class="half-control">
                      <div class="input-lbl">Alt sınır</div>
                      <input 
                        type="range" 
                        min="1" 
                        max="10" 
                        step="1" 
                        v-model.number="sdLow" 
                        @change="saveThresholds"
                        class="accent-slider"
                      />
                    </div>
                    <div class="half-control">
                      <div class="input-lbl">Üst sınır</div>
                      <input 
                        type="range" 
                        min="10" 
                        max="30" 
                        step="1" 
                        v-model.number="sdHigh" 
                        @change="saveThresholds"
                        class="accent-slider"
                      />
                    </div>
                  </div>
                </div>
              </div>

              <!-- Section B: API Integrations -->
              <div class="settings-card">
                <h3 class="card-section-title">API Entegrasyonları</h3>
                <p class="card-section-desc">Opsiyonel harici servis bağlantılarını yönetin</p>

                    <!-- Google Search Console Connection -->
                <div class="api-control-row border-bottom flex-column items-stretch">
                  <div class="flex-row-between">
                    <div class="api-identity">
                      <div class="api-icon blue-bg">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#60a5fa" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                          <circle cx="11" cy="11" r="8" />
                          <path d="m21 21-4.3-4.3" />
                        </svg>
                      </div>
                      <div>
                        <div class="api-name">Google Search Console API</div>
                        <div 
                          class="api-status font-medium"
                          :style="{ color: gscConnected ? '#34d399' : '#7e776c' }"
                        >
                          {{ gscConnected ? 'Bağlı · ' + gscSites.length + ' mülk listelendi' : 'Bağlı Değil' }}
                        </div>
                      </div>
                    </div>
                    <button 
                      v-if="gscConnected"
                      class="api-action-btn disconnect"
                      @click="handleGscDisconnect"
                    >
                      Bağlantıyı Kes
                    </button>
                  </div>

                  <!-- GSC Subpanel for token and login -->
                  <div v-if="!gscConnected" class="gemini-sub-config animate-fade" style="margin-top: 12px; border-top: 1px solid var(--border-color); padding-top: 12px; display: flex; flex-direction: column; gap: 14px;">
                    <!-- Option A: Client ID + OAuth Button -->
                    <div class="sub-input-group" style="border-bottom: 1px dashed rgba(255,255,255,0.06); padding-bottom: 12px;">
                      <div class="sub-input-label">1. Yöntem: Google OAuth ile Hızlı Giriş</div>
                      <div style="margin-bottom: 8px;">
                        <span class="sub-input-desc" style="display: block; font-size: 11px; margin-bottom: 4px; color: var(--text-muted);">OAuth Client ID (Özel Client ID kullanmak istiyorsanız değiştirin)</span>
                        <input 
                          type="text" 
                          v-model="gscClientId"
                          placeholder="Google OAuth Client ID girin..." 
                          class="settings-input"
                          style="width: 100%; font-size: 11px;"
                        />
                      </div>
                      <button class="gsc-oauth-btn btn-primary" @click="startGscOAuth" style="background: #E1251A; color: white; border: none; width: 100%; height: 36px; border-radius: 8px; font-weight: 600; font-size: 12px; cursor: pointer; transition: background 0.2s;">
                        Tarayıcıda Google ile Giriş Yap
                      </button>
                    </div>

                    <!-- Option B: Paste Token Directly + Connect Button -->
                    <div class="sub-input-group">
                      <div class="sub-input-label">2. Yöntem: Google OAuth Access Token Manuel Yapıştır</div>
                      <div class="gsc-token-row" style="display: flex; gap: 8px;">
                        <input 
                          type="password" 
                          :value="gscAccessToken"
                          @input="onGscTokenInput"
                          placeholder="Google Access Token yapıştırın (Bearer/ya da Google Playground tokenı)..." 
                          class="settings-input"
                          style="flex: 1;"
                        />
                        <button class="gsc-oauth-btn" @click="connectWithToken" style="background: #34d399; color: black; border: none; padding: 0 16px; border-radius: 8px; font-weight: 600; font-size: 12px; cursor: pointer; transition: background 0.2s;">
                          Doğrula & Bağlan
                        </button>
                      </div>
                      <div class="sub-input-desc" style="font-size: 11px; margin-top: 5px; color: var(--text-muted); line-height: 1.5;">
                        <span style="color: #fb923c; font-weight: 600; display: block; margin-bottom: 4px;">💡 Kolay Bağlantı Kılavuzu (30 Saniye):</span>
                        1. Aşağıdaki butona tıklayarak tarayıcınızda <b>Google OAuth Playground</b>'ı açın.<br />
                        2. Sol listenin en altındaki <b>"Input your own scopes"</b> kutusuna <code>https://www.googleapis.com/auth/webmasters.readonly</code> yapıştırın ve <b>Authorize APIs</b> butonuna tıklayın.<br />
                        3. Google hesabınızla giriş yapıp izin verdikten sonra Step 2 alanındaki <b>Exchange authorization code for tokens</b> butonuna tıklayın.<br />
                        4. Oluşan <code>access_token</code> değerini kopyalayıp yukarıdaki kutuya yapıştırın ve <b>Doğrula & Bağlan</b> deyin.<br />
                        <span style="color: #ef4444; font-weight: 600; display: block; margin-top: 6px;">⚠️ Not: Google geçici Access Token jetonları tam 1 saat (60 dakika) geçerlidir. Süre dolunca hata alırsanız yukarıdaki adımlarla yeni bir token kopyalamanız gerekir.</span>
                        <button class="gsc-oauth-btn" @click="openBrowser('https://developers.google.com/oauthplayground')" style="background: rgba(255,255,255,0.06); border: 1px solid var(--border-color); color: #60a5fa; padding: 0 10px; border-radius: 6px; font-weight: 500; font-size: 11px; cursor: pointer; height: 26px; margin-top: 8px;">
                          Tarayıcıda Google OAuth Playground'ı Aç
                        </button>
                      </div>
                    </div>
                  </div>

                  <!-- Selected property display when connected -->
                  <div v-if="gscConnected && gscSites !== null" class="gemini-sub-config animate-fade" style="margin-top: 12px; border-top: 1px solid var(--border-color); padding-top: 12px;">
                    <div class="sub-input-group">
                      <div class="sub-input-label">Aktif Arama Konsolu Mülkü</div>
                      <select v-model="selectedGscSite" @change="saveSelectedGscSite" class="settings-select" style="width: 100%; height: 38px; border-radius: 8px; background: rgba(0,0,0,0.25); border: 1px solid var(--border-color); color: var(--text-primary); padding: 0 10px; font-size: 12px; outline: none; cursor: pointer;">
                        <option v-for="site in gscSites" :key="site" :value="site">{{ site }}</option>
                        <option v-if="gscSites.length === 0" value="" disabled>Mülk yüklenemedi...</option>
                      </select>
                    </div>
                  </div>
                </div>

                <!-- Gemini API Key Connection -->
                <div class="api-control-row flex-column items-stretch">
                  <div class="flex-row-between">
                    <div class="api-identity">
                      <div class="api-icon red-bg">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#ff6b5e" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                          <path d="M12 3l1.9 5.8a2 2 0 0 0 1.3 1.3L21 12l-5.8 1.9a2 2 0 0 0-1.3 1.3L12 21l-1.9-5.8a2 2 0 0 0-1.3-1.3L3 12l5.8-1.9a2 2 0 0 0 1.3-1.3z" />
                        </svg>
                      </div>
                      <div>
                        <div class="api-name">Gemini API Yeteneği</div>
                        <div 
                          class="api-status font-medium"
                          :style="{ color: geminiOn ? '#34d399' : '#7e776c' }"
                        >
                          {{ geminiOn ? 'Etkin' : 'Devre Dışı' }}
                        </div>
                      </div>
                    </div>
                    <!-- Custom Switch Toggle button -->
                    <button 
                      :class="['switch-toggle', { active: geminiOn }]" 
                      @click="geminiOn = !geminiOn"
                    >
                      <span class="switch-knob"></span>
                    </button>
                  </div>

                  <!-- Gemini configurations subpanel -->
                  <div v-if="geminiOn" class="gemini-sub-config animate-fade">
                    <div class="sub-input-group">
                      <div class="sub-input-label">Gemini API Anahtarı</div>
                      <input 
                        type="password" 
                        :value="apiKey"
                        @input="onApiKeyInput"
                        placeholder="API Anahtarı girin..." 
                        class="settings-input"
                      />
                    </div>
                    <div class="sub-input-group">
                      <div class="sub-input-label">Aktif Analiz Modeli</div>
                      <div class="models-row-buttons">
                        <button 
                          v-for="m in [['flash', 'Gemini 2.5 Flash'], ['pro', 'Gemini 2.5 Pro'], ['flash2', 'Gemini 2.0 Flash']]"
                          :key="m[0]"
                          @click="model = m[0]"
                          :class="['model-btn', { active: model === m[0] }]"
                        >
                          {{ m[1] }}
                        </button>
                      </div>
                    </div>
                  </div>
                </div>
              </div>



            </div>
          </div>
        </div>
      </main>
    </div>

    <!-- AUTO-UPDATER MODAL -->
    <transition name="fade">
      <div v-if="showUpdateModal && latestReleaseData" class="update-modal-overlay" style="position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0, 0, 0, 0.75); display: flex; align-items: center; justify-content: center; z-index: 99999; padding: 20px; backdrop-filter: blur(4px);">
        <div class="update-modal-panel animate-zoom" style="background: #2A2927; border: 1px solid rgba(225, 37, 26, 0.35); border-radius: 16px; max-width: 520px; width: 100%; box-shadow: 0 10px 40px rgba(0, 0, 0, 0.7); overflow: hidden; display: flex; flex-direction: column;">
          <!-- Header -->
          <div style="background: rgba(225, 37, 26, 0.08); padding: 18px 24px; border-bottom: 1px solid rgba(225, 37, 26, 0.15); display: flex; align-items: center; gap: 12px;">
            <div style="background: #E1251A; width: 34px; height: 34px; border-radius: 8px; display: flex; align-items: center; justify-content: center; color: white;">
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
            </div>
            <div>
              <h3 style="margin: 0; font-size: 15px; font-weight: 700; color: white;">Yeni Sürüm Mevcut!</h3>
              <span style="font-size: 11px; color: #ff6b5e; font-weight: 600;">Sürüm {{ latestReleaseData.version }}</span>
            </div>
          </div>
          <!-- Body -->
          <div style="padding: 24px; max-height: 280px; overflow-y: auto; color: var(--text-secondary); line-height: 1.6; font-size: 13px;">
            <p style="margin-top: 0; font-weight: 600; color: var(--text-primary); margin-bottom: 12px;">QueryGSC için yeni bir güncelleme yayınlandı. Değişiklik ayrıntıları:</p>
            <div class="markdown-body" v-html="renderMarkdown(latestReleaseData.body || 'Yenilikler belirtilmemiş.')"></div>
          </div>
          <!-- Footer -->
          <div style="padding: 16px 24px; background: rgba(0,0,0,0.15); border-top: 1px solid var(--border-color); display: flex; justify-content: flex-end; gap: 12px;">
            <button @click="showUpdateModal = false" style="background: transparent; border: 1px solid var(--border-color); color: var(--text-secondary); padding: 8px 16px; border-radius: 8px; font-size: 12.5px; font-weight: 600; cursor: pointer; transition: background 0.2s;">
              Daha Sonra
            </button>
            <button @click="triggerUpdateInstall" style="background: #E1251A; color: white; border: none; padding: 8px 20px; border-radius: 8px; font-size: 12.5px; font-weight: 600; cursor: pointer; transition: background 0.2s;">
              Şimdi Güncelle
            </button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style>
/* CSS VARIABLES & RESET */
:root {
  --bg-primary: #1e1c1b;
  --bg-secondary: #2A2927;
  --bg-sidebar: #151413;
  --border-color: rgba(90, 86, 82, 0.4);
  --border-hover: rgba(90, 86, 82, 0.7);
  --accent-red: #E1251A;
  --accent-hover: #ff6b5e;
  --accent-light: #ffb3ab;
  --text-primary: #f7f4f0;
  --text-secondary: #cdc7bd;
  --text-muted: #7e776c;
  --color-green: #34d399;
  --color-orange: #fb923c;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  margin: 0;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  background-color: var(--bg-primary);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* APP LAYOUT STRUCTURE */
.root-layout {
  display: flex;
  width: 100vw;
  height: 100vh;
  background: radial-gradient(1200px 600px at 20% -10%, rgba(225, 37, 26, 0.08), transparent), var(--bg-primary);
  overflow: hidden;
}

/* SIDEBAR STYLES */
.sidebar {
  width: 260px;
  flex: 0 0 260px;
  height: 100%;
  display: flex;
  flex-direction: column;
  background-color: var(--bg-sidebar);
  border-right: 1px solid var(--border-color);
}

.identity {
  height: 60px;
  flex: 0 0 60px;
  display: flex;
  align-items: center;
  gap: 11px;
  padding: 0 20px;
  border-bottom: 1px solid var(--border-color);
}

.logo-box {
  width: 32px;
  height: 32px;
  border-radius: 9px;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 14px rgba(225, 37, 26, 0.35);
}

.app-name {
  font-family: 'Outfit', sans-serif;
  font-weight: 700;
  font-size: 17px;
  color: var(--text-primary);
  letter-spacing: -0.3px;
}

.app-sub {
  font-size: 10px;
  color: var(--text-muted);
  font-weight: 500;
  letter-spacing: 0.3px;
}

.nav-menu {
  flex: 1;
  overflow-y: auto;
  padding: 14px 12px;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.nav-section-title {
  font-size: 10px;
  font-weight: 600;
  color: #5d564c;
  letter-spacing: 1px;
  padding: 8px 12px 6px;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  padding: 10px 12px;
  border-radius: 10px;
  border: none;
  background: transparent;
  color: var(--text-secondary);
  font-size: 13px;
  font-weight: 500;
  text-align: left;
  cursor: pointer;
  transition: all 0.18s;
}

.nav-item:hover {
  background: rgba(255, 255, 255, 0.04);
  color: var(--text-primary);
}

.nav-item.active {
  background: rgba(225, 37, 26, 0.12);
  color: var(--text-primary);
  font-weight: 600;
}

.nav-icon {
  display: flex;
  width: 19px;
  height: 19px;
  color: var(--text-muted);
}

.nav-item.active .nav-icon {
  color: var(--accent-light) !important;
}

.nav-badge {
  font-size: 9px;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 6px;
  background: rgba(225, 37, 26, 0.15);
  color: var(--accent-hover);
  letter-spacing: 0.5px;
}

.user-box {
  padding: 14px 12px;
  border-top: 1px solid var(--border-color);
}

.account-card {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 11px;
  border-radius: 11px;
  background: rgba(225, 37, 26, 0.05);
  border: 1px solid rgba(225, 37, 26, 0.15);
}

.avatar {
  width: 34px;
  height: 34px;
  border-radius: 9px;
  background: linear-gradient(135deg, #b01810, var(--accent-red));
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 13px;
  color: #fff;
}

.account-details {
  flex: 1;
  min-width: 0;
}

.username {
  font-size: 12.5px;
  font-weight: 600;
  color: var(--text-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.plan-label {
  font-size: 10.5px;
  color: var(--accent-hover);
  font-weight: 500;
}

/* MAIN CONTENT WRAPPER */
.main-wrapper {
  flex: 1;
  height: 100%;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.topbar {
  height: 60px;
  flex: 0 0 60px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 26px;
  border-bottom: 1px solid var(--border-color);
  background: rgba(26, 24, 23, 0.6);
  backdrop-filter: blur(14px);
  z-index: 10;
}

.view-title {
  font-family: 'Outfit', sans-serif;
  font-size: 18px;
  font-weight: 600;
  color: var(--text-primary);
  letter-spacing: -0.3px;
}

.view-subtitle {
  font-size: 11.5px;
  color: var(--text-muted);
  margin-top: 1px;
}

.topbar-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.data-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 5px 12px;
  border-radius: 9px;
  background: rgba(52, 211, 153, 0.1);
  border: 1px solid rgba(52, 211, 153, 0.2);
}

.pulse-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--color-green);
  box-shadow: 0 0 8px var(--color-green);
  animation: pulse 2s infinite;
}

.badge-text {
  font-size: 11.5px;
  font-weight: 500;
  color: #6ee7b7;
}

.clear-data-btn {
  background: transparent;
  border: none;
  color: #6ee7b7;
  cursor: pointer;
  display: flex;
  align-items: center;
  padding: 2px;
  border-radius: 4px;
  margin-left: 2px;
}

.clear-data-btn:hover {
  background: rgba(52, 211, 153, 0.2);
}

.icon-btn {
  width: 36px;
  height: 36px;
  border-radius: 9px;
  border: 1px solid var(--border-color);
  background: rgba(62, 60, 57, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #a8a199;
  cursor: pointer;
  transition: all 0.18s;
}

.icon-btn:hover {
  border-color: var(--border-hover);
  color: var(--text-primary);
  background: rgba(62, 60, 57, 0.9);
}

.content-viewport {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  position: relative;
  background-color: var(--bg-primary);
}

.view-container {
  min-height: 100%;
}

/* ============ 1. DASHBOARD VIEW STYLES ============ */
.dashboard-view {
  padding: 30px 34px;
  max-width: 1180px;
  margin: 0 auto;
}

.upload-screen {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.upload-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.dropzone {
  border-radius: 16px;
  height: 240px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.22s;
  padding: 24px;
}

.primary-dropzone {
  background: rgba(225, 37, 26, 0.04);
  border: 2px dashed rgba(225, 37, 26, 0.25);
}

.primary-dropzone:hover {
  border-color: var(--accent-red);
  background: rgba(225, 37, 26, 0.07);
}

.secondary-dropzone {
  background: rgba(62, 60, 57, 0.3);
  border: 2px dashed var(--border-color);
}

.secondary-dropzone:hover {
  border-color: var(--border-hover);
  background: rgba(62, 60, 57, 0.4);
}

.hidden-input {
  display: none;
}

.upload-icon-box {
  width: 56px;
  height: 56px;
  border-radius: 15px;
  background: rgba(225, 37, 26, 0.12);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 16px;
  border: 1px solid rgba(225, 37, 26, 0.25);
}

.upload-icon-box.grayscale {
  background: rgba(90, 86, 82, 0.3);
  border-color: var(--border-color);
}

.drop-title {
  font-size: 15.5px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 6px;
}

.drop-title.secondary-title {
  color: var(--text-secondary);
}

.drop-desc {
  font-size: 12.5px;
  color: var(--text-muted);
  line-height: 1.5;
  text-align: center;
  max-width: 250px;
  margin-bottom: 16px;
}

.upload-action-btn {
  font-size: 11.5px;
  font-weight: 600;
  color: #fff;
  background-color: var(--accent-red);
  padding: 8px 18px;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 10px rgba(225, 37, 26, 0.3);
  transition: all 0.18s;
}

.upload-action-btn:hover {
  background-color: #b01810;
  box-shadow: 0 6px 14px rgba(225, 37, 26, 0.4);
}

.upload-action-btn.secondary-btn {
  background-color: rgba(90, 86, 82, 0.4);
  color: var(--text-secondary);
  border: 1px solid var(--border-color);
  box-shadow: none;
}

.upload-action-btn.secondary-btn:hover {
  background-color: rgba(90, 86, 82, 0.6);
  color: var(--text-primary);
}

.upload-help-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 18px;
  border-radius: 12px;
  background: rgba(62, 60, 57, 0.4);
  border: 1px solid var(--border-color);
  font-size: 12.5px;
  color: var(--text-secondary);
}

/* GSC API Quick Connect Card Styles */
.gsc-api-quick-card {
  background: rgba(42, 41, 39, 0.95);
  border: 1px solid rgba(96, 165, 250, 0.25);
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 24px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.api-card-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 16px;
  color: #60a5fa;
  font-weight: 600;
  font-size: 14px;
}

.api-card-title {
  color: var(--text-primary);
}

.api-card-body-row {
  display: flex;
  align-items: flex-end;
  gap: 16px;
}

.site-select-group {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.site-select-label {
  font-size: 11.5px;
  color: var(--text-muted);
  font-weight: 500;
}

.site-dropdown {
  width: 100%;
  height: 38px;
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.25);
  border: 1px solid var(--border-color);
  color: var(--text-primary);
  padding: 0 12px;
  font-size: 13px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.2s;
}

.site-dropdown:focus {
  border-color: #60a5fa;
}

.gsc-load-btn {
  height: 38px;
  padding: 0 20px;
  background: #E1251A;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 13px;
  cursor: pointer;
  transition: opacity 0.2s, background 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.gsc-load-btn:hover:not(:disabled) {
  background: #ff3b30;
}

.gsc-load-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.gsc-oauth-btn {
  height: 38px;
  padding: 0 14px;
  background: #E1251A;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 12px;
  cursor: pointer;
  transition: background 0.2s;
}

.gsc-oauth-btn:hover {
  background: #ff3b30;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.upload-help-bar a {
  color: var(--accent-hover);
  text-decoration: underline;
  font-weight: 600;
}

.upload-help-bar a:hover {
  color: var(--accent-light);
}

/* KPI CARDS STYLES */
.kpi-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 18px;
  margin-bottom: 26px;
}

.kpi-card {
  padding: 20px;
  border-radius: 16px;
  background: rgba(62, 60, 57, 0.3);
  border: 1px solid var(--border-color);
}

.kpi-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
}

.kpi-icon-box {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.kpi-trend {
  font-size: 11px;
  font-weight: 600;
  padding: 3px 8px;
  border-radius: 7px;
}

.kpi-value {
  font-family: 'Outfit', sans-serif;
  font-size: 26px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.5px;
}

.kpi-label {
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 3px;
}

/* WIDGETS */
.dashboard-widgets {
  display: grid;
  grid-template-columns: 1.6fr 1fr;
  gap: 18px;
}

.widget-panel {
  padding: 22px;
  border-radius: 16px;
  background: rgba(62, 60, 57, 0.3);
  border: 1px solid var(--border-color);
}

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}

.panel-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-secondary);
}

.panel-title.spacer-bottom {
  margin-bottom: 18px;
}

.panel-badge {
  font-size: 11px;
  color: var(--text-muted);
  font-weight: 500;
  padding: 4px 9px;
  border-radius: 7px;
  border: 1px solid var(--border-color);
}

.bar-chart {
  height: 190px;
  display: flex;
  align-items: flex-end;
  gap: 5px;
}

.bar-wrapper {
  flex: 1;
  height: 100%;
  display: flex;
  align-items: flex-end;
}

.bar {
  width: 100%;
  border-radius: 4px 4px 2px 2px;
  transition: all 0.3s ease;
  cursor: pointer;
}

.bar:hover {
  opacity: 0.8;
  transform: scaleY(1.05);
}

.distribution-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-bottom: 22px;
}

.dist-item {
  display: flex;
  flex-direction: column;
}

.dist-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 6px;
}

.dist-name {
  font-size: 12.5px;
  color: var(--text-secondary);
  font-weight: 500;
}

.dist-count {
  font-size: 12.5px;
  color: var(--text-muted);
  font-weight: 600;
}

.progress-track {
  height: 7px;
  border-radius: 4px;
  background: rgba(90, 86, 82, 0.3);
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: 4px;
}

.primary-btn-neon {
  width: 100%;
  padding: 11px;
  border-radius: 10px;
  border: none;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  color: #fff;
  font-size: 12.5px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 4px 14px rgba(225, 37, 26, 0.3);
  transition: all 0.18s;
}

.primary-btn-neon:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 18px rgba(225, 37, 26, 0.45);
}

.primary-btn-neon:active {
  transform: translateY(0);
}

.primary-btn-neon.inline-fit {
  width: auto;
  padding: 11px 24px;
}

/* ============ 2. EXPLORER VIEW STYLES ============ */
.explorer-view {
  padding: 24px 34px;
  display: flex;
  flex-direction: column;
  height: 100%;
}

.filter-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 20px;
}

.source-filter-row {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-top: -10px;
  margin-bottom: 20px;
  padding: 8px 16px;
  background: rgba(255, 255, 255, 0.02);
  border-radius: 8px;
  border: 1px solid var(--border-color);
  font-size: 12px;
}

.source-filter-label {
  color: var(--text-muted);
  font-weight: 600;
}

.source-checkbox-label {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  user-select: none;
  color: var(--text-secondary);
}

.source-checkbox-label input {
  accent-color: #E1251A;
  cursor: pointer;
}

.source-tag {
  font-size: 10.5px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid var(--border-color);
  padding: 2px 6px;
  border-radius: 4px;
  color: var(--text-secondary);
  display: inline-block;
}

.datatable th.sortable {
  cursor: pointer;
  user-select: none;
  transition: background 0.15s, color 0.15s;
}

.datatable th.sortable:hover {
  color: var(--text-primary) !important;
  background: rgba(255, 255, 255, 0.03);
}

.sort-indicator {
  font-size: 9px;
  color: #E1251A;
  margin-left: 2px;
}

.search-input-box {
  flex: 1;
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 13px;
  display: flex;
  color: var(--text-muted);
}

.search-input {
  width: 100%;
  padding: 11px 13px 11px 38px;
  border-radius: 11px;
  background: rgba(62, 60, 57, 0.3);
  border: 1px solid var(--border-color);
  color: var(--text-primary);
  font-size: 13px;
  transition: border-color 0.18s;
}

.search-input:focus {
  border-color: var(--border-hover);
}

.filter-tags {
  display: flex;
  gap: 7px;
}

.filter-tag-btn {
  padding: 9px 14px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.15s;
  border: 1px solid var(--border-color);
  background: rgba(62, 60, 57, 0.3);
  color: var(--text-secondary);
}

.filter-tag-btn:hover {
  border-color: var(--border-hover);
  color: var(--text-primary);
}

.filter-tag-btn.active {
  border-color: rgba(225, 37, 26, 0.4);
  background: rgba(225, 37, 26, 0.12);
  color: var(--accent-light);
  font-weight: 600;
}

/* DATATABLE STYLES */
.table-container {
  border-radius: 16px;
  background: rgba(62, 60, 57, 0.2);
  border: 1px solid var(--border-color);
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.table-scroll {
  overflow-x: auto;
  max-height: calc(100vh - 250px);
}

.datatable {
  width: 100%;
  border-collapse: collapse;
  min-width: 960px;
}

.datatable th {
  padding: 12px 16px;
  font-size: 10.5px;
  font-weight: 600;
  color: var(--text-muted);
  letter-spacing: 0.5px;
  border-bottom: 1px solid var(--border-color);
  white-space: nowrap;
  background-color: rgba(31, 29, 27, 0.5);
  position: sticky;
  top: 0;
  z-index: 2;
}

.table-row {
  border-top: 1px solid rgba(90, 86, 82, 0.25);
  cursor: pointer;
  transition: background-color 0.15s;
}

.table-row:hover {
  background-color: rgba(255, 255, 255, 0.02);
}

.table-row.selected {
  background-color: rgba(225, 37, 26, 0.05);
}

.datatable td {
  padding: 13px 16px;
  font-size: 13px;
  vertical-align: middle;
}

.query-cell {
  color: var(--text-primary);
  max-width: 220px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.page-cell {
  max-width: 220px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 12px;
}

.page-cell.size-small {
  font-size: 11px;
}

.text-left { text-align: left; }
.text-right { text-align: right; }
.text-center { text-align: center; }

.font-semibold { font-weight: 600; }
.font-medium { font-weight: 500; }
.font-bold { font-weight: 700; }

.text-success { color: var(--color-green); }
.text-muted { color: var(--text-muted); }

/* BADGES */
.badge-tag {
  font-size: 10.5px;
  font-weight: 600;
  padding: 4px 9px;
  border-radius: 7px;
  white-space: nowrap;
  display: inline-block;
}

.badge-tag.striking-distance {
  background: rgba(225, 37, 26, 0.15);
  color: var(--accent-light);
}

.badge-tag.low-ctr {
  background: rgba(52, 211, 153, 0.13);
  color: #6ee7b7;
}

.badge-tag.internal-link {
  background: rgba(245, 158, 11, 0.14);
  color: #fcd34d;
}

.priority-badge {
  font-size: 10.5px;
  font-weight: 600;
  padding: 4px 10px;
  border-radius: 20px;
  display: inline-block;
}

.priority-badge.high {
  background: rgba(239, 68, 68, 0.14);
  color: #fca5a5;
}

.priority-badge.medium {
  background: rgba(251, 146, 60, 0.14);
  color: #fdba74;
}

.priority-badge.low {
  background: rgba(90, 86, 82, 0.3);
  color: var(--text-secondary);
}

.empty-state-row {
  padding: 40px !important;
  color: var(--text-muted);
  font-size: 14px;
}

.table-footer {
  padding: 11px 18px;
  border-top: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 12px;
  color: var(--text-muted);
}

/* DRAWER / DETAILS PANEL */
.drawer-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
}

.drawer-backdrop {
  position: absolute;
  inset: 0;
  background: rgba(20, 18, 17, 0.6);
  backdrop-filter: blur(2px);
}

.drawer-panel {
  position: relative;
  width: 440px;
  height: 100%;
  background: #211f1d;
  border-left: 1px solid rgba(90, 86, 82, 0.8);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  box-shadow: -20px 0 50px rgba(0, 0, 0, 0.4);
}

.drawer-header {
  padding: 24px 26px;
  border-bottom: 1px solid var(--border-color);
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 14px;
}

.drawer-title {
  font-size: 17px;
  font-weight: 600;
  color: var(--text-primary);
  margin-top: 11px;
  line-height: 1.35;
}

.drawer-url {
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 5px;
  word-break: break-all;
}

.close-drawer-btn {
  width: 32px;
  height: 32px;
  flex: 0 0 32px;
  border-radius: 8px;
  border: 1px solid var(--border-color);
  background: rgba(62, 60, 57, 0.6);
  color: #a8a199;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
}

.close-drawer-btn:hover {
  color: var(--text-primary);
  border-color: var(--border-hover);
}

.drawer-content {
  padding: 24px 26px;
  display: flex;
  flex-direction: column;
  gap: 22px;
  flex: 1;
}

.drawer-stats {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 10px;
}

.mini-stat-card {
  padding: 13px;
  border-radius: 11px;
  background: rgba(62, 60, 57, 0.3);
  border: 1px solid var(--border-color);
  text-align: center;
}

.mini-stat-card.highlight-border {
  border-color: rgba(225, 37, 26, 0.3);
  background: rgba(225, 37, 26, 0.04);
}

.stat-num {
  font-family: 'Outfit', sans-serif;
  font-size: 18px;
  font-weight: 700;
  color: var(--text-primary);
}

.stat-num.text-accent {
  color: var(--accent-hover);
}

.stat-lbl {
  font-size: 10.5px;
  color: var(--text-muted);
  margin-top: 2px;
}

.drawer-section {
  display: flex;
  flex-direction: column;
}

.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 11px;
  font-size: 13.5px;
  font-weight: 600;
  color: var(--text-primary);
}

.section-title.text-success-title svg {
  stroke: var(--color-green);
}

.section-body {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.65;
}

.action-box {
  padding: 15px 16px;
  border-radius: 12px;
  background: rgba(52, 211, 153, 0.05);
  border: 1px solid rgba(52, 211, 153, 0.18);
}

.action-body {
  font-size: 13px;
  color: #a7f3d0;
  line-height: 1.6;
  font-weight: 500;
}

.gain-banner {
  padding: 15px 16px;
  border-radius: 12px;
  background: linear-gradient(135deg, rgba(225, 37, 26, 0.1), rgba(176, 24, 16, 0.05));
  border: 1px solid rgba(225, 37, 26, 0.2);
}

.gain-lbl {
  font-size: 11px;
  color: var(--text-secondary);
  margin-bottom: 6px;
}

.gain-val {
  font-size: 20px;
  font-weight: 700;
  color: var(--color-green);
  font-family: 'Outfit', sans-serif;
}

.drawer-ai-btn {
  width: 100%;
  padding: 13px;
  border-radius: 11px;
  border: 1px solid rgba(225, 37, 26, 0.3);
  background: rgba(225, 37, 26, 0.1);
  color: var(--accent-light);
  font-size: 12.5px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: all 0.15s;
}

.drawer-ai-btn:hover {
  background: rgba(225, 37, 26, 0.18);
  border-color: rgba(225, 37, 26, 0.5);
  color: var(--text-primary);
}

/* ============ 3. CANNIBALIZATION STYLES ============ */
.cannibal-view {
  padding: 28px 34px;
  max-width: 1080px;
  margin: 0 auto;
  width: 100%;
}

.cannibal-content {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.warning-banner {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 16px 20px;
  border-radius: 14px;
  background: rgba(251, 146, 60, 0.06);
  border: 1px solid rgba(251, 146, 60, 0.18);
}

.warning-banner.info-blue {
  background: rgba(96, 165, 250, 0.06);
  border-color: rgba(96, 165, 250, 0.18);
}

.warning-text {
  flex: 1;
}

.warning-title {
  font-size: 13.5px;
  font-weight: 600;
  color: #fdba74;
}

.warning-desc {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 2px;
  line-height: 1.5;
}

.cannibal-list {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.cannibal-card {
  border-radius: 16px;
  background: rgba(62, 60, 57, 0.2);
  border: 1px solid var(--border-color);
  overflow: hidden;
}

.cannibal-card-header {
  padding: 16px 20px;
  border-bottom: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.cannibal-left {
  display: flex;
  align-items: center;
  gap: 11px;
}

.cannibal-count-badge {
  font-size: 11px;
  font-weight: 700;
  padding: 3px 9px;
  border-radius: 7px;
  background: rgba(251, 146, 60, 0.15);
  color: var(--color-orange);
}

.cannibal-query-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
}

.cannibal-total-impr {
  font-size: 12px;
  color: var(--text-muted);
}

.cannibal-pages {
  display: flex;
  flex-direction: column;
}

.cannibal-page-row {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 13px 20px;
  border-top: 1px solid rgba(90, 86, 82, 0.2);
}

.cannibal-page-row:first-child {
  border-top: none;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex: 0 0 8px;
}

.cannibal-url {
  flex: 1;
  font-size: 12.5px;
  color: var(--text-secondary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.cannibal-pos, .cannibal-clicks {
  font-size: 12px;
  color: var(--text-muted);
  width: 90px;
  text-align: right;
}

.cannibal-tag {
  font-size: 10px;
  font-weight: 600;
  padding: 3px 9px;
  border-radius: 6px;
  width: 52px;
  text-align: center;
  background: rgba(251, 146, 60, 0.14);
  color: #fdba74;
}

.cannibal-tag.primary {
  background: rgba(52, 211, 153, 0.14);
  color: #6ee7b7;
}

/* ============ 4. DECAY VIEW STYLES ============ */
.decay-view {
  padding: 28px 34px;
  max-width: 1120px;
  margin: 0 auto;
  width: 100%;
}

.decay-content {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.decay-stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}

.decay-stat-card {
  padding: 18px 20px;
  border-radius: 14px;
  border: 1px solid var(--border-color);
}

.decay-stat-card.red-bg {
  background: rgba(239, 68, 68, 0.05);
  border-color: rgba(239, 68, 68, 0.18);
}

.decay-stat-card.green-bg {
  background: rgba(52, 211, 153, 0.05);
  border-color: rgba(52, 211, 153, 0.18);
}

.decay-stat-card.dark-bg {
  background: rgba(62, 60, 57, 0.3);
}

.stat-title-num {
  font-size: 24px;
  font-weight: 700;
  font-family: 'Outfit', sans-serif;
}

.decay-stat-card.red-bg .stat-title-num { color: #f87171; }
.decay-stat-card.green-bg .stat-title-num { color: #34d399; }
.decay-stat-card.dark-bg .stat-title-num { color: var(--text-primary); }

.stat-desc {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 3px;
}

.change-tag {
  font-size: 12.5px;
  padding: 3px 10px;
  border-radius: 7px;
  display: inline-block;
}

/* ============ 5. AI VIEW STYLES ============ */
.ai-view {
  height: calc(100vh - 60px);
  display: flex;
  flex-direction: column;
}

.ai-disabled-box {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 34px;
}

.disabled-inner {
  max-width: 420px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.spark-icon-box {
  width: 66px;
  height: 66px;
  border-radius: 18px;
  background: rgba(225, 37, 26, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid rgba(225, 37, 26, 0.25);
  margin-bottom: 20px;
}

.disabled-title {
  font-size: 18px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 9px;
}

.disabled-desc {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.6;
  margin-bottom: 22px;
}

.ai-chat-interface {
  height: 100%;
  display: flex;
  flex-direction: column;
  max-width: 820px;
  margin: 0 auto;
  width: 100%;
}

.chat-messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 28px 30px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.chat-row {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

.chat-row.user {
  flex-direction: row-reverse;
}

.ai-avatar-badge {
  width: 32px;
  height: 32px;
  flex: 0 0 32px;
  border-radius: 9px;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  display: flex;
  align-items: center;
  justify-content: center;
}

.chat-bubble {
  max-width: 78%;
  padding: 14px 17px;
  border-radius: 16px;
  font-size: 13px;
  line-height: 1.6;
  background: rgba(62, 60, 57, 0.3);
  border: 1px solid var(--border-color);
  color: var(--text-secondary);
  border-top-left-radius: 5px;
}

/* Markdown formatting inside chat bubbles */
.markdown-body {
  font-size: 13.5px;
  line-height: 1.6;
}

.markdown-body h1, 
.markdown-body h2, 
.markdown-body h3 {
  color: var(--text-primary);
  font-weight: 700;
  margin-top: 14px;
  margin-bottom: 8px;
}

.markdown-body h1 { font-size: 16px; border-bottom: 1px solid rgba(255,255,255,0.06); padding-bottom: 4px; }
.markdown-body h2 { font-size: 14.5px; }
.markdown-body h3 { font-size: 13.5px; }

.markdown-body p {
  margin-top: 0;
  margin-bottom: 10px;
  color: var(--text-secondary);
}

.markdown-body ul, 
.markdown-body ol {
  margin-top: 0;
  margin-bottom: 12px;
  padding-left: 20px;
}

.markdown-body li {
  margin-bottom: 4px;
  color: var(--text-secondary);
}

.markdown-body code {
  font-family: 'Courier New', Courier, monospace;
  background: rgba(255, 255, 255, 0.08);
  padding: 2px 5px;
  border-radius: 4px;
  color: #ff8c82;
  font-size: 12px;
  font-weight: 500;
}

.markdown-body pre {
  background: rgba(0, 0, 0, 0.25);
  border: 1px solid var(--border-color);
  padding: 10px 14px;
  border-radius: 8px;
  overflow-x: auto;
  margin-bottom: 12px;
}

.markdown-body pre code {
  background: transparent;
  padding: 0;
  color: #f8f8f2;
}

.markdown-body strong {
  color: var(--text-primary);
  font-weight: 600;
}

.markdown-body blockquote {
  border-left: 3px solid #E1251A;
  padding-left: 12px;
  margin: 10px 0;
  color: var(--text-muted);
}

.chat-row.user .chat-bubble {
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  color: #fff;
  border: none;
  border-top-left-radius: 16px;
  border-top-right-radius: 5px;
}

.chat-input-container {
  flex: 0 0 auto;
  padding: 16px 30px 24px;
  border-top: 1px solid var(--border-color);
  background-color: var(--bg-primary);
}

.chat-input-wrapper {
  display: flex;
  gap: 10px;
  align-items: flex-end;
}

.chat-input {
  flex: 1;
  padding: 13px 16px;
  border-radius: 13px;
  background: rgba(62, 60, 57, 0.4);
  border: 1px solid var(--border-color);
  color: var(--text-primary);
  font-size: 13px;
  outline: none;
  transition: border-color 0.18s;
}

.chat-input:focus {
  border-color: var(--border-hover);
}

.send-prompt-btn {
  width: 46px;
  height: 46px;
  flex: 0 0 46px;
  border-radius: 13px;
  border: none;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  color: #fff;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 14px rgba(225, 37, 26, 0.3);
  transition: all 0.18s;
}

.send-prompt-btn:hover {
  box-shadow: 0 6px 18px rgba(225, 37, 26, 0.45);
}

.model-info-footer {
  font-size: 10.5px;
  color: var(--text-muted);
  margin-top: 9px;
  text-align: center;
}

/* ============ 6. SETTINGS VIEW STYLES ============ */
.settings-view {
  padding: 28px 34px;
  max-width: 820px;
  margin: 0 auto;
  width: 100%;
}

.settings-content {
  display: flex;
  flex-direction: column;
  gap: 22px;
}

.settings-card {
  border-radius: 16px;
  background: rgba(62, 60, 57, 0.2);
  border: 1px solid var(--border-color);
  padding: 24px 26px;
}

.card-section-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 4px;
}

.card-section-desc {
  font-size: 12px;
  color: var(--text-muted);
  margin-bottom: 22px;
}

.settings-control {
  display: flex;
  flex-direction: column;
}

.spacer-bottom {
  margin-bottom: 26px;
}

.control-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
}

.control-label {
  font-size: 13px;
  color: var(--text-secondary);
  font-weight: 500;
}

.control-val {
  font-size: 13px;
  font-weight: 600;
  color: var(--accent-hover);
}

.accent-slider {
  width: 100%;
  accent-color: var(--accent-red);
  height: 5px;
  background: rgba(90, 86, 82, 0.5);
  border-radius: 3px;
  cursor: pointer;
}

.slider-boundaries {
  display: flex;
  justify-content: space-between;
  margin-top: 6px;
  font-size: 11px;
  color: var(--text-muted);
}

.flex-control-row {
  display: flex;
  gap: 16px;
}

.half-control {
  flex: 1;
}

.input-lbl {
  font-size: 11px;
  color: var(--text-muted);
  margin-bottom: 6px;
}

/* API ENTEGRATIONS */
.api-control-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 0;
}

.api-control-row.border-bottom {
  border-bottom: 1px solid rgba(90, 86, 82, 0.3);
}

.api-identity {
  display: flex;
  align-items: center;
  gap: 13px;
}

.api-icon {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.api-icon.blue-bg {
  background: rgba(66, 133, 244, 0.12);
  border: 1px solid rgba(66, 133, 244, 0.25);
}

.api-icon.red-bg {
  background: rgba(225, 37, 26, 0.12);
  border: 1px solid rgba(225, 37, 26, 0.25);
}

.api-name {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--text-primary);
}

.api-status {
  font-size: 11.5px;
}

.api-action-btn {
  padding: 8px 16px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
  border: 1px solid rgba(66, 133, 244, 0.35);
  background: rgba(66, 133, 244, 0.1);
  color: #93c5fd;
}

.api-action-btn:hover {
  background: rgba(66, 133, 244, 0.18);
}

.api-action-btn.disconnect {
  border-color: rgba(239, 68, 68, 0.3);
  background: rgba(239, 68, 68, 0.08);
  color: #fca5a5;
}

.api-action-btn.disconnect:hover {
  background: rgba(239, 68, 68, 0.15);
}

.switch-toggle {
  width: 44px;
  height: 25px;
  flex: 0 0 44px;
  border-radius: 20px;
  border: none;
  cursor: pointer;
  position: relative;
  transition: all 0.2s ease;
  background: rgba(90, 86, 82, 0.5);
  padding: 0;
}

.switch-toggle.active {
  background: linear-gradient(135deg, var(--accent-red), #b01810);
}

.switch-knob {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 19px;
  height: 19px;
  border-radius: 50%;
  background: #fff;
  transition: all 0.2s ease;
  box-shadow: 0 1px 3px rgba(0,0,0,0.3);
}

.switch-toggle.active .switch-knob {
  left: 22px;
}

.flex-row-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.gemini-sub-config {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding-left: 53px;
  margin-top: 14px;
}

.sub-input-group {
  display: flex;
  flex-direction: column;
}

.sub-input-label {
  font-size: 11.5px;
  color: var(--text-muted);
  margin-bottom: 6px;
}

.settings-input {
  width: 100%;
  padding: 10px 13px;
  border-radius: 10px;
  background: rgba(20, 18, 17, 0.6);
  border: 1px solid var(--border-color);
  color: var(--text-primary);
  font-size: 12.5px;
  font-family: monospace;
  outline: none;
  transition: border-color 0.15s;
}

.settings-input:focus {
  border-color: var(--border-hover);
}

.models-row-buttons {
  display: flex;
  gap: 8px;
}

.model-btn {
  flex: 1;
  padding: 9px 8px;
  border-radius: 9px;
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  border: 1px solid var(--border-color);
  background: rgba(20, 18, 17, 0.4);
  color: var(--text-secondary);
}

.model-btn:hover {
  border-color: var(--border-hover);
}

.model-btn.active {
  border-color: rgba(225, 37, 26, 0.4);
  background: rgba(225, 37, 26, 0.12);
  color: var(--accent-light);
  font-weight: 600;
}

/* PREMIUM BANNER CARD */
.premium-ad-card {
  border-radius: 16px;
  background: linear-gradient(135deg, rgba(225, 37, 26, 0.12), rgba(176, 24, 16, 0.05));
  border: 1px solid rgba(225, 37, 26, 0.25);
  padding: 26px;
  position: relative;
  overflow: hidden;
}

.radial-glow {
  position: absolute;
  top: -40px;
  right: -40px;
  width: 160px;
  height: 160px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(225, 37, 26, 0.25), transparent 70%);
}

.premium-card-body {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  position: relative;
  z-index: 2;
}

.premium-card-left {
  flex: 1;
}

.premium-tag-header {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-bottom: 10px;
}

.premium-tag {
  font-size: 10px;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 6px;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  color: #fff;
  letter-spacing: 0.5px;
}

.premium-cta-title {
  font-size: 15px;
  font-weight: 600;
  color: var(--text-primary);
  font-family: 'Outfit', sans-serif;
}

.premium-desc {
  font-size: 12.5px;
  color: var(--text-secondary);
  line-height: 1.6;
  max-width: 440px;
}

.connect-account-btn {
  flex: 0 0 auto;
  padding: 13px 22px;
  border-radius: 12px;
  border: none;
  background: linear-gradient(135deg, var(--accent-red), #b01810);
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 6px 20px rgba(225, 37, 26, 0.35);
  white-space: nowrap;
  transition: all 0.18s;
}

.connect-account-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 8px 24px rgba(225, 37, 26, 0.45);
}

/* ANIMATIONS & TRANSITIONS */
@keyframes ql-fade {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.animate-fade {
  animation: ql-fade 0.3s ease;
}

@keyframes ql-drawer {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

.animate-slide {
  animation: ql-drawer 0.28s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes pulse {
  0% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(52, 211, 153, 0.7);
  }
  70% {
    transform: scale(1);
    box-shadow: 0 0 0 6px rgba(52, 211, 153, 0);
  }
  100% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(52, 211, 153, 0);
  }
}

.slide-enter-active, .slide-leave-active {
  transition: all 0.3s ease;
}

.slide-enter-from, .slide-leave-to {
  opacity: 0;
}

.slide-enter-from .drawer-panel, .slide-leave-to .drawer-panel {
  transform: translateX(100%);
}
</style>