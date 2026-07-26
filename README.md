# 3E Akademi — Denetim Zinciri Tanığı

Bu depo, 3E Akademi platformunun **denetim kayıtlarının bütünlük zincirini** dışarıya yayımladığı yerdir.

Burada **hiçbir denetim kaydı, kişisel veri veya müşteri bilgisi yoktur.** Yalnızca hash'ler ve sayılar bulunur. Bir hash, kapsadığı satırlar hakkında hiçbir şey açığa çıkarmaz — bu yüzden yayımlanması güvenlidir, ve tam da bu yüzden işe yarar.

## Bu depo neye yarar

Platformun denetim tabloları append-only'dir ve her partisi bir **Merkle kökü** ile mühürlenir; mühürler birbirine **hash zinciriyle** bağlanır. Bu zincir, *hiçbir kaydın sonradan değiştirilmediğini* gösterir.

Zincirin tek başına gösteremediği şey **ne zaman kurulduğudur** — prensipte tümü bugün baştan üretilmiş olabilir. Bu deponun varlık sebebi budur: zincir hash'leri buraya yayımlandıkça, geçmiş bizim tarafımızdan yeniden yazılamaz hale gelir.

## Tanıklığın dayandığı üç şart

| Şart | Neden |
|---|---|
| Depo **public** | Private depo yalnız GitHub'ın kendi kayıtlarına karşı tanıklık eder; üçüncü bir tarafa bağımsız olarak gösterilemez. |
| `main` üzerinde **force-push ve silme kapalı** | Açık olsaydı geçmişi biz yeniden yazabilirdik ve tüm mekanizma anlamını yitirirdi. Aktif kurallar: `non_fast_forward`, `deletion`. |
| Yazma yetkisi **yalnız bu depoya** | Dar yetkili bir deploy key kullanılır; süresi dolan bir token, tanığın sessizce yazılmaz hale gelmesi demektir. |

⚠ **Commit'in kendi tarihi kanıt değildir.** Commit tarihi yazan tarafın verdiği bir alandır ve geriye alınabilir. Tanıklık değeri, **GitHub'ın push'u ne zaman aldığını kendi tarafında kaydetmesinden** gelir; bu kayıt deponun public etkinlik geçmişinden okunur.

## Dosya düzeni

```
roots/<denetim_tablosu>/<YYYY-AA-GG>.jsonl
```

Her satır bir mühürdür (JSON):

| Alan | Anlamı |
|---|---|
| `audit_table` | Mührün kapsadığı denetim tablosu |
| `epoch_date` | Mührün alındığı gün |
| `from_id` / `to_id` | Partinin kapsadığı satır aralığı (id bazlı) |
| `row_count` | Partideki satır sayısı |
| `merkle_root` | Parti satırlarının Merkle kökü (sha256, hex) |
| `previous_chain_hash` | Bir önceki mührün zincir hash'i (ilk mühürde `null`) |
| `chain_hash` | `sha256(previous_chain_hash ‖ merkle_root)` |
| `hash_algorithm` | Mührün üretildiği **kural adı** — bugün `sha256-utc-iso-v1`. Yalnız hash fonksiyonunu değil, yaprağın nasıl yazıldığını da adlandırır (aşağıya bakınız). Kurallar zincirden kısa ömürlü olabilir; hangi halkanın hangi kural altında üretildiği halkanın kendisinde yazılıdır |

## Doğrulama

**1. Zincirin kendi içinde tutarlılığı** — bu depo tek başına yeter, bize hiçbir şey sormaya gerek yok:

```bash
cat roots/audit_writes/*.jsonl | python3 - <<'PY'
import hashlib, json, sys
prev = None
for line in sys.stdin:
    if not line.strip():
        continue
    seal = json.loads(line)
    if seal["previous_chain_hash"] != prev:
        print("ZINCIR KOPUK:", seal["from_id"], "->", seal["to_id"])
    want = hashlib.sha256(((seal["previous_chain_hash"] or "") + seal["merkle_root"]).encode()).hexdigest()
    if want != seal["chain_hash"]:
        print("HASH TUTMUYOR:", seal["from_id"], "->", seal["to_id"])
    prev = seal["chain_hash"]
print("kontrol bitti")
PY
```

Zincirde bir kopukluk, o noktadan sonra bir şeyin değiştiğini gösterir.

**2. Kayıtların köke uygunluğu** — bu adım denetim kayıtlarına erişim gerektirir (mahkeme/bilirkişi bağlamı).

Kural `sha256-utc-iso-v1`. Yaprak, her satır için:

```
sha256( id ‖ "|" ‖ created_at ‖ "|" ‖ coalesce(tenant_id, "") )
```

`created_at` **UTC'ye çevrilip sabit kalıpla** yazılır: `YYYY-MM-DDTHH24:MI:SS.US` — örneğin `2026-07-26T02:55:20.188807`. Mikrosaniye altı yuvarlama yoktur, ofset yazılmaz, ayırıcı `T`'dir.

> ⚠ Bu ayrıntı kritiktir. PostgreSQL'de bir `timestamptz` doğrudan metne çevrilirse **oturumun `TimeZone` ve `DateStyle` ayarına göre** yazılır; aynı satır `Europe/Istanbul` altında `2026-07-26 05:55:20+03`, `German, DMY` altında `26.07.2026 05:55:20 +03` görünür ve üçü **üç farklı yaprak** üretir. Doğrulayan taraf kendi oturum ayarıyla hesap yaparsa zinciri hatalı biçimde bozuk görür. Bu yüzden kural yazımı sabitler; üretim tarafında da `app.audit_leaf_line()` fonksiyonu aynı kalıbı uygular.

`id` ve `tenant_id` ondalık tam sayı olarak yazılır; `tenant_id` boşsa yerine boş dize konur. Yapraklar **id sırasına göre** dizilir, ikişerli hash'lenerek katlanır. Tek kalan düğüm **yukarı olduğu gibi taşınır** — kopyalanmaz (kopyalamak, iki farklı partinin aynı kökü üretmesine izin veren bilinen bir açıktır).

**3. Yayım zamanı** — bir mührün ne zaman yayımlandığı, deponun public etkinlik geçmişinden okunur. Commit'in kendi tarihine değil, GitHub'ın push kaydına bakılır.

## Geliştirme dönemi

⚠ `hash_algorithm` alanı `sha256-utc-iso-v1` **olmayan** mühürler, sistem geliştirilirken yapılan denemelerden kalmadır. Yaprak kuralı o sırada zaman damgasını oturum ayarına bağlı yazıyordu; kusur üretime çıkılmadan bulunup düzeltildi. Bu mühürler geliştirme verisini kapsar ve **hiçbir delil iddiası taşımaz**. Depo geçmişi geriye dönük silinemediği için burada duruyorlar; kayıt olarak bırakılmaları, geçmişi temizlemeye çalışmaktan daha dürüsttür.

## Sınır

Bu yapı **takdiri delildir**, kesin delil değildir. 5070 sayılı kanunun güvenli elektronik imzaya tanıdığı statü BTK'ya kayıtlı bir elektronik sertifika hizmet sağlayıcısından gelir; bu zincir onu taşımaz. Bilirkişi zinciri yeniden hesaplayıp doğrulayabilir ve kanıt gücü ciddidir; ancak itiraz halinde ispat yükü yayımlayan taraftadır.
