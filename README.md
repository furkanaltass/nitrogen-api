# NitrogenAPI

Nitrogen, Lapex ağının sunucu çekirdeğidir. Profil, ekonomi, seviye, başarım, oyun iskeleti, harita, scoreboard, hologram ve NPC sistemlerini tek çatı altında toplar. Oyun modları (örn. **Katil Kim**) Nitrogen'e bağımlı ayrı eklentiler olarak yazılır ve tüm ortak işleri `NitrogenAPI` üzerinden çekirdeğe bırakır.

> Sürüm: `son.bahar:Nitrogen:1.0.0` — Spigot 1.8.8 (XSpigot)

---

## İçindekiler

- [Kurulum](#kurulum)
- [Hızlı Başlangıç: Bir Oyun Modu Yazmak](#hızlı-başlangıç-bir-oyun-modu-yazmak)
- [NitrogenAPI Metodları](#nitrogenapi-metodları)
- [Game Sınıfı (Oyun İskeleti)](#game-sınıfı-oyun-iskeleti)
- [GameTeam](#gameteam)
- [Harita Sistemi: GameMap ve MapManager](#harita-sistemi-gamemap-ve-mapmanager)
- [GameScoreboard](#gamescoreboard)
- [UpdateEvent (Tick Motoru)](#updateevent-tick-motoru)
- [Oyun Eventleri](#oyun-eventleri)
- [Yardımcı Sınıflar](#yardımcı-sınıflar)
- [Sık Yapılan Hatalar](#sık-yapılan-hatalar)

---

## Kurulum

### Maven

Oyun eklentileri `nitrogen-parent` altında bir modül olarak açılır ve çekirdeğe `provided` kapsamıyla bağlanır:

```xml
<dependency>
    <groupId>son.bahar</groupId>
    <artifactId>Nitrogen</artifactId>
    <version>1.0.0</version>
    <scope>provided</scope>
</dependency>
```

### plugin.yml

```yaml
name: KatilKim
main: son.bahar.katilkim.GameService
version: 1.0.0
depend: [Nitrogen]
```

`depend` zorunludur: Nitrogen önce yüklenir, API sınıfları classpath'ten çekirdek jar'ından çözülür.

---

## Hızlı Başlangıç: Bir Oyun Modu Yazmak

Bir oyun modu üç parçadan oluşur: **ana sınıf** (tur döngüsü), **oyun sınıfı** (`Game`/`SoloGame` alt sınıfı) ve **mekanikler** (Listener sınıfları).

```java
public final class GameService extends JavaPlugin implements Listener {

    private MyGame game;

    @Override
    public void onEnable() {
        Bukkit.getPluginManager().registerEvents(this, this);
        Bukkit.getScheduler().runTask(this, this::startRound);
    }

    public MyGame getGame() {
        return game;
    }

    public void startRound() {
        if (game != null && game.getState() != Game.GameState.Dead) {
            return;
        }

        MyGame next = new MyGame();

        GameMap map = NitrogenAPI.getMapManager().loadRandom(next);
        if (map == null) {
            getLogger().warning("Oynanabilir harita bulunamadı.");
            return;
        }

        next.PlayerMin = Math.max(2, map.getMinPlayers());
        next.PlayerFull = Math.max(next.PlayerMin, map.getMaxPlayers());

        game = next;
        NitrogenAPI.registerGame(next);
    }

    @EventHandler
    public void onGameDead(GameDeadEvent event) {
        Bukkit.getScheduler().runTaskLater(this, this::startRound, 200L);
    }
}
```

```java
public final class MyGame extends SoloGame {

    public MyGame() {
        super("Oyun Adı", new String[]{
                "Oyunun kısa açıklaması,",
                "birkaç satır halinde."
        }, null);

        ModeName = "Klasik";
        GameTimeout = 300_000L;
        DeathMessages = false;
        PlayerGameMode = GameMode.ADVENTURE;
        Prepare = false;
    }

    @Override
    public void endCheck() {
        if (!isLive()) {
            return;
        }
        if (getPlayers(true).size() <= 1) {
            announceEnd(getPlayers(true));
        }
    }
}
```

Önemli kalıplar:

- **Her tur yeni bir oyun nesnesi** kurulur; tur durumu (roller, sayaçlar) oyun nesnesinin alanlarında yaşar, `GameDeadEvent` ile birlikte çöpe gider — elle temizlik gerekmez.
- **Mekanikler bir kez register edilir**, her event'te `service.getGame()` üzerinden güncel turu çözer ve `game == null || !game.isLive()` guard'ıyla başlar.
- `registerGame` çağrısı oyunu çekirdeğe bağlar: state döngüsü, sayaçlar, spectator sistemi, bekleme lobisi itemleri, ölüm/çıkış işleme, scoreboard ve ödüller çekirdek tarafından yürütülür.

---

## NitrogenAPI Metodları

`son.bahar.nitrogen.NitrogenAPI` — tamamı statiktir.

### Profil ve İstatistik

| Metod | Açıklama |
|---|---|
| `getProfile(Player)` / `getProfile(UUID)` | Online oyuncunun profili (yoksa `null`). |
| `loadOfflineProfile(UUID)` | Diskten offline profil yükler. |
| `addGameStat(Player, String, long)` | Bulunulan sunucu tipine stat ekler, başarımları tetikler. Yeni değeri döndürür. |
| `addGameStat(Player, ServerType, String, long)` | Belirli oyun moduna stat ekler. |
| `getGameStat(Player, ServerType, String)` | Stat okur. |

Profil üzerinde ayrıca `getStat(key)`, `setStat(key, value)`, `addStat(key, delta)` ve `addExperience(amount)` bulunur (örn. KatilKim rol ağırlıkları bunlarla tutulur).

### Ekonomi, Tecrübe ve Seviye

| Metod | Açıklama |
|---|---|
| `getCoins(Player)` / `addCoins(Player, long)` / `removeCoins(Player, long)` | Ham coin işlemleri (mesajsız). |
| `rewardCoins(Player, long)` | Booster çarpanı uygular, coin verir ve `+N Coin` mesajını basar. Verilen miktarı döndürür. |
| `getExperience(Player)` | Toplam tecrübe. |
| `rewardExperience(Player, long)` | Booster çarpanıyla tecrübe verir, mesaj basar, başarımları kontrol eder. |
| `getLevel(Player)` / `getLevelByExperience(long)` | Seviye çözümleme. |
| `isBoosterActive()` / `getBoosterMultiplier()` | Ağ geneli ödül çarpanı. |

### Hazır Ödül Akışları

| Metod | Ne yapar |
|---|---|
| `recordKill(Player)` | `kills` + sezonluk `kills` statı, **4 coin** (çarpanlı, "Öldürme" etiketiyle), kılıçla öldürmede **40**, diğer hallerde **75 tecrübe**. |
| `recordWin(Player)` | `wins` + sezonluk `wins` + `winstreak`, **10 coin**, **100 tecrübe**. |
| `recordLoss(Player)` | Winstreak'i sıfırlar. |
| `recordGamePlayed(Player)` | `played` statını artırır. |

Bu metodlar oyun iskeleti tarafından otomatik çağrılır (ölüm attribution'ı, oyun sonu ödülleri); mekaniklerden elle çağırmak genelde gerekmez.

### Görevler (Challenge)

| Metod | Açıklama |
|---|---|
| `isChallengeActive(Player, [ServerType,] String)` | Görev oyuncu için aktif mi. |
| `recordChallengeCompletion(Player, [ServerType,] String)` | İlerleme kaydeder. |
| `getChallengeCompletions(Player, ServerType, String)` | İlerleme okur. |

### Oyun İskeleti

| Metod | Açıklama |
|---|---|
| `getGame()` | Sunucudaki aktif `Game` (yoksa `null`). |
| `registerGame(Game)` | Oyunu çekirdeğe kaydeder ve döngüyü başlatır. |
| `getGameManager()` | Oyun yöneticisi (build modu, spectator vb. iç işler). |
| `getMapManager()` | Harita yöneticisi. |
| `getBeklemeSpawn()` | `/nitrogen bekleme` ile ayarlanan ana bekleme noktası (yoksa `null`; iskelet bu durumda haritanın `bekleme` spawn'ına düşer). |

### Sunucu ve Ağ

| Metod | Açıklama |
|---|---|
| `getServerType()` | Sunucunun oyun tipi (`ServerType`). |
| `isLobbyInstance()` | Bu sunucu lobi mi. |
| `getBungeeName()` | BungeeCord'daki sunucu adı. |
| `getChatPrefix()` | Standart chat prefix'i. |
| `getOnlineServers()` | Redis heartbeat'inden canlı sunucu listesi (`ServerStatus`). |

`ServerStatus` alanları: `getBungeeName()`, `getServerName()`, `getTypeName()`, `isLobby()`, `getPlayerCount()`, `getMaxPlayers()`, `isJoinable()`.

- Heartbeat 5 saniyede bir yazılır, 15 saniye TTL ile düşer — çökmüş sunucu listeden kendiliğinden silinir.
- `isJoinable()` = sunucudaki oyun **Recruit** aşamasında (katılınabilir). "Müsait arenaya katıl" bu bayrağı kullanır.

### Sosyal, Hologram ve NPC

| Metod | Açıklama |
|---|---|
| `getTakim(Player)` / `getLonca(Player)` | Oyuncunun takım/lonca görünümü. |
| `createHologram(Location, String...)` | Paket tabanlı hologram oluşturur. |
| `spawnNpc(Location, EntityType, String, Consumer<Player>)` | Tıklanabilir NPC spawnlar; tıklamada verilen aksiyon çalışır. |

---

## Game Sınıfı (Oyun İskeleti)

`son.bahar.nitrogen.game.Game` soyut sınıftır; solo modlar için `SoloGame` hazır tabanı kullanılır (tek gizli takım, otomatik `chooseTeam`/`canStart`).

### Yaşam Döngüsü

```
Loading → Recruit → (Prepare) → Live → End → Dead
```

| State | Açıklama |
|---|---|
| `Loading` | Harita yükleniyor (30 sn'de hazır olmazsa oyun `Dead`'e düşer). |
| `Recruit` | Bekleme: min oyuncu sayısına göre 60/30/10 sn geri sayım, eksikte action bar uyarısı. |
| `Prepare` | Isınma (opsiyonel, `Prepare=false` ile atlanır). |
| `Live` | Oyun; `GameTimeout` dolarsa `handleTimeout()` çağrılır. |
| `End` | Oyun sonu: 10 sn, kazanan fireworkleri, AutoGG payload'ı, ödüller. |
| `Dead` | Tur bitti; `GameDeadEvent` yayınlanır, dünya bir sonraki yüklemede sıfırlanır. |

### Bayraklar

Ctor içinde set edilir; tamamı public alan olarak durur.

| Bayrak | Varsayılan | Açıklama |
|---|---|---|
| `TeamMode` | `false` | Takım modu (takım seçme pusulası yalnızca bunda görünür). |
| `ModeName` | `"Solo"` | Scoreboard `{mod}` değeri. |
| `PlayerMin` / `PlayerFull` | `2` / `12` | Başlama alt sınırı ve kapasite. |
| `GameTimeout` | `1_200_000L` | Live süresi (ms), `-1` = sınırsız. |
| `Damage` / `DamagePvP` / `DamageSelf` / `DamageFall` / `DamageTeamSelf` | `true/true/true/true/false` | Hasar anahtarları. |
| `BlockBreak` / `BlockBreakAllow` / `BlockBreakDeny` | `false` | Blok kırma + materyal bazlı istisna setleri. |
| `BlockPlace` / `BlockPlaceAllow` / `BlockPlaceDeny` | `false` | Blok koyma + istisnalar. |
| `ItemDrop` / `ItemPickup` / `InventoryClick` | `false` | Item ve envanter anahtarları. |
| `DeathOut` | `true` | Ölen oyuncu oyundan çıkar (spectator olur). |
| `DeathDropItems` | `false` | Ölümde item düşer mi. |
| `DeathMessages` | `true` | Çekirdek ölüm mesajları. |
| `KillCreditMillis` | `15_000L` | Dolaylı ölümlerde (void, su, combat-log) son vuranın kill sayılma penceresi. |
| `QuitOut` | `true` | Oyundan çıkan Live'da tam ölüm işlemi görür (drop, kill kredisi, mesaj). |
| `CreatureAllow` | `false` | Doğal mob spawn'ı. |
| `FireballEnabled` + hız/yield/cooldown | `false` | Ateş topu itemi. |
| `TntAutoIgnite` / `TntFuseTicks` | `false` / `60` | TNT otomatik ateşleme. |
| `WorldTimeSet` | `12000` | Oyun başlarken set edilen dünya saati (harita yüklenirken de uygulanır). |
| `WorldWeatherEnabled` vb. dünya bayrakları | `false` | Yağmur, yangın yayılımı, yaprak çürümesi, ekin çiğneme. |
| `HungerSet` / `HealthSet` | `-1` | Başlangıç açlık/can (−1 = dokunma/20). |
| `AnnounceJoinQuit` | `true` | Giriş/çıkış duyuruları. |
| `AnnounceAlive` | `true` | Ölümlerde "Hayatta kalan: N" duyurusu. |
| `Prepare` / `PrepareTime` / `PrepareFreeze` | `true` / `9000L` / `true` | Isınma aşaması, süresi ve hareket kilidi. |
| `JoinInProgress` | `false` | Oyun ortasında girene spectator ver. |
| `PlayerGameMode` | `SURVIVAL` | Live'da oyunculara verilecek mod. |
| `KitRegisterState` | `Live` | Kit perk'lerinin register edileceği state. |

### Önemli Metodlar

| Metod | Açıklama |
|---|---|
| `getState()` / `setState(GameState)` | State okuma/geçiş; geçişte `GameStateChangeEvent` yayınlanır. |
| `isLive()` / `inProgress()` / `inLobby()` | Durum kısayolları. |
| `getPlayers(true)` | Hayatta olan oyuncular; `false` tüm katılımcılar. |
| `isAlive(Player)` | Oyuncu hayatta mı. |
| `setPlayerState(Player, PlayerState)` | `IN` / `OUT` işaretleme. |
| `announce(String)` | Prefix'li duyuru (`playSound` overload'ı vardır). |
| `announceEnd(GameTeam)` / `announceEnd(List<Player>)` | Kazananı ilan eder: `WinnerTeam`/`WinnerPlaces` set edilir, KAZANDIN animasyonu ve ödüller `isWinner` üzerinden dağıtılır, state `End`'e çekilir. |
| `getWaitingSpawn()` | Ana bekleme noktası; yoksa haritanın `bekleme` spawn'ı. |
| `getMap()` / `getScoreboard()` | Bağlı harita ve scoreboard. |

### Override Noktaları

| Metod | Zorunlu | Açıklama |
|---|---|---|
| `endCheck()` | Evet | Her ölüm/çıkış sonrası çağrılır; bitiş şartını kontrol edip oyunu bitir. |
| `canStart()` | Evet (`SoloGame` sağlar) | Geri sayım sıfırlanınca oyunun başlayabilirliği. |
| `onStateChange(GameState)` | Hayır | State geçiş kancası (örn. Live'da rol dağıtımı). |
| `isWinner(Player)` | Hayır | Ödül ve KAZANDIN title'ının tek kaynağı. Varsayılan: `WinnerTeam` üyeliği, yoksa `WinnerPlaces` birincisi. |
| `handleTimeout()` | Hayır | Süre dolunca; varsayılan uygulama oyunu berabere bitirir. |

---

## GameTeam

| Metod | Açıklama |
|---|---|
| `getFormattedName()` | Renk + isim. |
| `getCapacity()` | `setMaxPlayers` verilmişse o; yoksa `PlayerFull / takım sayısı`. |
| `addPlayer(Player, boolean)` / `removePlayer(Player)` | Üyelik; takım modunda katılım mesajı basılır. |
| `getPlayers(aliveOnly)` / `isAlive(Player)` / `isTeamAlive()` | Durum sorguları. |
| `getSpawn()` / `spawnTeleport(Player)` | Spawn dağıtımı (sıralı). |
| `getPlacements(includeAlive)` | Eleniş sırasına göre sıralama. |

---

## Harita Sistemi: GameMap ve MapManager

Haritalar şablon klasörlerde tutulur; her tur `game_world` adına **temiz bir kopya** açılır. Kopya `autoSave` kapalı yüklenir, açılışta ve her chunk yüklemesinde oyuncu dışı tüm entityler silinir, saat sabitlenir (`doDaylightCycle=false` + `WorldTimeSet`). Tur bitince dünya kaydedilmeden atılır — haritaya ne olursa olsun sonraki tur tertemiz başlar.

### MapManager

| Metod | Açıklama |
|---|---|
| `loadRandom(Game)` | Rastgele haritayı yükler ve oyuna bağlar (`applyTo`). `null` = oynanabilir harita yok. |
| `load(String, Game)` | İsimle yükler. |
| `unload()` | Runtime dünyayı kapatıp siler. |

### GameMap

| Metod | Açıklama |
|---|---|
| `getDisplayName()` / `getAuthor()` | map.yml'deki `isim` / `yapimci`. |
| `getMinPlayers()` / `getMaxPlayers()` | Harita kapasitesi. |
| `getWorld()` | Runtime dünya. |
| `getSpawns(String)` / `getSpawnGroups()` | Spawn grupları (örn. `oyuncular`, `bekleme`). |
| `getPoints(String)` | Nokta grupları (örn. KatilKim `altin` noktaları). |

### Harita Kurulumu (özet)

```
/nitrogen map olustur <isim>     şablon dünyayı oluşturur
/nitrogen map duzenle <isim>     kurulum oturumu açar (marker hologram + partiküller görünür)
/nitrogen map spawn <grup>       bulunduğun yere spawn ekler
/nitrogen map nokta <grup>       bulunduğun yere nokta ekler
/nitrogen map kaydet             dünyayı ve map.yml'yi kaydeder
/nitrogen bekleme                ana bekleme noktasını ayarlar (sunucu geneli)
```

Kurulum markerları yalnızca kurulum oturumunda görünür; kayıtta temizlenir.

---

## GameScoreboard

Çekirdek scoreboard'u state'e göre kendisi çizer. Live görünümünün orta bloğunu oyununa özel satırlarla değiştirebilirsin.

### Placeholder'lar

| Placeholder | Değer |
|---|---|
| `{sure}` | Live'da geçen süre (`mm:ss`). |
| `{kalan}` | `GameTimeout`'a kalan süre. |
| `{hayatta}` | Hayatta oyuncu sayısı. |
| `{harita}` | Harita görünen adı. |
| `{mod}` | `ModeName`. |

### Oyuncuya Özel Satırlar

```java
@Override
protected void onStateChange(GameState state) {
    if (state != GameState.Live) {
        return;
    }
    if (getScoreboard() != null) {
        getScoreboard().setLineProvider(this::scoreboardLines);
    }
}

private List<String> scoreboardLines(Player player) {
    List<String> lines = new ArrayList<>();
    lines.add(" &fHarita: &e{harita}");
    lines.add(" ");
    lines.add(" &fRolün: " + rolYazisi(player));
    lines.add(" &fZaman: &a{kalan}");
    return lines;
}
```

Provider her güncellemede oyuncu başına çağrılır; placeholder'lar satırlara otomatik uygulanır.

---

## UpdateEvent (Tick Motoru)

Çekirdek tek bir zamanlayıcıdan `UpdateEvent` yayınlar; kendi scheduler'ını kurmak yerine bunu dinle.

```java
@EventHandler
public void onUpdate(UpdateEvent event) {
    if (event.getType() != UpdateType.SEC) {
        return;
    }
}
```

| Tip | Sıklık | Tipik kullanım |
|---|---|---|
| `UpdateType.TICK` | Her tick | Animasyon (dönen item), akıcı action bar. |
| `UpdateType.SEC` | Saniyede bir | Geri sayım, periyodik spawn, tarama. |

---

## Oyun Eventleri

| Event | Ne zaman | Tipik kullanım |
|---|---|---|
| `GameStateChangeEvent` | Her state geçişinde | Duyuru, kurulum işleri (`getGame()`, `getState()`). |
| `GameDeadEvent` | Tur tamamen bittiğinde | Mekanik temizliği + yeni tur başlatma. |

```java
@EventHandler
public void onGameDead(GameDeadEvent event) {
    aktifStandlar.clear();
    Bukkit.getScheduler().runTaskLater(plugin, service::startRound, 200L);
}
```

---

## Yardımcı Sınıflar

Hepsi `son.bahar.nitrogen.util` altındadır.

| Sınıf | Öne çıkanlar |
|---|---|
| `ColorUtil` | `color(String)` — `&` kodlarını çevirir; `center(String)` — chat'te piksel bazlı ortalar (renklendirmeyi de yapar). |
| `TitleUtil` | `send(player, title, subtitle, in, stay, out)`, `sendActionBar(player, text)`, `sendJson(player, json, ...)` — Lapex istemcisinin `animZoom###` gibi title animasyonları için ham JSON. |
| `ItemBuilder` | `new ItemBuilder(Material).name("&e...").lore(...).build()`; kafalar için `.skull(name)`. |
| `Recharge` | `Recharge.use(player, "anahtar", 3000L)` — basit cooldown, kullanılabildiyse `true`. |
| `NametagUtil` | `hide(Player)` / `show(Player)` — ITab üzerinden isim etiketini gizler/gösterir (KatilKim rol gizliliği). |
| `RewardMessage` | `coins(player, miktar[, etiket])`, `experience(player, miktar)` — standart ödül mesajları. |
| `ClientPayload` | `send(player, method)` / `sendAutoGG(player)` — Lapex istemcisine özel payload (`Teyyap` kanalı). AutoGG, `End` state'ine geçişte iskelet tarafından herkese otomatik gönderilir. |

---

## Sık Yapılan Hatalar

- **`PlayerInteractEvent` + `ignoreCancelled=true` tuzağı:** 1.8'de havaya sağ tık (`RIGHT_CLICK_AIR`) event'i cancelled doğar. Havaya tıklamayı dinleyen handler'a `ignoreCancelled=true` koyarsan hiç çalışmaz. Blok tıklamalarında sorun yoktur.
- **Turlar arası state saklama:** Tur verisini mekaniklerde `static`/alan olarak tutma; oyun nesnesinin üstünde tut. Yeni tur = yeni nesne = otomatik sıfırlama. Mekanikte tutulması zorunlu koleksiyonları `GameDeadEvent`'te temizle.
- **`provided` scope:** Nitrogen sınıflarını kendi jar'ına gömme; `depend: [Nitrogen]` + `provided` yeterli.
- **Harita dünyasına kalıcı değişiklik:** Runtime dünya her tur çöpe gider. Kalıcı değişiklik haritanın şablonunda (`/nitrogen map duzenle` oturumunda) yapılır.
- **Ölüm işleme sırası:** Kill attribution `LOWEST`'ta, çekirdek ölüm işleme `HIGH`'da çalışır; `endCheck` ölümden bir tick sonra tetiklenir. Ölüm anında veri toplayan mekanikler için `MONITOR` güvenlidir.

# Nitrogen Depolama (Storage) Rehberi

Nitrogen'in kalıcı veri katmanı `son.bahar.nitrogen.storage` paketinde yaşar. Profiller (ve ileride diğer koleksiyonlar) tek bir belge modeliyle tutulur; arkadaki sağlayıcı (YAML dosyaları, SQL veritabanları, MongoDB, Cassandra) `config.yml` ile seçilir. Bu belge hem sunucu yöneticileri (kurulum, yapılandırma, taşıma, sorun giderme) hem de geliştiriciler (yeni koleksiyon kullanma, yeni sağlayıcı yazma, test araçları) içindir.

> Bu belge 2026-09-15 tarihli sürümün kodundan okunarak yazılmış ve taşıma komutu ile lonca kalıcılığı bölümleri son kodla karşılaştırılarak doğrulanmıştır.

---

## İçindekiler

- [Mimari özeti](#mimari-özeti)
- [Yapılandırma referansı](#yapılandırma-referansı)
- [Sağlayıcılar](#sağlayıcılar)
- [JDBC sürücüleri](#jdbc-sürücüleri)
- [Veri modeli](#veri-modeli)
- [Dayanıklılık davranışı](#dayanıklılık-davranışı)
- [Sağlık durumları](#sağlık-durumları)
- [Komutlar](#komutlar)
- [Depolar arası taşıma (migration)](#depolar-arası-taşıma-migration)
- [Lonca kalıcılığı](#lonca-kalıcılığı)
- [Geliştiriciler için](#geliştiriciler-için)
- [Standalone test araçları](#standalone-test-araçları)
- [Sorun giderme](#sorun-giderme)

---

## Mimari özeti

```
NitrogenAPI.getStorage()
        │
        ▼
StorageService ─── StorageSettings (config.yml → storage.*)
   │  │  │  │
   │  │  │  └── ProviderRegistry ── yerleşik fabrikalar (yaml, sqlite, h2, mysql, mariadb, postgresql)
   │  │  │                       └─ harici jar'lar (storage/providers/*.jar, ServiceLoader, ProviderClassLoader)
   │  │  └───── StorageHealth (10 sn'de bir ping, UP/DEGRADED/DOWN)
   │  └──────── DocumentStore("profiles") ── ProfileStore (ProfileCodec ile Profile ↔ StorageDocument)
   │            DocumentStore("<koleksiyon>")  ← NitrogenAPI.getStorage().store("...")
   │                    │ okuma: IO havuzu (3 thread), 2 deneme
   │                    │ yazma: WriteQueue (belge başına zincir, geri çekilmeli yeniden deneme)
   │                    ▼
   │            StorageProvider.collection(name) → DocumentCollection (load/save/delete/exists/forEachId/count)
   │
   ├── Outbox        storage/outbox/<koleksiyon>/<id>.json   (sağlayıcıya yazılamayan kayıtlar)
   ├── LocalCache    storage/cache/<koleksiyon>/<id>.json    (son başarılı okuma/yazma kopyası)
   ├── ConflictResolver storage/conflicts/<koleksiyon>/<id>-<zaman>.json (sürüm çakışması dökümleri)
   └── import sağlayıcısı (storage.import-from, yalnızca profiller için tembel içe aktarma)
```

Katmanlar:

| Sınıf | Görev |
|---|---|
| `StorageService` | Yaşam döngüsü (`start`/`shutdown`), sağlayıcı oluşturma ve açma, sağlık probu, outbox yeniden gönderimi, koleksiyon başına `DocumentStore` üretimi, otomatik kayıt zamanlayıcısı. |
| `StorageSettings` | `config.yml` `storage` bölümünü okur, eksik anahtarları varsayılanlarla tamamlar. |
| `DocumentStore` | Bir koleksiyon için asenkron okuma API'si + `WriteQueue`'ya yazma; okumalarda önce kuyruktaki bekleyen anlık görüntüye bakar, başarılı okumaları `LocalCache`'e yazar. |
| `ProfileStore` | `profiles` koleksiyonu üzerinde `Profile` nesneleri: giriş öncesi yükleme, parmak izi ile değişmemiş otomatik kayıtları atlama, önbellekten geçici yükleme, eski depodan içe aktarma. |
| `WriteQueue` | Belge başına sıralı zincir; geçici hatalarda üstel geri çekilme, tükenince `Outbox`; sürüm çakışmasında `ConflictResolver`; sonuçları `SaveCallback` ve `Listener`'lara bildirir. |
| `StorageHealth` / `HealthSnapshot` | Durum makinesi ve sayaçlar (okuma/yazma/başarısız/zorlanan/atılan, ortalama gecikme). |
| `ProviderRegistry` / `ProviderClassLoader` | Fabrika kaydı, `storage/providers/` altındaki jar'ların keşfi ve izole yüklenmesi. |
| `StorageLog` | Anahtar bazlı hız sınırlı loglama (aynı anahtar 30 sn'de bir, tekrar sayısıyla). |

Thread'ler: `Nitrogen-Storage-IO-N` (3 thread, okuma ve profil yükleme), `Nitrogen-Storage-Health` (prob zamanlayıcısı), kuyruk şeritleri (`storage.queue.worker-threads` kadar tek thread'li zamanlayıcı). Ana thread'e yalnızca `loadOfflineProfile` gibi açıkça senkron çağrılarda ve sınırlı süreyle beklenir.

Açılış sırası (`StorageService.start`): fabrikalar kaydedilir → harici jar'lar taranır → birincil sağlayıcı oluşturulur (oluşturulamazsa **YAML'a geri dönülür**, `SEVERE` log) → sağlayıcı IO thread'inde açılır, en fazla 10 sn beklenir (açılamazsa sağlık `DOWN`, prob her 10 sn'de yeniden dener) → `import-from` varsa aynı şekilde açılır → sağlık probu başlar → sağlayıcı açıksa outbox yeniden gönderilir → `Depolama hazır: sağlayıcı=..., otomatik kayıt=..., politika=..., durum=...` logu.

Kapanış (`shutdown`): otomatik kayıt durur, sağlık probu durur, kuyruk en fazla 10 sn boşaltılır, kalanlar outbox'a taşınır, sağlayıcılar kapatılır, IO havuzu kapanır. `Nitrogen.onDisable` bundan önce `ProfileManager.saveAll()` ile tüm online profilleri `SHUTDOWN` kaynağıyla kuyruğa alır.

---

## Yapılandırma referansı

Tüm anahtarlar `config.yml` içindeki `storage:` bölümündedir. Eksik anahtarlar ilk açılışta varsayılanlarıyla dosyaya yazılır (`config.yml içine varsayılan 'storage' bölümü eklendi.`). Sağlayıcı bölüm adları ve `type` değeri küçük harfe çevrilerek okunur.

### Genel

| Anahtar | Varsayılan | Anlamı |
|---|---|---|
| `storage.type` | `yaml` | Birincil sağlayıcı tipi: `yaml`, `sqlite`, `h2`, `mysql`, `mariadb`, `postgresql`, `mongodb`, `cassandra` (son ikisi harici jar ister). Bilinmeyen/oluşturulamayan tipte YAML'a geri dönülür. |
| `storage.import-from` | `none` | Tembel içe aktarma kaynağı (eski sağlayıcı tipi). `none`, boş veya `type` ile aynıysa kapalı. Bkz. [Depolar arası taşıma](#depolar-arası-taşıma-migration). |
| `storage.autosave-seconds` | `60` | Online profillerin periyodik kaydı (saniye). Alt sınır 10. |
| `storage.unavailable-policy` | `provisional` | Sağlayıcı erişilemezken profil yerel önbellekten yüklenebiliyorsa: `provisional` → geçici profille içeri al; `deny-login` (eş anlamlıları `deny`, `kick`) → girişi reddet. Önbellekte kopya yoksa her iki politikada da giriş reddedilir. |
| `storage.queue.worker-threads` | `1` | Yazma kuyruğu şerit sayısı (aynı belge her zaman aynı şeritte, sıralı kalır). |
| `storage.queue.retry-attempts` | `6` | Geçici hatada yeniden deneme sayısı; tükenince outbox. |
| `storage.queue.base-delay-ms` | `500` | İlk yeniden deneme gecikmesi; her denemede ikiye katlanır. |
| `storage.queue.max-delay-ms` | `30000` | Gecikme tavanı. |

### Sağlayıcı bölümleri

| Anahtar | Varsayılan | Anlamı |
|---|---|---|
| `storage.yaml.folder` | `profiles` | Profil dosyalarının klasörü; göreli yol `plugins/Nitrogen/` altına çözülür. Diğer koleksiyonlar bu klasörün yanına, koleksiyon adıyla açılır. |
| `storage.sqlite.file` | `storage/nitrogen.db` | SQLite dosyası (göreli → `plugins/Nitrogen/`). |
| `storage.h2.file` | `storage/nitrogen` | H2 dosya tabanı (uzantısız; `AUTO_SERVER=FALSE`). |
| `storage.<mysql|mariadb|postgresql>.host` | `localhost` | Sunucu adresi. |
| `... .port` | `3306` / `3306` / `5432` | Port. |
| `... .database` | `nitrogen` | Veritabanı adı. |
| `... .username` / `... .password` | `root` / `""` | Kimlik bilgileri (kullanıcı adı boşsa hiç gönderilmez). |
| `... .pool-size` | `6` | HikariCP havuz boyutu (1..64). SQLite için sabit 1, H2 için sabit 2. |
| `... .connection-timeout-ms` | `5000` | Bağlantı ve doğrulama zaman aşımı (alt sınır 250). |
| `... .ssl` | `false` | MySQL `useSSL`, MariaDB `sslMode=trust`, PostgreSQL `sslmode=require`. |
| `... .params` | `""` | JDBC URL'sine eklenecek ek parametreler (`a=1&b=2`; baştaki `?`/`&` temizlenir). |
| `storage.<jdbc tipi>.driver-jar` | (yok) | Sürücü jar'ının elle yolu (göreli → `plugins/Nitrogen/`). Verilirse indirme/önbellek atlanır; dosya yoksa açılış kalıcı hatayla başarısız olur. Tüm JDBC tipleri için geçerlidir; varsayılan bölümde yazılı değildir. |
| `storage.<jdbc tipi>.driver-download` | `true` | Sürücünün Maven Central'dan indirilmesine izin. `false` ise önbellek, `driver-jar` veya sunucuyla gelen eski sürücü kullanılır. |
| `storage.mongodb.uri` | `mongodb://localhost:27017` | Bağlantı URI'si (`mongodb+srv://` desteklenir; loglarda parola maskelenir). |
| `storage.mongodb.database` | `nitrogen` | Veritabanı; boş bırakılırsa URI'deki, o da yoksa `nitrogen`. |
| `storage.mongodb.collection-prefix` | `""` | Koleksiyon adı öneki (`smoke_profiles` gibi). |
| `storage.mongodb.server-selection-timeout-ms` | `5000` | Sunucu seçimi zaman aşımı; bağlantı/soket okuma zaman aşımları sabit 5000 ms. |
| `storage.cassandra.contact-points` | `["localhost"]` | `host` veya `host:port` listesi (IPv6 `[::1]:9042`). |
| `storage.cassandra.port` | `9042` | Port belirtmeyen contact point'ler için port. |
| `storage.cassandra.keyspace` | `nitrogen` | Keyspace; yoksa `SimpleStrategy` ile oluşturulur. Ad küçük harfe ve `[a-z0-9_]`'e indirgenir. |
| `storage.cassandra.datacenter` | `datacenter1` | Yerel veri merkezi (load balancing için zorunlu). |
| `storage.cassandra.username` / `password` | `""` | Kullanıcı adı boşsa kimlik doğrulama yapılmaz. |
| `storage.cassandra.request-timeout-ms` | `5000` | İstek zaman aşımı (250..600000). Şema işlemleri en az 15 sn, tarama (`forEachId`/`count`) en az 30 sn kullanır. |
| `storage.cassandra.connect-timeout-ms` | `request-timeout-ms` | Bağlantı kurma zaman aşımı (varsayılan bölümde yazılı değildir). |
| `storage.cassandra.replication-factor` | `1` | Keyspace oluşturulurken kullanılan çoğaltma faktörü. |

Boolean değerler için `true/false` dışında `evet/hayır/yes/no` da kabul edilir (`ProviderConfig.getBoolean`).

### Klasör yerleşimi

```
plugins/Nitrogen/
├── config.yml
├── profiles/                     yaml sağlayıcısı: <uuid>.yml (+ .yml.bak, .yml.tmp)
├── <koleksiyon>/                 yaml sağlayıcısı: diğer koleksiyonlar
└── storage/
    ├── nitrogen.db               sqlite varsayılan dosyası
    ├── providers/                harici sağlayıcı jar'ları (Nitrogen-MongoDB.jar, Nitrogen-Cassandra.jar)
    ├── drivers/                  indirilen JDBC sürücüleri (<artifact>-<sürüm>.jar + .sha1)
    ├── outbox/<koleksiyon>/      yazılamayan kayıtlar
    ├── cache/<koleksiyon>/       yerel önbellek
    ├── conflicts/<koleksiyon>/   çakışma dökümleri
    └── corrupt/                  karantinaya alınan bozuk YAML dosyaları
```

---

## Sağlayıcılar

| Tip | Görünen ad | Konum | Arka uç |
|---|---|---|---|
| `yaml` | YAML dosyaları | yerleşik | Belge başına `.yml` dosyası, yedek ve karantina ile. |
| `sqlite` | SQLite | yerleşik | Tek dosya, WAL modu; sürücü indirilir. |
| `h2` | H2 | yerleşik | Dosya tabanlı H2; sürücü indirilir. |
| `mysql` | MySQL | yerleşik | HikariCP havuzu; sürücü indirilir. |
| `mariadb` | MariaDB | yerleşik | HikariCP havuzu; sürücü indirilir (eski sürücüye düşerse `jdbc:mysql` URL'si kullanılır). |
| `postgresql` | PostgreSQL | yerleşik | HikariCP havuzu; sürücü indirilir. |
| `mongodb` | MongoDB | **harici**: `Nitrogen-MongoDB.jar` | `mongodb-driver-sync 4.9.0` (jar'a gömülü, `son.bahar.nitrogen.lib.mongodb` altına taşınmış). |
| `cassandra` | Cassandra | **harici**: `Nitrogen-Cassandra.jar` | `java-driver-core-shaded 4.17.0` + slf4j (jar'a gömülü). |

`/nitrogen storage providers` kayıtlı fabrikaları ve bulunan harici jar'ları listeler.

### Harici sağlayıcı jar'ları

1. Modülü derle: `storage-mongodb` → `storage-mongodb/target/Nitrogen-MongoDB.jar`, `storage-cassandra` → `storage-cassandra/target/Nitrogen-Cassandra.jar` (pom `finalName`).
2. Jar'ı `plugins/Nitrogen/storage/providers/` klasörüne koy (klasör yoksa açılışta oluşturulur).
3. `storage.type` değerini `mongodb` veya `cassandra` yap, ilgili bölümü doldur, sunucuyu yeniden başlat.

Keşif: `ProviderRegistry.loadExternal` klasördeki her `.jar` için bir `ProviderClassLoader` açar ve `META-INF/services/son.bahar.nitrogen.storage.spi.StorageProviderFactory` dosyasındaki fabrikaları `ServiceLoader` ile yükler. Başarılı keşif logu: `Harici depolama sağlayıcısı bulundu: mongodb (MongoDB) <- Nitrogen-MongoDB.jar`. Fabrika içermeyen jar `(sağlayıcı yok)` etiketiyle listelenir. Jar'lar yalnızca açılışta taranır; sonradan eklenen jar için yeniden başlatma gerekir.

Sınıf yükleme kuralı (`ProviderClassLoader`): `son.bahar.nitrogen.`, `java.`, `javax.`, `org.bukkit.`, `com.google.gson.`, `net.minecraft.` önekleri **önce üst yükleyiciden** (Nitrogen jar'ı ve sunucu), geri kalan her şey **önce jar'ın kendisinden** çözülür. Böylece her sağlayıcı kendi sürücü sürümünü taşıyabilir ve Nitrogen'in shaded kütüphaneleriyle çakışmaz. Kaynaklar (`getResource`) da önce jar'dan aranır.

### Geri dönüş davranışı

`storage.type` bilinmiyorsa (örn. jar konmamış) veya fabrika sağlayıcıyı oluşturamazsa: `Depolama sağlayıcısı oluşturulamadı (<tip>): ... YAML sağlayıcısına GERİ DÖNÜLÜYOR — veriler yapılandırılan depoya YAZILMAYACAK!` logu basılır ve YAML ile devam edilir. `/nitrogen storage status` bu durumu `(yapılandırılan: <tip>, geri dönüldü)` ile gösterir. Sağlayıcı oluşturulup **açılamıyorsa** (veritabanı kapalı) geri dönüş yapılmaz; sağlık `DOWN` kalır, kayıtlar kuyruk/outbox'ta bekler ve prob her 10 sn'de yeniden açmayı dener.

---

## JDBC sürücüleri

JDBC sağlayıcıları sürücüleri jar'a gömmez; `DriverLoader` çalışma zamanında yükler.

### Sabitlenmiş sürümler (`DriverCatalog`)

| Tip | Maven koordinatı | Sürücü sınıfı | Eski (sunucu classpath) sürücü |
|---|---|---|---|
| `sqlite` | `org.xerial:sqlite-jdbc:3.46.1.3` | `org.sqlite.JDBC` | `org.sqlite.JDBC` |
| `h2` | `com.h2database:h2:2.2.224` | `org.h2.Driver` | yok |
| `mysql` | `com.mysql:mysql-connector-j:8.0.33` | `com.mysql.cj.jdbc.Driver` | `com.mysql.jdbc.Driver` |
| `mariadb` | `org.mariadb.jdbc:mariadb-java-client:3.3.3` | `org.mariadb.jdbc.Driver` | `com.mysql.jdbc.Driver` |
| `postgresql` | `org.postgresql:postgresql:42.7.3` | `org.postgresql.Driver` | yok |

### Çözümleme sırası

1. `storage.<tip>.driver-jar` verilmişse o jar (yoksa kalıcı hata: `Yapılandırmada belirtilen JDBC sürücü dosyası bulunamadı`).
2. Önbellek: `plugins/Nitrogen/storage/drivers/<artifact>-<sürüm>.jar`. Yanındaki `.sha1` dosyasıyla doğrulanır; `.sha1` yoksa ve indirme açıksa özet Maven Central'dan çekilir. Özet uyuşmazsa jar silinip yeniden indirilir; özet hiç alınamazsa jar doğrulanmadan kullanılır (uyarı ile).
3. İndirme (`driver-download: true`): `https://repo1.maven.org/maven2/...` adresinden `.part` dosyasına indirilir (15 sn bağlantı/okuma zaman aşımı), SHA-1 kontrol edilir, atomik taşınır. Loglar: `JDBC sürücüsü indiriliyor: ...`, `JDBC sürücüsü indirildi: ... (N KB, SHA-1 doğrulandı)`.
4. İndirilemezse veya indirme kapalıysa: sunucu classpath'indeki **eski sürücü** (Spigot 1.8 ile gelen SQLite ve MySQL sürücüleri) `Eski yerleşik JDBC sürücüsü kullanılıyor: ...` uyarısıyla kullanılır. MariaDB bu durumda `jdbc:mysql://` URL'siyle bağlanır; H2 ve PostgreSQL için eski sürücü yoktur.
5. Hiçbiri yoksa açılış kalıcı hatayla başarısız olur: `... sürücüsü bulunamadı ve indirme kapalı (storage.<tip>.driver-download: false); jar dosyasını <yol> yoluna koyun`.

İndirilen sürücü `DriverJarClassLoader` ile izole yüklenir: `java.`, `javax.`, `sun.`, `jdk.`, `com.sun.`, `org.w3c.`, `org.xml.`, `org.ietf.`, `son.bahar.nitrogen.` üstten, geri kalan jar'dan; `org.slf4j` hiçbir zaman üst yükleyiciye düşmez (Nitrogen'in shaded slf4j'i ile karışmaz).

### Bağlantı havuzu ve URL

- HikariCP havuz adı `Nitrogen-<tip>`; `maximumPoolSize = minimumIdle = pool-size` (SQLite 1, H2 2); `connectionTimeout = connection-timeout-ms`; doğrulama sorgusu `SELECT 1`; dosya tabanlı tiplerde `maxLifetime = 0`; `initializationFailTimeout = 1` (havuz açılışta bağlanamazsa `open()` hata verir).
- SQLite her bağlantıda `PRAGMA journal_mode=WAL`, `PRAGMA busy_timeout=5000`, `PRAGMA synchronous=NORMAL` çalıştırır.
- URL parametreleri: MySQL `useSSL`, (SSL kapalıysa) `allowPublicKeyRetrieval=true`, `useUnicode=true`, `characterEncoding=utf8`, `serverTimezone=UTC`, `connectTimeout`; MariaDB `sslMode`, `connectTimeout`; PostgreSQL `sslmode`, `connectTimeout`, `loginTimeout` (saniye), `ApplicationName=Nitrogen`; ardından `params`.
- Başarılı açılış logu: `MySQL bağlantısı kuruldu: host:port/db (havuz 6, sürücü indirildi: mysql-connector-j-8.0.33.jar)`. Tablo hazırlığı ilk kullanımda: `MySQL tablosu hazır: nitrogen_profiles`.

---

## Veri modeli

### Belge zarfı

Her kayıt bir `StorageDocument`'tır:

| Alan | Tip | Açıklama |
|---|---|---|
| `id` | `String` | Belge kimliği (profillerde oyuncu UUID'si). |
| `name` | `String` | Görünen ad (profillerde oyuncu adı; SQL'de 48 karaktere kesilir). |
| `version` | `long` | Optimistic-lock sürümü. `VERSION_NEW = 0` (henüz kaydedilmedi), `VERSION_ANY = -1` (koşulsuz üzerine yaz). |
| `updatedAt` | `long` | Sağlayıcının yazdığı epoch ms. |
| `data` | `Map<String, Object>` | İçerik. `DocumentJson.normalize` ile normalize edilir: tam sayılar `Long`, ondalıklar `Double`, enum → `name()`, `UUID` → string, iç içe `Map`/`List` korunur, diğer nesneler `String.valueOf`. |

### Sürüm semantiği (`DocumentCollection.save(document, expectedVersion)`)

| `expectedVersion` | Kayıt yok | Kayıt var, sürüm eşit | Kayıt var, sürüm farklı |
|---|---|---|---|
| `VERSION_NEW` (0) | `SAVED(1)` | `CONFLICT(mevcut)` | `CONFLICT(mevcut)` |
| `n > 0` | `SAVED(n+1)` (ekleme) | `SAVED(n+1)` | `CONFLICT(mevcut)` |
| `VERSION_ANY` (-1) | `SAVED(1)` | `SAVED(mevcut+1)` | `SAVED(mevcut+1)` |

Tüm sağlayıcılar bu tabloyu uygular; `/nitrogen storage test` ve smoke araçları bunu doğrular.

### Profil belgesi (`ProfileCodec`)

Koleksiyon `profiles`, `id` = UUID, `name` = oyuncu adı. `data` anahtarları: `name`, `coins`, `experience`, `credits`, `vipRankColor`, `lastBonusClaim`, `bonusStreak`, `achievements` (liste), `achievementDates` (anahtar → epoch ms), `cosmetics` (liste), `activeCosmetics` (slot → kozmetik), `ignored` (UUID listesi), `preferences` (anahtar → string), `stats` (anahtar → sayı; `KATIL_KIM:kills` gibi `TİP:stat` biçiminde).

### SQL yerleşimi (`JdbcProvider`)

Koleksiyon adı `[A-Za-z0-9_]{1,40}` olmalıdır; tablo adı `nitrogen_<koleksiyon>` (küçük harf).

```sql
CREATE TABLE IF NOT EXISTS nitrogen_profiles (
    id         VARCHAR(36) PRIMARY KEY,
    name       VARCHAR(48),
    version    BIGINT NOT NULL,
    updated_at BIGINT NOT NULL,
    data       TEXT            -- MySQL/MariaDB: LONGTEXT, H2: CLOB
) -- MySQL/MariaDB: ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
CREATE INDEX idx_nitrogen_profiles_name ON nitrogen_profiles (name)
```

`data` sütunu `DocumentJson.toJson` çıktısıdır (JSON metni). Sürümlü güncelleme `UPDATE ... WHERE id = ? AND version = ?` ile, `VERSION_ANY` MySQL/MariaDB'de `ON DUPLICATE KEY UPDATE`, SQLite/PostgreSQL'de `ON CONFLICT (id) DO UPDATE`, H2'de ve eski sürücülerde güncelle-yoksa-ekle döngüsüyle yapılır. Eşzamanlı ekleme yarışları 3 kez yeniden denenir. `forEachId` 1000'lik sayfalarla (`WHERE id > ? ORDER BY id LIMIT ?`) gezer.

### MongoDB belgesi

Veritabanı `storage.mongodb.database`, koleksiyon `<collection-prefix><koleksiyon>`; `name` alanında dizin oluşturulur.

```json
{ "_id": "<uuid>", "name": "Oyuncu", "version": 7, "updatedAt": 1757900000000, "data": { ... } }
```

BSON anahtar kısıtları için `data` içindeki anahtarlarda `.` → `．` (U+FF0E) ve baştaki `$` → `＄` (U+FF04) olarak kaçışlanır, okurken geri çevrilir. `Date` epoch ms'ye, `ObjectId` hex string'e dönüşür. Sürümlü kayıt `replaceOne({_id, version})`, ekleme `insertOne` (duplicate key → conflict), `VERSION_ANY` `findOneAndUpdate` + `$inc version` upsert (3 deneme).

### Cassandra tablosu

Keyspace `storage.cassandra.keyspace` (yoksa `CREATE KEYSPACE ... SimpleStrategy, replication_factor`). Tablo adı `nitrogen_<koleksiyon>` sadeleştirilir: küçük harf, `[a-z0-9]` dışı karakterler `_`, en fazla 48 karakter, harfle başlamıyorsa başına `n`.

```sql
CREATE TABLE IF NOT EXISTS nitrogen.nitrogen_profiles (
    id text PRIMARY KEY, name text, version bigint, updated_at bigint, data text)
```

`data` JSON metnidir. Sürüm kontrolü hafif işlemlerle (LWT) yapılır: `INSERT ... IF NOT EXISTS`, `UPDATE ... IF version = ?`; `VERSION_ANY` oku-güncelle döngüsüyle (5 deneme). Tutarlılık `LOCAL_QUORUM` / `LOCAL_SERIAL`; `forEachId` ve `count` tam tarama olduğundan büyük tablolarda yavaştır (sayfa boyutu 500, en az 30 sn zaman aşımı).

### YAML dosya düzeni

Sürüm 2 (mevcut) — `<klasör>/<id>.yml`:

```yaml
format-version: 2
name: Oyuncu
version: 7
updated-at: 1757900000000
data:
  name: Oyuncu
  coins: 1250
  experience: 9800
  credits: 7
  vipRankColor: '&6'
  lastBonusClaim: 1700000000000
  bonusStreak: 3
  achievements:
  - KATIL_KIM_KILLS_10
  achievementDates:
    KATIL_KIM_KILLS_10: 1700000001000
  cosmetics:
  - sapka_altin
  activeCosmetics:
    SAPKA: sapka_altin
  ignored:
  - 11111111-2222-3333-4444-555555555555
  preferences:
    sohbet: 'false'
  stats:
    'KATIL_KIM:kills': 5
```

Yazma sırası: `<id>.yml.tmp` yazılıp `fsync` edilir → mevcut `<id>.yml` varsa `<id>.yml.bak` olarak kopyalanır → tmp atomik olarak `<id>.yml` üzerine taşınır. Kimlik doğrulaması: boş, `.` ile başlayan, `/`, `\`, `..` içeren veya 200 karakterden uzun kimlikler reddedilir. Belge başına 64 şeritli kilit ile eşzamanlı erişim korunur.

Sürüm 1 uyumluluğu: `format-version` anahtarı olmayan dosyalar eski `YamlProfileRepository` düzeni sayılır (`name`, `coins`, `experience`, `credits`, `vip-rank-color`, `last-bonus-claim`, `bonus-streak`, `achievements`, `achievement-dates`, `cosmetics`, `cosmetics-active`, `ignored`, `preferences`, `stats`). Okunurken `version = 1` ve `updated-at = dosya değişim zamanı` ile dönüştürülür; iç içe bölümler `a.b` biçiminde düzleştirilir, `stats` anahtarlarındaki `-` ayırıcı `:` olur (`KATIL_KIM-kills` → `KATIL_KIM:kills`). İlk kayıtta dosya v2 biçimine geçer. Yeniden bir şey yapmak gerekmez; eski `profiles/` klasörü olduğu gibi çalışır.

---

## Dayanıklılık davranışı

### Giriş akışı

1. `AsyncPlayerPreLoginEvent` (`PlayerListener`, LOWEST) → `ProfileManager.handlePreLogin`: profil IO thread'inde yüklenir, en fazla **8 sn** beklenir. Kaynaklar sırasıyla: kuyrukta bekleyen anlık görüntü (`PENDING`), sağlayıcı (`PROVIDER`), `import-from` sağlayıcısı (`IMPORTED`), hiçbiri yoksa yeni profil (`NEW`). Sağlayıcı hata verirse yerel önbellek (`CACHE`) denenir.
2. Zaman aşımı, yükleme hatası veya (önbellek de yoksa) bulunamama → giriş `&cVeri sunucusuna şu anda ulaşılamıyor, lütfen az sonra tekrar dene.` mesajıyla reddedilir; log `Giriş reddedildi (isim / uuid): <sebep>`.
3. Kaynak `CACHE` ve politika `deny-login` ise giriş reddedilir; politika `provisional` ise oyuncu `isProvisional() = true` profille girer ve 2 sn sonra `&eProfilin geçici olarak yüklendi; veri sunucusu geri geldiğinde eşitlenecek.` mesajını görür.
4. `PlayerJoinEvent` (LOWEST) → `handleJoin`: ön yüklenen sonuç kullanılır; yoksa (ör. eklenti çalışırken yeniden yüklendi) 5 sn'lik senkron yükleme denenir, o da başarısızsa oyuncu bir sonraki tick atılır. Bekleyen ön yüklemeler 60 sn sonra temizlenir.

Provisional bayrağı, profilin ilk başarılı kaydında (`SAVED`/`FORCED`) veya outbox'taki kaydı aynı sürümle sağlayıcıya ulaştığında kalkar.

### Yeniden giriş yarışı

Oyuncu çıkınca profili `LIVE` kaynağıyla kuyruğa girer ve bellekten silinir. Aynı oyuncu kayıt daha sağlayıcıya ulaşmadan tekrar girerse `WriteQueue.pendingFor(collection, id)` kuyruktaki **en yeni** anlık görüntüyü döndürür; giriş bu kopya (`PENDING`) ile yapılır, bayat bir sağlayıcı okuması yaşanmaz. Zincirdeki bir kayıt kalıcı olunca sonraki kaydın `expectedVersion` değeri otomatik ileri alınır (aynı temel sürümden türeyen kayıtlar için), böylece çıkış + tekrar giriş + otomatik kayıt dizisi çakışma üretmez.

### Yazma kaynakları ve otomatik kayıt

| `WriteOrigin` | Ne zaman | Çakışmada |
|---|---|---|
| `LIVE` | Oyuncu çıkışı. | Canlı kazanır (zorla yazılır). |
| `AUTOSAVE` | `autosave-seconds` periyodu; parmak izi değişmemişse kayıt atlanır. | Canlı kazanır. |
| `SHUTDOWN` | `saveAll()` (eklenti kapanışı). | Canlı kazanır. |
| `OFFLINE` | `NitrogenAPI.saveOfflineProfile` (oyuncu online değilken). | Atılır. |
| `OUTBOX` | Outbox yeniden gönderimi. | Atılır. |
| `MIGRATION` | `import-from` ile içe aktarma. | Atılır. |

### Yeniden deneme, geri çekilme ve outbox

- Yazma: geçici (`StorageException.isTransient()`) hatada `retry-attempts` (6) kez, gecikme `base-delay-ms` (500 ms) ile başlayıp her denemede ikiye katlanarak `max-delay-ms` (30 sn) tavanına kadar, üstüne %25'e kadar rastgele jitter. Log: `Kayıt yazılamadı (profiles/<id>), 1000 ms sonra yeniden denenecek (2/6): ...`. Kuyruk boşaltılırken (`flush`) gecikme 250 ms'e kısılır.
- Denemeler tükenince veya hata kalıcıysa kayıt outbox'a yazılır: `Kayıt sağlayıcıya yazılamadı, outbox'a alındı (profiles/<id>)`. Outbox dosyası: `storage/outbox/<koleksiyon>/<id>.json` — belge zarfı + `expectedVersion`, `origin`, `storedAt`; belge başına tek dosya, en yeni kayıt öncekinin üzerine yazar.
- Outbox yeniden gönderimi: sağlık `DOWN`'dan çıkınca otomatik, açılışta sağlayıcı açıksa ve `/nitrogen storage replay` ile. Kuyrukta zaten bekleyen belgeler atlanır; başarıyla yazılan dosya (o arada değişmediyse) silinir. Okunamayan dosya `.corrupt-<zaman>` uzantısıyla yerinde bırakılır.
- Outbox'a da yazılamazsa: çalışırken kayıt kuyrukta tutulup `max-delay-ms` sonra yeniden denenir; kapanışta `VERİ KAYBI RİSKİ: kapanışta kayıt hiçbir yere yazılamadı (...)` logu belge içeriğiyle basılır.
- Okuma: `DocumentStore` geçici hatada 150 ms arayla 2 deneme yapar; sağlık `DOWN` iken önce yeniden açma/ping denenir, başarısızsa sağlayıcıya gidilmeden geçici hata döner (profil yüklemeleri önbelleğe düşer).

### Çakışma politikası

Sağlayıcı `CONFLICT` döndürdüğünde `ConflictResolver` devreye girer:

- Kaynak `LIVE`/`AUTOSAVE`/`SHUTDOWN` (**canlı kazanır**): sağlayıcıdaki kopya `storage/conflicts/<koleksiyon>/<id>-<zaman>.json` dosyasına (`side: provider`) dökülür, kayıt `VERSION_ANY` ile zorla yazılır (`FORCED`), `forcedWrites` sayacı artar. Log: `Sürüm çakışması (profiles/<id>): beklenen 3, sağlayıcıdaki 5. Canlı profil kazandı; sağlayıcıdaki eski kayıt conflicts/ altına yedeklendi.`
- Kaynak `OFFLINE`/`OUTBOX`/`MIGRATION`: bizim kopya dökülür (`side: ours`), kayıt atılır (`DROPPED`), `droppedWrites` artar. Log: `... atıldı ve conflicts/ altına yazıldı.`

Döküm dosyaları `origin`, `expectedVersion`, `providerVersion`, `dumpedAt` alanlarını taşır; elle inceleyip gerekirse `saveOfflineProfile` ile geri yazılabilir.

### Yerel önbellek

`storage/cache/<koleksiyon>/<id>.json` her başarılı okuma ve kalıcı yazmadan sonra güncellenir, silmede silinir. Yalnızca sağlayıcı okuması başarısız olduğunda kullanılır; buradan yüklenen profil `provisional` işaretlenir ve `Profil yerel önbellekten geçici olarak yüklendi (<id>, sürüm N).` uyarısı basılır.

### Bozuk YAML karantinası

Ana dosya çözümlenemezse: `.yml.bak` sağlamsa ondan kurtarılır (`Bozuk belge yedekten kurtarıldı`), bozuk ana dosya `storage/corrupt/<id>-<zaman>.yml` olarak taşınır ve yedek ana dosyaya kopyalanır. Yedek de yoksa/bozuksa bozuk dosya karantinaya **kopyalanır**, `.yml.bak` olarak yerinde bırakılır ve kalıcı hata fırlatılır (`Belge dosyası bozuk, karantinaya alındı`); oyuncuya boş profil verilmez, giriş reddedilir (ya da önbellek varsa provisional). Yarım kalmış yazmalar (`.yml.tmp` var, `.yml` yok) açılışta tamamlanır. Bozuk kayıt üzerine yeni bir kayıt gelirse `Bozuk belge üzerine yeni sürüm yazılıyor` uyarısıyla yazılır.

### Kapanış

`WriteQueue.flush(10000)` bekleyen kayıtları hızlandırılmış gecikmeyle boşaltır; süre yetmezse `Kapanışta kuyruk 10 sn içinde boşaltılamadı (N/M kaldı), kalanlar outbox'a taşınıyor.` logu ve outbox. Çalışmakta olan görevlerin bitmesi için 1 sn daha beklenir.

---

## Sağlık durumları

`StorageHealth` (10 sn'de bir `ping`, 8 sn zaman aşımı):

| Durum | Koşul | Etkisi |
|---|---|---|
| `UP` | Sağlayıcı açık, son ping başarılı, kuyruk ≤ 100, outbox boş, son 30 sn'de hata yok. | Normal çalışma. |
| `DEGRADED` | Açık ve ping başarılı ama kuyruk > 100 veya outbox'ta dosya var veya son 30 sn'de geçici hata görüldü. | Okuma/yazma devam eder; `/nitrogen storage status` turuncu gösterir. |
| `DOWN` | Sağlayıcı açılamadı, ping başarısız veya 3 ardışık geçici hata. | Okumalar önce yeniden açmayı dener, olmazsa önbelleğe düşer; yazmalar kuyrukta bekler/outbox'a gider; `replay` reddedilir; giriş politikası devreye girer. |

Geçişler loglanır: `Depolama bağlantısı KOPTU (<tip>): <sebep>. Kayıtlar kuyrukta bekletilecek, gerekirse outbox'a alınacak.`, `Depolama bağlantısı geri geldi, N bekleyen kayıt yeniden gönderiliyor.`, `Depolama durumu UP -> DEGRADED (...)`. Kalıcı (transient olmayan) hatalar sayacı artırmaz, yalnızca "son hata" olarak kaydedilir.

Programatik erişim: `NitrogenAPI.getStorageHealth()` → `HealthSnapshot` (`getState()`, `isAvailable()`, `getQueueSize()`, `getOutboxSize()`, `getLastErrorMessage()`, `getAverageLatencyMs()` vb.). `StorageService.isAvailable()` = başlatıldı ve `DOWN` değil.

---

## Komutlar

Tümü `/nitrogen storage <alt komut>` altında, `nitrogen.management` izni ister. Depolama servisi çalışmıyorsa `Depolama servisi çalışmıyor.` döner.

### `status`

Anlık sağlık ve sayaçlar:

```
Depolama » Durum: UP (12 dk 3 sn süredir)
Sağlayıcı: mysql (açık)
Harici sağlayıcı jar'ları: yok
Son başarı: 2 sn önce
Son hata: hiç
Ardışık hata: 0
Okuma / yazma: 120 / 340 (başarısız: 0, zorlanan: 0, atılan: 0)
Ortalama gecikme: 3.4 ms
Kuyruk: 0 | Outbox: 0
Otomatik kayıt: 60 sn | Politika: PROVISIONAL | İçe aktarma: yok
Yüklü profil: 14
```

YAML'a geri dönülmüşse sağlayıcı satırında `(yapılandırılan: mysql, geri dönüldü)` görünür.

### `test`

`selftest` koleksiyonunda sağlayıcıya **doğrudan** (kuyruk ve sağlık katmanını atlayarak) altı adımlı bir tur atar ve sonucu gönderene basar:

```
✔ Yeni kayıt (VERSION_NEW) (4 ms) PASS
✔ Okuma (2 ms) PASS
✔ Sürümlü güncelleme (beklenen 1) (3 ms) PASS
✔ Bayat yazma çakışmalı (beklenen 1) (2 ms) PASS
✔ Varlık kontrolü (1 ms) PASS
✔ Silme (2 ms) PASS
Kendi kendine test BAŞARILI (sağlayıcı: mysql)
```

Başarısız adımlar `✘ ... FAIL — <sebep>` ile, özet `Kendi kendine test BAŞARISIZ (N hata)` ile gösterilir. Yeni bir veritabanına geçmeden önce çalıştır.

### `flush`

Yazma kuyruğunu hemen boşaltmaya çalışır (5 sn): `Kuyruk boşaltıldı.` veya `Kuyruk 5 sn içinde boşaltılamadı, kalan: N`.

### `replay`

Outbox dosyalarını kuyruğa geri alır: `Outbox'tan N kayıt kuyruğa alındı.` Depolama `DOWN` ise `Depolama şu anda erişilemez durumda, outbox yeniden gönderilemez (N kayıt bekliyor).`

### `providers`

Kayıtlı fabrikalar (`[aktif]` işaretli) ve bulunan harici jar'lar:

```
Depolama » Kayıtlı sağlayıcılar:
 - yaml (YAML dosyaları)
 - sqlite (SQLite)
 - h2 (H2)
 - mysql (MySQL) [aktif]
 - mariadb (MariaDB)
 - postgresql (PostgreSQL)
 - mongodb (MongoDB)
Harici jar'lar: Nitrogen-MongoDB.jar
```

### `migrate ...`

Bkz. [Depolar arası taşıma](#depolar-arası-taşıma-migration).

---

## Depolar arası taşıma (migration)

İki yol vardır: **toplu komut** (tüm kayıtları bir kerede kopyalar) ve **tembel içe aktarma** (`import-from`; oyuncu giriş yaptıkça taşır). Her ikisi de aynı belge modelini kullandığından sağlayıcılar arasında veri kaybı olmadan geçiş yapılabilir.

### Toplu komut

Kod: `storage.migration.MigrationService` (Bukkit'siz çekirdek), `MigrationCommand` (komut adaptörü), `MigrationReport`.

```
/nitrogen storage migrate <kaynak> <hedef> [--dry-run] [--force] [--collection <ad>] [--retry <rapor.yml>]
/nitrogen storage migrate durum
/nitrogen storage migrate iptal
/nitrogen storage migrate raporlar
```

- `<kaynak>` ve `<hedef>` config'teki sağlayıcı tipleridir (`yaml`, `mysql`, `mongodb`...); farklı olmalıdır ve her ikisinin bölümü `storage.*` altında dolu olmalıdır. Aktif sağlayıcı kaynak ya da hedef ise mevcut bağlantı yeniden kullanılır (aynı SQLite dosyası ikinci kez açılmaz), diğeri komut süresince ayrıca açılıp sonunda kapatılır. Bilinmeyen tip girilirse bilinen tipler listelenir.
- Kayıtlar **koşulsuz üzerine yazılarak** (`VERSION_ANY`) kopyalanır; komut idempotenttir, tekrar çalıştırmak güvenlidir (hedefteki sürüm numarası her çalıştırmada artar).
- Varsayılan koleksiyon `profiles`'tır; başka bir koleksiyon için `--collection <ad>` verilir (ör. `loncalar`, `lonca_uyelik`, `lonca_isim`). Her koleksiyon ayrı komutla taşınır.
- `profiles` koleksiyonunda her belge `ProfileCodec.fromDocument` ile doğrulanır; çözümlenemeyen belge kopyalanmaz ve `basarisiz` olarak raporlanır.
- **Online oyuncuların** profilleri atlanır (bellekteki canlı kopya kaynaktan daha yenidir); `--force` ile yine de kopyalanır. Hedef aktif sağlayıcıysa ve oyuncu varsa komut `--force` olmadan reddedilir; kaynak aktif sağlayıcıysa uyarı verilir ve önce yazma kuyruğu 5 sn boşaltılır.
- `--dry-run` hiçbir şey yazmadan sayıları ve olası hataları raporlar.
- Kayıt bazlı hatalar taşımayı durdurmaz; geçici hatalar 3 kez (200/400/800 ms) yeniden denenir, sonra `basarisiz` olarak raporlanır. Her 100 kayıtta ilerleme mesajı gönderene ve konsola gider; komut `Nitrogen-Storage-Migration` adlı ayrı bir iş parçacığında çalışır, aynı anda tek taşıma yürür.
- Rapor `plugins/Nitrogen/storage/migrations/<yyyy-MM-dd_HH-mm-ss>.yml` dosyasına yazılır:

```yaml
kaynak: yaml
hedef: mysql
koleksiyon: profiles
kuru-calisma: false
zorla: false
iptal-edildi: false
baslangic: 2026-09-15T02:40:11
bitis: 2026-09-15T02:40:19
sure-ms: 8123
toplam: 1834
basarili: 1830
basarisiz: 3
atlanan: 1
basarisiz-kayitlar:
  8f1c...-...: "hedefe yazılamadı: Connection refused"
atlanan-kayitlar:
  2b77...-...: "oyuncu çevrimiçi"
```

- `--retry <rapor.yml>` yalnızca ilgili rapordaki başarısız kimlikleri yeniden dener; kaynak/hedef/koleksiyon verilmezse rapordakiler kullanılır (rapor `tekrar-raporu` alanıyla işaretlenir).
- `durum` süren taşımanın ilerlemesini, `iptal` durdurulmasını (kayıtlar arasında), `raporlar` en yeni 10 rapor dosyasını listeler. İngilizce eşdeğerleri (`status`, `cancel`, `reports`) de kabul edilir.

Adım adım:

1. Hedef sağlayıcının bölümünü `config.yml`'e yaz (tip henüz değiştirilmez), sunucuyu yeniden başlat ya da `/nitrogen storage test` ile mevcut sağlayıcının sağlıklı olduğunu doğrula.
2. Tercihen sunucu boşken `/nitrogen storage migrate <eski> <yeni> --dry-run` çalıştırıp sayıları kontrol et.
3. `/nitrogen storage migrate <eski> <yeni>` çalıştır; `durum` ile izle, bitince raporu incele.
4. `basarisiz > 0` ise sebepleri gider (bozuk YAML karantinası vb.) ve `--retry <rapor.yml>` ile tamamla.
5. `storage.type` değerini hedef tipe çek, sunucuyu yeniden başlat, `/nitrogen storage status` ile sağlayıcının hedef olduğunu doğrula.
6. Eski depoyu bir süre yedek olarak sakla.

### Tembel içe aktarma (`import-from`)

Kod: `StorageSettings.getImportFrom`, `StorageService.openImport`, `ProfileStore.importBlocking`.

1. Yeni sağlayıcının bölümünü doldur, `storage.type: <yeni>`, `storage.import-from: <eski>` yap ve sunucuyu yeniden başlat. Açılışta `İçe aktarma sağlayıcısı açıldı: <eski>` logu görülmeli.
2. Giriş yapan her oyuncu için: profil yeni depoda yoksa eski depodan okunur, `Profil eski depodan içe aktarıldı: <uuid>` loglanır, `MIGRATION` kaynağıyla yeni depoya yazılır (`version = NEW`) ve oyuncu `IMPORTED` kaynağıyla girer. Sonraki girişlerde yalnızca yeni depo kullanılır.
3. Yalnızca `profiles` koleksiyonu içe aktarılır; diğer koleksiyonlar için toplu komut gerekir.
4. Uzun süredir girmeyen oyuncular için toplu komutu çalıştır (kalanları taşır; mevcut kayıtlar üzerine yazılır).
5. Bitince `storage.import-from: none` yap ve yeniden başlat.

Dikkat: `import-from` açıkken eski sağlayıcı erişilemezse yeni depoda **bulunmayan** profillerin yüklemesi geçici hatayla başarısız olur (önbellek yoksa giriş reddedilir). Eski depo kalıcı olarak kapatıldıysa `import-from`'u mutlaka kaldır.

---

## Lonca kalıcılığı

Kod: `lonca.LoncaManager` (komut/menü mantığı), `LoncaCodec` (belge dönüşümü), `LoncaCache` (bellek içi önbellek), `LoncaStore` (DocumentStore üzerinden okuma/yazma), `LoncaRedisImport` (tek seferlik içe aktarma), `LoncaJoinListener` (girişte üyelik kontrolü).

Lonca verisi Redis anahtarlarından depolama koleksiyonlarına taşınmıştır:

| Koleksiyon | İçerik |
|---|---|
| `loncalar` | Lonca kaydı: `name`, `leader` (UUID), `members` (UUID → isim, ekleme sırası korunur), `createdAt`, `updatedAt`, dağıtılırken geçici `disbanded` bayrağı. Belge kimliği lonca kimliği; sürüm numarası çakışma kontrolü için kullanılır. |
| `lonca_uyelik` | Oyuncu → lonca eşlemesi (`loncaId`), belge kimliği oyuncu UUID'si. |
| `lonca_isim` | Küçük harfli lonca adı → lonca kimliği (`loncaId`, `name`); benzersizlik ve arama için. |
| `lonca_meta` | `redis-import` belgesi: Redis'ten içe aktarmanın yapıldığını işaretler. |

- Redis yalnızca sunucular arası **pub/sub kanalı** (`nitrogen:lonca`; INVITE/JOIN/LEAVE/LEADER/KICK/DENY/DISBAND ve yeni CREATE olayı) ve 60 saniyelik **davet anahtarları** için kalır. Eski sunucular bilinmeyen CREATE olayını yok sayar.
- Ana thread'deki tüm okumalar bellek içi önbellekten yapılır. Açılışta tüm loncalar arka planda yüklenir (`N lonca yüklendi.`); yükleme başarısızsa 30 sn'de bir yeniden denenir ve bu sırada lonca komutları `&cLonca verisi şu anda yüklenemiyor, lütfen az sonra tekrar dene.` yanıtını verir. Önbellek **5 dakikada bir** tamamen yenilenir; bir olay geldiğinde ilgili lonca 2 sn sonra yeniden yüklenir; giriş yapan oyuncunun üyeliği önbellekte yoksa `lonca_uyelik` belgesi okunur.
- Yazmalar önce önbelleği günceller, sonra `loncalar` belgesi sürüm kontrolüyle (`LIVE` kaynağı; çakışmada canlı kazanır) ve üyelik/isim belgeleri koşulsuz olarak kuyruğa verilir; dağıtma önce `disbanded` bayrağını yazar, kalıcı olunca belgeleri siler.
- Koleksiyon boşsa, Redis erişilebilirse ve işaret belgesi yoksa eski `nitrogen:lonca:*` anahtarlarından **tek seferlik otomatik içe aktarma** yapılır (`Redis'ten N lonca içe aktarıldı.`); eski anahtarlar silinmez.
- SQL sağlayıcılarında bu koleksiyonlar `nitrogen_loncalar`, `nitrogen_lonca_uyelik`, `nitrogen_lonca_isim`, `nitrogen_lonca_meta` tablolarıdır; YAML'da `plugins/Nitrogen/` altında aynı adlı klasörlerdir.
- Bilinen sınırlar: iki sunucu aynı saniyelerde aynı loncayı değiştirirse son yazan kazanır ve diğer sunucu bir sonraki yenilemede uyumlanır; Redis kapalıyken sunucular en geç 5 dakikada birbirini görür.

---

## Geliştiriciler için

### NitrogenAPI erişimi

| Metod | Açıklama |
|---|---|
| `NitrogenAPI.getStorage()` | `StorageService`; koleksiyon için `store("ad")`. |
| `NitrogenAPI.getStorageHealth()` | `HealthSnapshot` (null olabilir). |
| `NitrogenAPI.loadOfflineProfileAsync(UUID)` | Tercih edilen offline profil yükleme (IO thread'inde tamamlanır). |
| `NitrogenAPI.loadOfflineProfile(UUID)` | Senkron, en fazla 3 sn; ana thread'de kullanma. |
| `NitrogenAPI.saveOfflineProfile(Profile)` | `OFFLINE` kaynaklı kayıt; oyuncu online ise yok sayılır. |

### Yeni koleksiyon kullanma

```java
DocumentStore store = NitrogenAPI.getStorage().store("lonca_puan");

store.load(loncaId).thenAccept(document -> {
    long puan = document == null ? 0L : DocumentValues.getLong(document.getData(), "puan", 0L);
    Bukkit.getScheduler().runTask(plugin, () -> goster(puan));
});

Map<String, Object> data = new LinkedHashMap<>();
data.put("puan", 1500L);
data.put("uyeler", Arrays.asList("a", "b"));
StorageDocument snapshot = new StorageDocument(loncaId, "Şahinler", StorageDocument.VERSION_ANY, 0L, data);
store.save(snapshot, StorageDocument.VERSION_ANY, WriteOrigin.LIVE,
        (outcome, version) -> plugin.getLogger().info("Lonca kaydı: " + outcome + " v" + version));
```

`DocumentStore` API'si:

| Metod | Açıklama |
|---|---|
| `load(id)` | `CompletableFuture<StorageDocument>`; kuyrukta bekleyen anlık görüntü varsa hemen onu döner; yoksa sağlayıcıdan okur (yoksa `null`), başarılı okumayı önbelleğe yazar. |
| `loadNow(id)` | Aynı işin bloklayan hali (`StorageException` fırlatır); yalnızca IO thread'inde kullan. |
| `exists(id)` / `delete(id)` / `ids()` / `count()` | `CompletableFuture<Boolean/List<String>/Long>`. |
| `save(snapshot, expectedVersion, origin, callback)` | Kuyruğa alır. `expectedVersion` sürüm semantiğine göre (`VERSION_NEW`, son okunan sürüm veya `VERSION_ANY`); `origin` çakışma politikasını belirler; `callback` `SaveOutcome` (`SAVED`, `FORCED`, `DROPPED`, `OUTBOXED`) ve yeni sürümle çağrılır (IO thread'inde). |
| `pending(id)` / `cached(id)` | Kuyruktaki son anlık görüntü / yerel önbellek kopyası (senkron, blok yok). |

Kurallar:

- Koleksiyon adları SQL için `[A-Za-z0-9_]{1,40}` ile sınırlıdır; Cassandra adı sadeleştirir, YAML klasör açar. Tüm sağlayıcılarda çalışması için küçük harf ve alt çizgi kullan.
- `save`'e verdiğin `StorageDocument` kuyrukta kalır; sonradan değiştirme, her kayıt için `copy()` ya da yeni nesne üret.
- Future'lar ve callback'ler IO thread'inde tamamlanır; Bukkit API'sine dokunmadan önce `Bukkit.getScheduler().runTask` ile ana thread'e geç.
- Okuma-değiştir-yaz döngüsü için son okunan `getVersion()` değerini `expectedVersion` olarak ver; `DROPPED` gelirse yeniden oku. Çakışması önemsiz sayaçlar için `VERSION_ANY`.
- `DocumentValues` (`getString/getLong/getInt/getDouble/getBoolean/getStringList/getMap/getLongMap/getStringMap`) tip güvenli okuma sağlar.

### Yeni sağlayıcı yazma

SPI paketi `son.bahar.nitrogen.storage.spi`:

| Tip | Sorumluluk |
|---|---|
| `StorageProviderFactory` | `getType()` (küçük harf tip adı, config bölümü adı), `getDisplayName()`, `create(ProviderConfig)`. Yapılandırma hatası için `StorageException(..., false)`. |
| `StorageProvider` | `getType()`, `open()` (bağlantı kur; ulaşılamıyorsa transient, geçersiz ayar ise permanent hata), `close()` (idempotent), `ping()` (hızlı, exception fırlatmaz), `collection(name)` (örnekleri önbellekle). |
| `DocumentCollection` | `getName()`, `load(id)` (yoksa `null`), `save(document, expectedVersion)` → `SaveResult.saved(yeniSürüm)` / `SaveResult.conflict(mevcutSürüm)`, `delete(id)` (vardıysa `true`), `exists(id)`, `forEachId(Consumer)`, `count()`. |
| `StorageException` | `transientOf(...)` (ağ, zaman aşımı, kilit — yeniden denenir) ve `permanentOf(...)` (şema, kimlik doğrulama, bozuk veri — hemen outbox). Sağlık makinesi yalnızca transient hataları sayar. |
| `ProviderConfig` | `storage.<tip>.*` bölümünün görünümü: `getString/getInt/getLong/getDouble/getBoolean/getStringList/getSection`, `get("a.b")` nokta yolu, `resolve(yol)` (göreli → `plugins/Nitrogen/`), `getStorageFolder()`, `getLogger()`. |
| `StorageDocument`, `DocumentJson`, `DocumentValues` | Belge zarfı, JSON/normalizasyon yardımcıları. |

`save` uygulaması yukarıdaki [sürüm semantiği tablosuna](#veri-modeli) birebir uymalı ve `SAVED` dönerken belgenin `version`/`updatedAt` alanlarını güncellemesi beklenir. `VERSION_ANY` için `mevcut + 1`, kayıt yoksa `1` üretilir; `n > 0` kayıt yokken ekleme yapar (`n + 1`).

Yerleşik ek için fabrikayı `StorageService.start` içinde `registry.register(...)` ile kaydet. Harici jar için:

1. `Nitrogen` bağımlılığı `provided`, sürücü bağımlılıkları jar'a shade edilir (relocation önerilir; bkz. `storage-mongodb/pom.xml`).
2. `src/main/resources/META-INF/services/son.bahar.nitrogen.storage.spi.StorageProviderFactory` dosyasına fabrika sınıfının tam adı yazılır (shade'de `ServicesResourceTransformer` kullan).
3. Jar `plugins/Nitrogen/storage/providers/` altına konur.
4. Sınıf yükleme kuralını hatırla: yalnızca `son.bahar.nitrogen.`, `java.`, `javax.`, `org.bukkit.`, `com.google.gson.`, `net.minecraft.` üstten gelir; bunun dışındaki her sınıfı (slf4j dahil) jar'ın kendisi taşımalıdır. Thread context classloader'a ihtiyaç duyan sürücüler için `open()` içinde geçici olarak `Thread.currentThread().setContextClassLoader(getClass().getClassLoader())` ayarla (Cassandra sağlayıcısı böyle yapar).
5. `ProviderConfig` sana `storage.<tip>` bölümünü verir; varsayılanları `StorageSettings.defaults()` içine ekleyerek config.yml'e otomatik yazdırabilirsin.

---

## Standalone test araçları

Sunucu olmadan, Java 8 ile çalışan duman testleri. Aşağıdaki komutlar proje kökünden (`C:\inetpub\Nitrogen`) ve Windows için verilmiştir; `Nitrogen.jar` shaded olduğundan HikariCP/slf4j ayrıca gerekmez, Gson ve SnakeYAML `SOSpigot.jar` içinden gelir.

```bat
set JAVA8="C:\Users\Administrator\.jdks\corretto-1.8.0_504\bin\java.exe"
set CP=core\target\Nitrogen.jar;libs\SOSpigot.jar
```

| Araç | Ne yapar | Komut |
|---|---|---|
| `StorageSmokeMain` | Sağlayıcı CRUD + sürüm senaryosu (YAML'da ek olarak v1 uyumluluk ve bozuk dosya senaryoları), ardından bellek içi `WriteQueue` senaryosu (geri çekilme, pendingFor, zincirleme, LIVE/OFFLINE çakışması, outbox, kapanış). `queue` tipi yalnızca kuyruk senaryosunu çalıştırır. Geçici klasörde çalışır. | `%JAVA8% -cp %CP% son.bahar.nitrogen.storage.tool.StorageSmokeMain yaml folder=C:\tmp\profiller`<br>`%JAVA8% -cp %CP% son.bahar.nitrogen.storage.tool.StorageSmokeMain sqlite file=C:\tmp\smoke.db`<br>`%JAVA8% -cp %CP% son.bahar.nitrogen.storage.tool.StorageSmokeMain mysql host=localhost port=3306 database=nitrogen username=root password=gizli`<br>`%JAVA8% -cp %CP% son.bahar.nitrogen.storage.tool.StorageSmokeMain queue` |
| `JdbcSmokeMain` | Yalnızca JDBC sağlayıcıları için sürüm semantiği turu (NEW, expected, stale, ANY, missing-expected, exists, forEachId, delete, ping, close). `data-folder=` ile sürücü önbellek klasörü sabitlenebilir. | `%JAVA8% -cp %CP% son.bahar.nitrogen.storage.provider.jdbc.JdbcSmokeMain sqlite file=C:\tmp\jdbc.db`<br>`%JAVA8% -cp %CP% son.bahar.nitrogen.storage.provider.jdbc.JdbcSmokeMain postgresql host=localhost port=5432 database=nitrogen username=postgres password=gizli data-folder=C:\tmp\nitrogen` |
| `MongoSmokeMain` | MongoDB sağlayıcısı: nokta/dolar anahtar kaçışı, Türkçe metin, sürüm semantiği. Koleksiyon öneki `smoke_`. | `%JAVA8% -cp storage-mongodb\target\Nitrogen-MongoDB.jar;%CP% son.bahar.nitrogen.storage.mongodb.MongoSmokeMain mongodb://localhost:27017 nitrogen_smoke` |
| `CassandraSmokeMain` | Cassandra sağlayıcısı: keyspace/tablo oluşturma, LWT sürüm semantiği, tarama. | `%JAVA8% -cp storage-cassandra\target\Nitrogen-Cassandra.jar;%CP% son.bahar.nitrogen.storage.cassandra.CassandraSmokeMain localhost 9042 datacenter1 nitrogen_smoke` |
| `MigrationSmokeMain` | Taşıma akışı duman testi: geçici klasörde 25 sentetik profil yaml → sqlite kopyalanır; idempotent tekrar, `--dry-run`, çevrimiçi atlama/`--force`, geçici hata + `--retry`, iptal ve rapor listesi doğrulanır (15 adım). İsteğe bağlı `driver-jar=<sqlite-jdbc.jar>` (çevrimdışı) ve `keep=true` (geçici klasörü silme). | `%JAVA8% -Dfile.encoding=UTF-8 -cp %CP% son.bahar.nitrogen.storage.migration.MigrationSmokeMain` |

Çıkış kodu: tüm adımlar geçtiyse `0`, aksi halde `1`; argüman hatasında `2`. `StorageSmokeMain` ve `JdbcSmokeMain` argümanları `anahtar=değer` biçimindedir ve doğrudan `storage.<tip>.*` anahtarlarına karşılık gelir (`driver-download=false`, `driver-jar=C:\...\mysql.jar` gibi). Ağ tabanlı tiplerde JDBC sürücüsü geçici klasöre indirilir; çevrimdışı makinede `driver-jar` ver.

---

## Sorun giderme

| Log / belirti | Anlamı | Ne yapmalı |
|---|---|---|
| `Depolama sağlayıcısı oluşturulamadı (<tip>) ... YAML sağlayıcısına GERİ DÖNÜLÜYOR` | Tip bilinmiyor (harici jar eksik) veya bölüm geçersiz. | `storage/providers/` içindeki jar'ı ve `storage.type` yazımını kontrol et; `/nitrogen storage providers`. |
| `Depolama sağlayıcısı 10 sn içinde açılamadı (...)` / `Depolama sağlayıcısı açılamadı (...): ... 10 sn sonra yeniden denenecek.` | Veritabanına ulaşılamıyor. | Bağlantı bilgileri, ağ, kullanıcı izinleri; sunucu bu sırada `DOWN` çalışır, girişler politikaya göre işlenir. |
| `JDBC sürücüsü indirilemedi (...)` + `Eski yerleşik JDBC sürücüsü kullanılıyor` | Maven Central'a erişim yok. | Sürücüyü elle `storage/drivers/` altına koy ya da `driver-jar` ver; `driver-download: false` ile denemeyi kapat. |
| `Bilinmeyen JDBC depolama türü` | `DriverCatalog`'da olmayan tip ile `DriverLoader` kuruldu. | `storage.type` yazımı. |
| `Giriş reddedildi (isim / uuid): ...` (`prelogin-deny`) | Profil 8 sn içinde yüklenemedi, yükleme hata verdi veya `deny-login` politikası. | `/nitrogen storage status` ile durumu gör; geçici kesintide `unavailable-policy: provisional` düşün. |
| `Profil yerel önbellekten geçici olarak yüklendi` | Sağlayıcı okuması başarısız, önbellek kullanıldı. | Sağlayıcıyı düzelt; profil ilk başarılı kayıtta eşitlenir. |
| `Kayıt sağlayıcıya yazılamadı, outbox'a alındı` / `Profil kaydı outbox'a alındı` | Yeniden denemeler tükendi. | Sağlayıcı dönünce otomatik gönderilir; gerekirse `/nitrogen storage replay`. |
| `Sürüm çakışması (...): Canlı profil kazandı` / `... atıldı ve conflicts/ altına yazıldı` | Aynı belgeye iki farklı kaynaktan yazıldı (iki sunucu, offline düzenleme). | `storage/conflicts/` dökümlerini incele; oyuncu iki sunucuda aynı anda olmamalı. |
| `Belge dosyası bozuk, karantinaya alındı` / `Bozuk belge yedekten kurtarıldı` | YAML dosyası okunamadı. | `storage/corrupt/` içindeki kopyayı düzeltip `profiles/<id>.yml` olarak geri koy (`.yml.bak`'ı kaldır). |
| `Ana iş parçacığı profil yüklemesi için N ms bekledi` (`profile-sync-slow`) | `loadOfflineProfile` ana thread'de bekledi. | Çağrıyı `loadOfflineProfileAsync` ile değiştir. |
| `Depolama sağlık kontrolü 8 sn içinde yanıt vermedi` | Ping takıldı (ağ/kilit). | Veritabanı yükü ve ağ; durum `DOWN`'a düşer, düzelince kendiliğinden döner. |
| `Kapanışta kuyruk 10 sn içinde boşaltılamadı` | Kapanışta sağlayıcı yavaş/kapalı. | Kalanlar outbox'ta; sonraki açılışta gönderilir. |
| `VERİ KAYBI RİSKİ: kapanışta kayıt hiçbir yere yazılamadı` | Disk de yazılamadı. | Log satırındaki belge içeriğini elle geri yükle; disk izin/alanını kontrol et. |
| `Outbox dosyası okunamadı, karantinaya alındı` | `storage/outbox/` içinde bozuk JSON. | `.corrupt-<zaman>` dosyasını incele. |
| `Harici depolama sağlayıcısı yüklenemedi (...)` / `Sağlayıcı jar'ı açılamadı, atlanıyor` | Jar bozuk, eksik bağımlılık veya sınıf yükleme çakışması. | Jar'ın shaded ve `META-INF/services` içerdiğinden emin ol; stack trace'teki sınıf adına bak. |
| `Kuyruk işleyicisi beklenmeyen hata verdi` | Sağlayıcı `StorageException` yerine runtime hata fırlattı. | Sağlayıcı kodunda hatayı `StorageException` ile sar. |
| Aynı uyarı `(+N tekrar)` ile seyrek görünüyor | `StorageLog` hız sınırı (30 sn). | Normal; sayı tekrar adedini gösterir. |
