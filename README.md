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
