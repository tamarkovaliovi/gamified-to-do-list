<script setup>
import { ref, computed, watch, onMounted, onUnmounted } from "vue";
import confetti from "canvas-confetti";

const bildirimler = ref([]);

function bildirimGonder(baslik, mesaj, ikon = "✨", tip = "normal") {
  const id = Date.now();
  bildirimler.value.push({ id, baslik, mesaj, ikon, tip });
  setTimeout(() => {
    bildirimler.value = bildirimler.value.filter((b) => b.id !== id);
  }, 4000);
}

const kayitliGorevler = JSON.parse(localStorage.getItem("yapilacaklar") || "[]");
const kayitliRozetler = JSON.parse(localStorage.getItem("rozetler") || "[]");

const yeniGorev = ref("");
const secilenOncelik = ref("must");
const gorevListesi = ref(kayitliGorevler);
const kazanilanRozetler = ref(kayitliRozetler);

const oncelikAyarlari = {
  must: { key: "must", puan: 50, etiket: "Must", renk: "kirmizi", noktaRenk: "#ef4444" },
  should: {
    key: "should",
    puan: 30,
    etiket: "Should",
    renk: "sari",
    noktaRenk: "#f59e0b",
  },
  could: { key: "could", puan: 20, etiket: "Could", renk: "yesil", noktaRenk: "#10b981" },
};

const menüAcikMi = ref(false);
const dropdownRef = ref(null);

function öncelikSeç(key) {
  secilenOncelik.value = key;
  menüAcikMi.value = false;
}

function disariTiklama(event) {
  if (dropdownRef.value && !dropdownRef.value.contains(event.target)) {
    menüAcikMi.value = false;
  }
}

onMounted(() => {
  document.addEventListener("click", disariTiklama);
});
onUnmounted(() => {
  document.removeEventListener("click", disariTiklama);
});

function ekle() {
  if (yeniGorev.value.trim() !== "") {
    const secimDetaylari = oncelikAyarlari[secilenOncelik.value];
    gorevListesi.value.push({
      baslik: yeniGorev.value,
      yapildi: false,
      oncelik: secilenOncelik.value,
      puan: secimDetaylari.puan,
      renk: secimDetaylari.renk,
    });
    bildirimGonder("Görev Eklendi", "Listeye başarıyla eklendi.");
    yeniGorev.value = "";
  }
}

function sil(sira) {
  const silinen = gorevListesi.value[sira];
  gorevListesi.value.splice(sira, 1);
  bildirimGonder("Görev Silindi", `"${silinen.baslik}" kaldırıldı.`, "");
}

function durumDegisti(gorev) {
  if (gorev.yapildi) {
    bildirimGonder("Harika İş!", `"${gorev.baslik}" tamamlandı!`);
    confetti({ particleCount: 40, spread: 50, origin: { y: 0.7 } });
  } else {
    bildirimGonder("Geri Alındı", "Görev tekrar aktif.");
  }
}

const suruklenenIndeks = ref(null);

function suruklemeBasladi(index) {
  suruklenenIndeks.value = index;
}

function suruklemeBitti() {
  suruklenenIndeks.value = null;
}

function uzerineBirakildi(birakilanIndeks) {
  const silinen = gorevListesi.value.splice(suruklenenIndeks.value, 1)[0];
  gorevListesi.value.splice(birakilanIndeks, 0, silinen);
}

const sprintModuAcik = ref(false);
const secilenSure = ref(25);
const kalanSaniye = ref(25 * 60);
const zamanlayici = ref(null);
const zamanlayiciCalisiyor = ref(false);

watch(secilenSure, (yeniDakika) => {
  if (!zamanlayiciCalisiyor.value) kalanSaniye.value = yeniDakika * 60;
});

function sprintBaslat() {
  if (zamanlayiciCalisiyor.value) {
    clearInterval(zamanlayici.value);
    zamanlayiciCalisiyor.value = false;
  } else {
    bildirimGonder("Sprint Başladı", "Odaklanma modu aktif! ");
    zamanlayiciCalisiyor.value = true;
    zamanlayici.value = setInterval(() => {
      if (kalanSaniye.value > 0) kalanSaniye.value--;
      else sprintBitti();
    }, 1000);
  }
}

function sprintSifirla() {
  clearInterval(zamanlayici.value);
  zamanlayiciCalisiyor.value = false;
  kalanSaniye.value = secilenSure.value * 60;
}

function sprintBitti() {
  sprintSifirla();
  bildirimGonder("Süre Doldu!", "Zaman Efendisi rozetine yaklaştın!");
  rozetAc("sprint_usta");
  confetti({ particleCount: 150, spread: 100, origin: { y: 0.6 } });
}

const formatliZaman = computed(() => {
  const dk = Math.floor(kalanSaniye.value / 60);
  const sn = kalanSaniye.value % 60;
  return `${dk < 10 ? "0" + dk : dk}:${sn < 10 ? "0" + sn : sn}`;
});

const sprintYuzdesi = computed(() => {
  const toplamSaniye = secilenSure.value * 60;
  return ((toplamSaniye - kalanSaniye.value) / toplamSaniye) * 100;
});

const tumRozetler = [
  { id: "ilk_gorev", baslik: "First Step", ikon: "🥉", renk: "turuncu" },
  { id: "yari_yol", baslik: "Halfway Hero", ikon: "🔥", renk: "sari" },
  { id: "sprint_usta", baslik: "Time Lord", ikon: "⚡", renk: "mor" },
  { id: "sampiyon", baslik: "Champion", ikon: "🏆", renk: "mavi" },
];

function rozetAc(rozetId) {
  if (!kazanilanRozetler.value.includes(rozetId)) {
    kazanilanRozetler.value.push(rozetId);
    localStorage.setItem("rozetler", JSON.stringify(kazanilanRozetler.value));
    const r = tumRozetler.find((x) => x.id === rozetId);
    bildirimGonder("Yeni Rozet!", `${r.baslik} rozetini kazandın!`, r.ikon);
    confetti({ particleCount: 100, spread: 70 });
  }
}

const toplamPuan = computed(() => {
  let toplam = 0;
  gorevListesi.value.forEach((g) => {
    if (g.yapildi) toplam += g.puan || 10;
  });
  return toplam;
});

const hedefPuan = computed(() => {
  let toplam = 0;
  gorevListesi.value.forEach((g) => {
    toplam += g.puan || 10;
  });
  return toplam;
});

const skorYuzdesi = computed(() => {
  if (hedefPuan.value === 0) return 0;
  return Math.round((toplamPuan.value / hedefPuan.value) * 100);
});

const nextMilestone = computed(() => {
  if (skorYuzdesi.value < 25) return 25;
  if (skorYuzdesi.value < 50) return 50;
  if (skorYuzdesi.value < 75) return 75;
  if (skorYuzdesi.value < 100) return 100;
  return 100;
});

const seviyeIsmi = computed(() => {
  const yuzde = skorYuzdesi.value;
  if (yuzde === 100) return "Legend 👑";
  if (yuzde >= 75) return "Expert ⚔️";
  if (yuzde >= 50) return "Apprentice 🔨";
  return "Novice 🐣";
});

watch(
  gorevListesi,
  (yeniListe, eskiListe) => {
    localStorage.setItem("yapilacaklar", JSON.stringify(yeniListe));
    const bitenSayisi = yeniListe.filter((g) => g.yapildi).length;

    if (bitenSayisi >= 1) rozetAc("ilk_gorev");
    if (skorYuzdesi.value >= 50) rozetAc("yari_yol");

    if (skorYuzdesi.value === 100 && yeniListe.length > 0) {
      const eskiYuzde = eskiListe
        ? Math.round((eskiListe.filter((g) => g.yapildi).length / eskiListe.length) * 100)
        : 0;
      if (eskiYuzde !== 100) {
        bildirimGonder("Mükemmel!", "Tüm görevler bitti!", "🏆");
        rozetAc("sampiyon");
        confetti({ particleCount: 200, spread: 150 });
      }
    }
  },
  { deep: true }
);

function herSeyiSifirla() {
  if (confirm("Tüm verileri silmek istediğine emin misin? Bu işlem geri alınamaz!")) {
    localStorage.clear();
    location.reload();
  }
}
</script>

<template>
  <h1>GAMİFİED TO-DO LİST</h1>
  <div class="sayfa-duzeni">
    <div class="bildirim-alani">
      <TransitionGroup name="liste-animasyon">
        <div
          v-for="bildirim in bildirimler"
          :key="bildirim.id"
          class="bildirim-karti"
          :class="{ 'silme-tipi': bildirim.tip === 'silme' }"
        >
          <div class="bildirim-ikon">{{ bildirim.ikon }}</div>
          <div class="bildirim-icerik">
            <div class="bildirim-baslik">{{ bildirim.baslik }}</div>
            <div class="bildirim-mesaj">{{ bildirim.mesaj }}</div>
          </div>
        </div>
      </TransitionGroup>
    </div>

    <div class="ana-konteyner">
      <div class="bilesen-kutu">
        <div class="istatistik-ust">
          <div class="sutun">
            <div class="baslik-ikonlu">
              <span class="ikon-daire mavi">🏆</span
              ><span class="baslik-yazi">Progress</span>
            </div>
            <div class="cubuk-dis-ince">
              <div
                class="cubuk-ic-ince mavi-bg"
                :style="{ width: skorYuzdesi + '%' }"
              ></div>
            </div>
            <div class="alt-bilgi">
              <strong>%{{ skorYuzdesi }} Complete</strong>
            </div>
          </div>
          <div class="sutun">
            <div class="baslik-ikonlu">
              <span class="ikon-daire sari">⭐</span
              ><span class="baslik-yazi">Total Points</span>
            </div>
            <div class="puan-kocaman">
              {{ toplamPuan }} <span class="gri">/ {{ hedefPuan }}</span>
            </div>
          </div>
          <div class="sutun">
            <div class="baslik-ikonlu">
              <span class="ikon-daire mor">🎖️</span><span class="baslik-yazi">Level</span>
            </div>
            <div class="seviye-badge">🔵 {{ seviyeIsmi }}</div>
          </div>
        </div>

        <div class="milestone-alani">
          <div class="milestone-header">
            <span class="milestone-baslik">Milestone Progress</span>
            <span class="milestone-next">Next: {{ nextMilestone }}%</span>
          </div>

          <div class="milestone-track-container">
            <div class="milestone-track">
              <div class="milestone-fill" :style="{ width: skorYuzdesi + '%' }"></div>
            </div>

            <div class="milestone-markers">
              <div class="marker-item" style="left: 25%">
                <div class="marker-tick"></div>
                <div class="marker-label">25%</div>
              </div>
              <div class="marker-item" style="left: 50%">
                <div class="marker-tick"></div>
                <div class="marker-label">50%</div>
              </div>
              <div class="marker-item" style="left: 75%">
                <div class="marker-tick"></div>
                <div class="marker-label">75%</div>
              </div>
              <div class="marker-item" style="left: 100%">
                <div class="marker-tick"></div>
                <div class="marker-label">100%</div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="bilesen-kutu sprint-kutusu">
        <div class="sprint-baslik-satir" @click="sprintModuAcik = !sprintModuAcik">
          <div class="baslik-sol">
            <span class="simsek">⚡</span><span>Work Sprint</span>
          </div>
          <select
            v-model="secilenSure"
            class="zaman-secici"
            @click.stop
            :disabled="zamanlayiciCalisiyor"
          >
            <option :value="5">5 Min</option>
            <option :value="25">25 Min</option>
            <option :value="45">45 Min</option>
          </select>
        </div>

        <div v-if="sprintModuAcik" class="sprint-panel">
          <div class="zaman-gostergesi">{{ formatliZaman }}</div>
          <div class="sprint-cubuk-dis">
            <div class="sprint-cubuk-ic" :style="{ width: sprintYuzdesi + '%' }"></div>
          </div>
          <div class="sprint-butonlar">
            <button
              @click="sprintBaslat"
              :class="zamanlayiciCalisiyor ? 'duraklat' : 'baslat'"
            >
              {{ zamanlayiciCalisiyor ? "Pause" : "Start" }}
            </button>
            <button @click="sprintSifirla" class="sifirla">Reset</button>
          </div>
        </div>
      </div>

      <div class="bilesen-kutu">
        <div class="baslik-ikonlu">
          <span class="ikon-daire turuncu-bg">🏷️</span
          ><span class="baslik-yazi">Badges Earned</span>
        </div>
        <div class="rozetler-listesi">
          <div
            v-for="rozet in tumRozetler"
            :key="rozet.id"
            class="rozet-karti"
            :class="{
              kazanildi: kazanilanRozetler.includes(rozet.id),
              [rozet.renk]: true,
            }"
          >
            <span class="rozet-ikon">{{ rozet.ikon }}</span
            ><span class="rozet-isim">{{ rozet.baslik }}</span>
          </div>
        </div>
      </div>

      <div class="bilesen-kutu giris-ozel-kutu">
        <div class="giris-kutusu">
          <div class="oncelik-dropdown-container" ref="dropdownRef">
            <div class="dropdown-tetikleyici" @click="menüAcikMi = !menüAcikMi">
              <span
                class="renk-noktasi"
                :style="{ backgroundColor: oncelikAyarlari[secilenOncelik].noktaRenk }"
              ></span>
              <span class="secilen-yazi">{{
                oncelikAyarlari[secilenOncelik].etiket
              }}</span>
              <span class="dropdown-ok">⌄</span>
            </div>
            <div v-if="menüAcikMi" class="dropdown-menu">
              <div
                v-for="(ayarlar, key) in oncelikAyarlari"
                :key="key"
                class="dropdown-item"
                :class="{ secili: secilenOncelik === key }"
                @click="öncelikSeç(key)"
              >
                <span
                  class="renk-noktasi"
                  :style="{ backgroundColor: ayarlar.noktaRenk }"
                ></span>
                <span class="item-yazi">{{ ayarlar.etiket }}</span>
              </div>
            </div>
          </div>
          <input
            v-model="yeniGorev"
            @keyup.enter="ekle"
            type="text"
            placeholder="Add a new task..."
          />
          <button @click="ekle" class="ekle-butonu">+</button>
        </div>
      </div>

      <div v-if="gorevListesi.length === 0" class="bos-durum-ozel">
        <div class="ikon-kutusu-yuvarlak">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke-width="2.5"
            stroke="currentColor"
            class="ikon-svg"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M4.5 12.75l6 6 9-13.5"
            />
          </svg>
        </div>
        <p>No tasks yet. Add your first task above!</p>
      </div>

      <div v-else class="bilesen-kutu liste-kutusu">
        <ul class="liste">
          <li
            v-for="(gorev, index) in gorevListesi"
            :key="index"
            class="gorev-karti-yeni"
            :class="[gorev.oncelik, { surukleniyor: suruklenenIndeks === index }]"
            draggable="true"
            @dragstart="suruklemeBasladi(index)"
            @dragend="suruklemeBitti"
            @dragover.prevent
            @drop="uzerineBirakildi(index)"
          >
            <div class="tutamac"><span>⋮⋮</span></div>
            <label class="gorev-icerik-alani">
              <input
                type="checkbox"
                v-model="gorev.yapildi"
                @change="durumDegisti(gorev)"
              />
              <span class="gorev-yazisi" :class="{ cizik: gorev.yapildi }">{{
                gorev.baslik
              }}</span>
            </label>
            <div class="gorev-sag-bilgi">
              <span class="oncelik-badge-yeni" :class="gorev.renk">
                <span v-if="gorev.yapildi">✔ Done</span>
                <span v-else>@ {{ oncelikAyarlari[gorev.oncelik].etiket }}</span>
              </span>
              <span class="puan-kapsulu">{{ gorev.puan }} pts</span>
              <button @click="sil(index)" class="sil-butonu-yeni">
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width="1.5"
                  stroke="currentColor"
                  style="width: 18px; height: 18px"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0"
                  />
                </svg>
              </button>
            </div>
          </li>
        </ul>
      </div>

      <div class="footer-alani">
        <button @click="herSeyiSifirla" class="reset-btn-iconlu">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke-width="1.5"
            stroke="currentColor"
            class="reset-icon"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0"
            />
          </svg>
          Reset All Data
        </button>
      </div>
    </div>
  </div>
</template>

<style>
/* --- SAYFA DÜZENİ --- */
body {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  background-color: #f3f4f6;
}
.sayfa-duzeni {
  display: flex;
  justify-content: center;
  min-height: 100vh;
  font-family: "Segoe UI", sans-serif;
  padding: 40px 20px;
}
.ana-konteyner {
  width: 100%;
  max-width: 850px;
  display: flex;
  flex-direction: column;
  gap: 25px;
}

/* --- KART GENEL STİLİ --- */
.bilesen-kutu {
  background: white;
  padding: 30px;
  border-radius: 20px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
  border: 1px solid #e5e7eb;
}
.liste-kutusu {
  padding: 0;
  overflow: hidden;
}

/* --- MILESTONE (YENİ GÖRÜNÜM) --- */
.milestone-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}
.milestone-baslik {
  font-weight: 600;
  color: #374151;
  font-size: 15px;
}
.milestone-next {
  font-size: 12px;
  color: #6b7280;
}

.milestone-track-container {
  position: relative;
  padding-bottom: 25px; /* Yazılar için yer açtık */
}
.milestone-track {
  background-color: #f3f4f6;
  height: 16px;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}
.milestone-fill {
  background-color: #3b82f6;
  height: 100%;
  transition: width 0.6s cubic-bezier(0.4, 0, 0.2, 1);
  border-radius: 10px;
}

.milestone-markers {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}
.marker-item {
  position: absolute;
  top: 0;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
}
.marker-tick {
  width: 2px;
  height: 16px;
  background-color: rgba(255, 255, 255, 0.7); /* Çubuk içinde beyaz çizgi */
  z-index: 2;
}
.marker-label {
  margin-top: 5px;
  font-size: 11px;
  color: #9ca3af;
  font-weight: 500;
}

h1 {
  text-align: center;
  margin-bottom: 30px;
  color: #347fe9;
  font-size: 28px;
}

/* --- RESET BUTONU --- */
.reset-btn-iconlu {
  display: flex;
  align-items: center;
  gap: 8px;
  background-color: #fee2e2;
  color: #ef4444;
  border: 1px solid #fecaca;
  padding: 8px 16px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 10px;
}
.reset-btn-iconlu:hover {
  background-color: #fecaca;
  transform: translateY(-1px);
}
.reset-icon {
  width: 16px;
  height: 16px;
}

/* --- RENKLİ GÖREV KARTLARI --- */
.liste {
  list-style: none;
  padding: 0;
  margin: 0;
}
.gorev-karti-yeni {
  display: flex;
  align-items: center;
  padding: 16px 20px;
  margin: 10px 20px;
  border-radius: 8px;
  background-color: #fff;
  border: 1px solid #f3f4f6;
  transition: all 0.2s ease;
  cursor: grab;
}
.gorev-karti-yeni:active {
  cursor: grabbing;
}
.gorev-karti-yeni.must {
  background-color: #fff5f5;
  border-left: 5px solid #ef4444;
}
.gorev-karti-yeni.should {
  background-color: #fffbeb;
  border-left: 5px solid #f59e0b;
}
.gorev-karti-yeni.could {
  background-color: #f0fdf4;
  border-left: 5px solid #10b981;
}
.surukleniyor {
  opacity: 0.5;
  transform: scale(0.98);
  border: 2px dashed #9ca3af;
}
.tutamac {
  color: #d1d5db;
  font-size: 20px;
  margin-right: 15px;
  cursor: grab;
  user-select: none;
}
.tutamac:hover {
  color: #9ca3af;
}
.gorev-icerik-alani {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  cursor: pointer;
}
.gorev-yazisi {
  font-weight: 500;
  color: #1f2937;
  font-size: 16px;
  transition: color 0.2s;
}
.gorev-yazisi.cizik {
  text-decoration: line-through;
  color: #9ca3af;
}
.gorev-sag-bilgi {
  display: flex;
  align-items: center;
  gap: 15px;
}
.oncelik-badge-yeni {
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 20px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 4px;
}
.oncelik-badge-yeni.kirmizi {
  background-color: white;
  color: #ef4444;
  border: 1px solid #fecaca;
}
.oncelik-badge-yeni.sari {
  background-color: white;
  color: #d97706;
  border: 1px solid #fde68a;
}
.oncelik-badge-yeni.yesil {
  background-color: white;
  color: #059669;
  border: 1px solid #a7f3d0;
}
.puan-kapsulu {
  background-color: #f3f4f6;
  color: #6b7280;
  font-size: 11px;
  padding: 4px 8px;
  border-radius: 6px;
  font-weight: 700;
}
.sil-butonu-yeni {
  background: none;
  border: none;
  color: #ef4444;
  opacity: 0.2;
  cursor: pointer;
  transition: opacity 0.2s;
  display: flex;
  align-items: center;
}
.gorev-karti-yeni:hover .sil-butonu-yeni {
  opacity: 1;
}

/* --- DİĞER BİLEŞENLER --- */
.bos-durum-ozel {
  border: 2px dashed #d1d5db;
  background-color: white;
  border-radius: 12px;
  padding: 50px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #6b7280;
}
.ikon-kutusu-yuvarlak {
  width: 60px;
  height: 60px;
  background-color: #eff6ff;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 15px;
}
.ikon-svg {
  width: 30px;
  height: 30px;
  color: #3b82f6;
}
.giris-ozel-kutu {
  padding: 20px;
}
.giris-kutusu {
  display: flex;
  gap: 10px;
}
.sprint-kutusu {
  border: 1px dashed #3b82f6;
  background: #fdfdff;
}
.sprint-baslik-satir {
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
}
.baslik-sol {
  display: flex;
  align-items: center;
  gap: 10px;
  font-weight: 600;
  color: #2563eb;
}
.zaman-secici {
  padding: 5px 10px;
  border-radius: 6px;
  border: 1px solid #dbeafe;
  color: #1e40af;
  font-weight: bold;
  background: white;
}
.sprint-panel {
  margin-top: 20px;
  text-align: center;
}
.zaman-gostergesi {
  font-size: 36px;
  font-weight: bold;
  font-family: monospace;
  color: #1e293b;
  margin-bottom: 10px;
}
.sprint-cubuk-dis {
  height: 6px;
  background: #eff6ff;
  margin-bottom: 15px;
  border-radius: 3px;
}
.sprint-cubuk-ic {
  height: 100%;
  background: #10b981;
  transition: width 1s linear;
}
.sprint-butonlar button {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  color: white;
  font-weight: 600;
  margin: 0 5px;
}
.baslat {
  background: #1040b9;
}
.duraklat {
  background: #f59e0b;
}
.sifirla {
  background: #ef4444;
}
.oncelik-dropdown-container {
  position: relative;
  width: 140px;
  min-width: 140px;
}
.dropdown-tetikleyici {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 15px;
  border: 2px solid #e5e7eb;
  border-radius: 10px;
  background: white;
  cursor: pointer;
  font-weight: bold;
  color: #374151;
  height: 100%;
  box-sizing: border-box;
}
.dropdown-tetikleyici:hover {
  border-color: #3b82f6;
}
.renk-noktasi {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  margin-right: 10px;
}
.secilen-yazi {
  flex: 1;
}
.dropdown-menu {
  position: absolute;
  top: 110%;
  left: 0;
  width: 100%;
  background: white;
  border: 2px solid #e5e7eb;
  border-radius: 10px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  z-index: 50;
  overflow: hidden;
}
.dropdown-item {
  display: flex;
  align-items: center;
  padding: 10px 15px;
  cursor: pointer;
  font-weight: 600;
  color: #4b5563;
}
.dropdown-item:hover {
  background-color: #f3f4f6;
}
.dropdown-item.secili {
  background-color: #eff6ff;
  color: #3b82f6;
}
input[type="text"] {
  flex: 1;
  padding: 12px 16px;
  border: 2px solid #e5e7eb;
  border-radius: 10px;
  outline: none;
  transition: border-color 0.2s;
}
input[type="text"]:focus {
  border-color: #3b82f6;
}
.ekle-butonu {
  background: #3b82f6;
  color: white;
  border: none;
  width: 48px;
  border-radius: 10px;
  font-size: 24px;
  cursor: pointer;
  transition: background 0.2s;
}
.ekle-butonu:hover {
  background: #2563eb;
}
.istatistik-ust {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-bottom: 25px;
}
.baslik-ikonlu {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}
.baslik-yazi {
  font-weight: 600;
  color: #6b7280;
  font-size: 13px;
}
.ikon-daire {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
}
.mavi {
  background: #eff6ff;
  color: #3b82f6;
}
.sari {
  background: #fefce8;
  color: #eab308;
}
.mor {
  background: #faf5ff;
  color: #a855f7;
}
.turuncu-bg {
  background: #fff7ed;
  color: #ea580c;
}
.cubuk-dis-ince {
  height: 6px;
  background: #f3f4f6;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 5px;
}
.cubuk-ic-ince {
  height: 100%;
  transition: width 0.5s ease;
}
.mavi-bg {
  background-color: #3b82f6;
}
.puan-kocaman {
  font-size: 26px;
  font-weight: 800;
  color: #111827;
}
.seviye-badge {
  font-weight: 700;
  font-size: 15px;
  color: #374151;
}
.rozetler-listesi {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 15px;
}
.rozet-karti {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border-radius: 20px;
  background: #f9fafb;
  color: #9ca3af;
  font-size: 12px;
  font-weight: 600;
  border: 1px solid #f3f4f6;
}
.rozet-karti.kazanildi {
  background: white;
  color: #374151;
  border-color: transparent;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}
.turuncu.kazanildi {
  color: #ea580c;
  background: #fff7ed;
}
.sari.kazanildi {
  color: #ca8a04;
  background: #fefce8;
}
.mor.kazanildi {
  color: #9333ea;
  background: #faf5ff;
}
.mavi.kazanildi {
  color: #2563eb;
  background: #eff6ff;
}
.footer-alani {
  text-align: center;
  margin-top: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}
.footer-yazi {
  color: #9ca3af;
  font-size: 13px;
}
.bildirim-alani {
  position: fixed;
  bottom: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 9999;
}
.bildirim-karti {
  background: white;
  width: 320px;
  padding: 15px;
  border-radius: 12px;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  gap: 15px;
  border-left: 5px solid #3b82f6;
  animation: slideIn 0.4s ease;
}
.bildirim-karti.silme-tipi {
  border-left-color: #ef4444;
}
.bildirim-ikon {
  font-size: 24px;
}
.bildirim-icerik {
  display: flex;
  flex-direction: column;
}
.bildirim-baslik {
  font-weight: bold;
  color: #1f2937;
  font-size: 14px;
}
.bildirim-mesaj {
  font-size: 12px;
  color: #6b7280;
  margin-top: 2px;
}
.liste-animasyon-enter-active,
.liste-animasyon-leave-active {
  transition: all 0.5s ease;
}
.liste-animasyon-enter-from,
.liste-animasyon-leave-to {
  opacity: 0;
  transform: translateX(30px);
}
@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}
</style>
