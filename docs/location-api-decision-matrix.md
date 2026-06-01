# Location / Map API Decision Matrix

> Web / モバイルアプリ向け 地図・位置情報 API 選定資料
> 対象プロバイダ: Google Maps Platform / Mapbox / MapTiler / AWS Amazon Location Service
> 最終調査日: 2026-06-01（料金・無料枠・API 名称・保存条件は変動します。意思決定前に必ず各社公式ページで再確認してください。）

価格はすべて **USD**、特記なき場合 **1,000 リクエスト（または 1,000 セッション）あたり** の最小ボリューム帯（base tier）単価です。大口になるほど単価は逓減します。**要確認** はこの資料作成時点で公式情報から確証が取れなかった、または変動が想定される項目です。

---

## 1. Executive Summary（結論）

| 観点 | 結論 |
|---|---|
| **日本の POI / 店舗名 / ビル名検索を重視** | **Google Maps Platform（Places API New）** が実質一択。日本の店舗・施設・建物名の網羅性と鮮度で他社を大きく上回る。単価は高いので無料枠とコスト管理が前提。 |
| **住所ジオコーディングを安く大量に** | **Mapbox Geocoding（Temporary）** が最安級（無料 100,000/月、超過 $0.75/1,000）。保存しないなら圧倒的。保存するなら Permanent（$5/1,000）か AWS / Google を比較。 |
| **地図表示を安く** | **MapTiler**（月額プランで読みやすい）または **AWS GeoMaps**（タイル単価 $0.04/1,000 と最安級）。MapLibre 構成なら MapTiler が自然。 |
| **Google Maps 風の統合検索 UX** | **Google Places API（New）**。Autocomplete → Place Details → Text Search が同一エコシステムで完結。 |
| **Autocomplete を低コストに** | **Mapbox Search Box API**（セッション課金 $3/1,000 sessions、500 無料）。Google はセッション設計次第で実質無料化も可能だが設計が複雑。AWS は per-request（Label $0.20/1,000）。 |
| **MapLibre / OSS 寄り構成** | **MapTiler**（MapLibre 主要スポンサー、SDK は MapLibre ベース）。 |
| **AWS 環境に統合** | **Amazon Location Service**。IAM / CloudWatch / コスト管理が AWS に統合。ただし日本 POI は弱く、HERE 利用時は日本データの保存禁止に注意。 |
| **請求事故を避けて個人学習** | **MapTiler Free**（ハード上限・超過で自動停止、課金事故ゼロ。ただし商用不可）。または **Google**（per-SKU 無料枠が大きく小規模なら $0）。従量課金の Google/Mapbox/AWS は上限アラート設定必須。 |
| **検索結果を DB 保存** | **AWS（intendedUse=Stored, $4/1,000）** または **Mapbox Permanent Geocoding（$5/1,000）**。Google は **Place ID のみ無期限保存可**、その他コンテンツは原則保存不可。 |

**最重要の前提（本文で詳述）**
- **Geocoding（住所⇄座標）と POI / Places Search（施設名・店舗名検索）は別物。** 住所ジオコーダーは店舗名・組織名検索に弱い、または対象外。
- **日本国内の POI 網羅性は Google が突出。** Mapbox / MapTiler（OSM ベース）と AWS（Esri / HERE）は、日本の中小店舗・建物名・日本語 POI 名で見劣りする傾向（実務観点・要実地検証）。
- **Autocomplete の課金単位は会社ごとに違う。** Mapbox はセッション、Google はセッショントークン方式（条件付きで無料化）、AWS は per-request。混同しないこと。

---

## 2. API カテゴリ比較表

| Category | Google Maps Platform | Mapbox | MapTiler | AWS Amazon Location Service | Notes |
|---|---|---|---|---|---|
| **地図 / Raster tile** | Map Tiles（Raster） | Raster Tiles API | Raster tiles | GeoMaps `GetTile`(raster) | — |
| **地図 / Vector tile** | Map Tiles（Vector） | Vector Tiles API | Vector tiles（PBF） | GeoMaps `GetTile`(vector) | MapTiler / Mapbox / AWS は MapLibre or 互換で描画可 |
| **地図 / Static map** | Maps Static API | Static Images API | Static Maps API | GeoMaps `GetStaticMap` | — |
| **地図 / SDK 描画** | Maps JS API, Maps SDK(Android/iOS) | Mapbox GL JS, Mobile SDKs | MapTiler SDK JS（MapLibre ベース） | （MapLibre + Amazon Location 連携） | Google は独自SDK / 他社は MapLibre 系 |
| **住所 / Forward geocoding** | Geocoding API | Geocoding API v6 | Geocoding API | GeoPlaces `Geocode` | 住所→座標 |
| **住所 / Reverse geocoding** | Geocoding API | Geocoding API v6(reverse) | Geocoding API(reverse) | GeoPlaces `ReverseGeocode` | 座標→住所 |
| **住所 / Batch geocoding** | （クライアント側で反復） | （反復） | Geocoding batch（最大50/call） | `BulkValidateAddress`(検証) | Google は専用バッチSKUなし |
| **POI / Text search** | Places API(New) **Text Search** | **Search Box** `/category`, Geocoding(types=poi) | Geocoding(POI 含む) | GeoPlaces `SearchText` | **Google が日本最強** |
| **POI / Nearby search** | Places API(New) **Nearby Search** | Search Box `/category` | （Geocoding proximity） | GeoPlaces `SearchNearby` | — |
| **POI / Category search** | Places(New) included types | Search Box `/category` | （限定的） | GeoPlaces `SearchNearby`(categories) | — |
| **POI / Place details** | Places(New) **Place Details** | Search Box `/retrieve` | （Geocoding feature） | GeoPlaces `GetPlace` | — |
| **入力補完 / Autocomplete** | Places(New) **Autocomplete**（session token） | Search Box `/suggest`（session） | Geocoding autocomplete（既定ON, per-request） | GeoPlaces `Autocomplete` / `Suggest`（per-request） | **課金単位が各社で異なる** |
| **入力補完 / Search session** | Autocomplete Session Usage（条件付き無料） | Search Box session（180秒/50 suggest で確定） | なし（per-request） | なし（per-request） | — |
| **経路 / Routing** | Routes API `Compute Routes` | Directions API | （routing は限定 / 要確認） | GeoRoutes `CalculateRoutes` | MapTiler は経路弱い |
| **経路 / Matrix** | Routes API `Compute Route Matrix` | Matrix API | — | GeoRoutes `CalculateRouteMatrix` | — |
| **経路 / Isochrone** | （なし） | Isochrone API | — | GeoRoutes `CalculateIsolines` | — |
| **経路 / Optimization** | （Routes 内一部） | Optimization API | — | GeoRoutes `OptimizeWaypoints` | — |
| **その他 / Elevation** | Elevation API | Tilequery / Terrain | Elevation API | （なし / 要確認） | — |
| **その他 / Time zone** | Time Zone API | （なし） | （なし） | GeoPlaces `additionalFeature=TimeZone` | — |
| **その他 / Boundaries** | （Data-driven styling 経由） | Boundaries | （なし） | （なし） | — |
| **その他 / Traffic** | Roads / Routes traffic | Directions(traffic) | （なし） | GeoRoutes(traffic) | — |
| **その他 / Data storage** | 不可（Place ID 除く） | Permanent geocoding で可 | Geocoding 結果は保存可 / タイルは server-cache 不可 | `intendedUse=Stored` で可（HERE 日本は不可） | **保存可否は §5 参照** |

---

## 3. 検索系 API の階層図（ツリー）

### Google Maps Platform
```text
Google Maps Platform
├── Geocoding API                         ← 住所 ⇄ 座標（POIではない）
│   ├── Forward geocoding（住所→座標）
│   └── Reverse geocoding（座標→住所）
├── Places API (New)                      ← 施設名・店舗名・組織名検索
│   ├── Text Search（フリーテキストでPOI検索）
│   ├── Nearby Search（座標周辺のPOI）
│   ├── Autocomplete（入力補完, session token）
│   └── Place Details（Place ID → 詳細情報）
│         ※ field mask で Essentials/Pro/Enterprise の SKU が変わる
├── Address Validation API                ← 住所の正規化・検証
└── Routes API（経路）
※ Geocoding と Places は完全に別物。住所変換は Geocoding、店名検索は Places。
※ 旧 Places API は Legacy。新規実装は Places API (New) を使う。
```

### Mapbox
```text
Mapbox
├── Geocoding API v6                      ← 住所 ⇄ 座標（POIも一部対応）
│   ├── Forward geocoding
│   ├── Reverse geocoding
│   ├── Temporary（既定 / 結果保存不可 / 安い）
│   └── Permanent（permanent=true / 結果保存可 / 高い・無料枠なし）
├── Search Box API                        ← 対話的なPOI/住所検索（セッション課金）
│   ├── /suggest（入力補完, session_token）
│   ├── /retrieve（候補→詳細, セッション確定）
│   └── /category（カテゴリ検索）
└── Directions / Matrix / Isochrone / Optimization（経路）
※ 住所変換=Geocoding、対話検索UX=Search Box、と役割が分かれる。
※ 保存したい結果は Permanent geocoding を使う（temporary は保存禁止）。
```

### MapTiler
```text
MapTiler Cloud
├── Geocoding API                         ← 住所/地名/POI（autocomplete 既定ON）
│   ├── Forward geocoding
│   ├── Reverse geocoding
│   ├── Autocomplete（per-request, 既定有効）
│   └── Batch（最大50クエリ/call）
├── Coordinates API（座標系変換 / EPSG）
└── （経路APIは限定的 / 要確認）
※ OSM ベース。日本の POI / 詳細住所は弱い想定（実務上要検証）。
※ 検索系に session 課金はなく、すべて「requests」プールを消費。
```

### AWS Amazon Location Service
```text
Amazon Location Service
├── GeoPlaces（新・スタンドアロンAPI / リソース作成不要）
│   ├── Geocode（住所→座標）
│   ├── ReverseGeocode（座標→住所）
│   ├── SearchText（フリーテキストPOI検索）
│   ├── SearchNearby（周辺POI）
│   ├── Autocomplete / Suggest（入力補完, per-request）
│   └── GetPlace（Place 詳細）
│         ※ additionalFeature と intendedUse で Label/Core/Advanced/Stored バケットが変わる
├── GeoRoutes（CalculateRoutes / Matrix / Isolines / OptimizeWaypoints）
├── GeoMaps（GetTile / GetStaticMap）
├── Trackers（位置追跡）
└── Geofencing（ジオフェンス）
※ 旧 Place Index リソースモデル（SearchPlaceIndexForText 等）も併存。
※ データプロバイダ（Esri / HERE / Grab / Open Data）で網羅性・規約が変わる。
※ 結果の永続保存は intendedUse=Stored（$4/1,000）。ただし HERE の日本データは保存不可。
```

---

## 4. 課金モデル比較表

| Provider | Billing Unit | Free Tier | Autocomplete Billing | Temporary / Permanent / Stored | Subscription | Cost Risk | Notes |
|---|---|---|---|---|---|---|---|
| **Google Maps Platform** | per request（多くは 1,000 単位 CPM） | **SKU ごとに毎月無料**（Essentials 10,000 / Pro 5,000 / Enterprise 1,000、Map Tiles 最大 100,000）。合計 ~$3,250/月相当。**$200 クレジットは 2025/3 廃止**。 | **session token 方式**。終端を Place Details Pro/Enterprise か Address Validation にすると Autocomplete は無料化。Essentials 終端だと先頭 12 リクエストまで課金。未確定セッションは per-request 課金。 | 原則 **保存不可**（キャッシュ制限）。**Place ID のみ無期限保存可**。一部データは限定的に短期キャッシュ可。 | 大口向け Subscription あり。標準は従量課金。 | **中〜高**。従量課金で上限が緩く、設計次第で高額化。予算アラート必須。 | 日本 POI 最強。SKU が細かく見積もりが複雑。 |
| **Mapbox** | per request（Geocoding 等） / **per session（Search Box）** / map load（GL JS） | Map loads 50,000 / Temporary Geocoding 100,000 / Search Box 500 sessions / Directions 等 100,000（各/月）。Permanent Geocoding は **無料枠なし**。 | **Search Box はセッション課金**（$3/1,000 sessions）。session_token で suggest+retrieve を 1 課金に束ねる。Geocoding は per-request。 | **Temporary**（既定・保存不可・安い）/ **Permanent**（permanent=true・保存可・$5/1,000・無料枠なし）。Search Box 結果は一時利用のみ（長期保存は要営業相談）。 | 標準は従量課金（Pay-as-you-go）。 | **中**。従量だが無料枠が大きく小規模は安全。要上限設定。 | 全プランで attribution（Mapbox ロゴ + OSM 表記）必須。日本 POI は弱め。 |
| **MapTiler** | **月額プラン + 超過従量**。指標は「sessions（SDK 地図表示）」と「requests（API/タイル/Geocoding）」。 | **Free プラン**: 5,000 sessions / 100,000 requests（**商用不可**）。Flex/Unlimited にプラン内枠。 | session 課金なし。**全て per-request**（requests プール消費）。Autocomplete も 1 キーストローク=1 request。 | **Geocoding 結果は保存可**（規約で明記）。**地図タイルは server-side キャッシュ禁止**（端末キャッシュのみ可）。 | **あり**。Free / Flex $25 / Unlimited $295 / Custom。 | **低〜中**。Free はハード上限で自動停止＝課金事故ゼロ。Flex/Unlimited は超過が後払い課金（spending-limit 機能で上限設定可）。Custom のみ true soft-limit。 | コストが読みやすい。OSS / MapLibre 寄り。日本 POI 弱め。 |
| **AWS Amazon Location** | **per request**（バケット別単価） | **3か月トライアル枠**（常時無料ではない）: Maps 500,000 タイル + Static 5,000 / Places 10,000 autocomplete + 20,000 geocode等 / Routes 10,000 など（各/月）。別途 2025/7/15 以降の新規アカウントは $200 クレジット(6か月)。 | **per-request**（セッション課金なし）。Autocomplete/Suggest は **Label $0.20/1,000** または Core $0.50/1,000。 | **Label/Core/Advanced は保存不可**（キャッシュのみ）。**intendedUse=Stored（$4/1,000）で永続保存可**。**ただし HERE の日本データは保存禁止**。 | なし（純従量、AWS 請求に統合）。 | **中**。AWS Budgets / アラートで管理。タイル課金はリクエスト粒度で読みづらい。 | データプロバイダで網羅性・規約が変動。日本 POI は弱め。 |

---

## 5. Temporary / Permanent / Stored の解説（保存可否・キャッシュ・DB保存）

意思決定で最も事故が起きやすい論点。**「検索結果を DB に保存できるか」は会社ごとに規約・料金が大きく異なる。**

### Google Maps Platform
- **原則保存不可。** Maps Service Specific Terms §3.2.3(b) によりコンテンツのキャッシュ/保存は明示的許可がない限り禁止。
- **例外: Place ID は無期限保存可**（§3.2.3(b) で明示的に除外）。→ 実務では「Place ID だけ自前 DB に持ち、表示時に Place Details で都度取得」が王道。
- 一部データ（例: Navigation SDK の緯度経度）は最大 30 日の短期キャッシュ可。
- **DB保存したいなら: Place ID を保存し、属性は都度取得する設計にする。**
- 出典: Maps Platform Service Specific Terms / Places Policies / Place IDs（§9 参照）。

### Mapbox
- **Temporary Geocoding（既定）**: 結果を **保存・キャッシュ・DB 蓄積は禁止**。表示・近接検索・経路・リアルタイム処理など一時利用のみ。安い（$0.75/1,000、無料 100,000/月）。
- **Permanent Geocoding（`permanent=true`）**: 結果を **DB / ローカルに保存・再利用・バッチ/オフライン処理が可能**。$5/1,000、**無料枠なし**。保存するのに temporary を使うと **ToS 違反**。
- **Search Box API**: 結果は **一時利用のみ**。長期保存は Mapbox 営業へ要相談（`permanent` 相当フラグなし）。
- **DB保存したいなら: Permanent Geocoding を使う。**

### MapTiler
- **Geocoding / 検索結果は Service 外での保存が許可**（Cloud Terms に明記: "Results of the search services ... are allowed for usage outside of the Service"）。→ **DB保存に追加料金・追加規約が不要な数少ない選択肢。**
- **地図タイルは server-side キャッシュ/保存/再配布が禁止**（端末ローカルキャッシュのみ可）。スクリーンショット保存も不可。
- attribution（MapTiler + OSM）必須（有料プランで MapTiler ロゴは非表示化可、データ表記は必要）。

### AWS Amazon Location Service
- **Label / Core / Advanced バケットの結果は永続保存不可**（キャッシュのみ）。
- **`intendedUse=Stored`** を指定すると永続保存可。料金は **Stored バケット $4/1,000**（全 feature 込みの上限価格）。旧モデルでは Place Index の `IntendedUse=Storage` 相当。
- **重要な例外: データプロバイダが HERE の場合、日本国内ロケーションの結果は保存禁止**（公式制約）。→ 日本データを保存したいなら Esri 等を選ぶか、保存可否を要確認。
- **DB保存したいなら: intendedUse=Stored、かつプロバイダの日本データ保存可否を確認。**

---

## 6. 用途別おすすめ

```text
用途: 日本国内の POI / 店舗名 / ビル名検索を重視
推奨: Google Places API (New) — Text Search / Autocomplete / Place Details
理由: 日本の店舗・施設・建物名の網羅性と鮮度が突出。統合検索UXを単一エコシステムで作れる。
注意: 単価が高い（Text Search Pro $32/1,000、Place Details Pro $17/1,000）。無料枠超過後のコスト管理必須。Place ID 保存設計で再取得コストを抑える。
```
```text
用途: 安価に住所ジオコーディングしたい（保存しない）
推奨: Mapbox Geocoding（Temporary）
理由: 無料 100,000/月、超過 $0.75/1,000 と最安級。住所⇄座標に特化なら十分。
注意: 結果を DB 保存すると ToS 違反。保存が必要なら Permanent（$5/1,000・無料枠なし）か AWS Core/Stored を比較。日本の番地レベル精度は要実地検証。
```
```text
用途: 住所ジオコーディング結果を DB に保存したい
推奨: AWS（intendedUse=Stored, $4/1,000）または Mapbox Permanent（$5/1,000）/ MapTiler（保存許可）
理由: 明示的に永続保存が許可される料金/規約がある。
注意: Google は Place ID 以外保存不可。AWS+HERE は日本データ保存禁止。MapTiler は OSM 由来で日本精度に注意。
```
```text
用途: Google Maps 風の統合検索 UX を作りたい
推奨: Google Places API (New)
理由: Autocomplete(session) → Place Details → Text Search/Nearby が同一API群で完結。日本語UXも良好。
注意: session token 設計を正しく行わないと per-request 課金で高額化。終端 SKU を意識する。
```
```text
用途: Autocomplete を低コストに実装したい
推奨: Mapbox Search Box API（セッション課金 $3/1,000 sessions, 500 無料）
理由: 1ユーザーの「入力→確定」を 1 課金に束ねられ、キーストロークごとに課金されない。
注意: 日本 POI 網羅性は Google 比で弱い。Google も session 設計次第で Autocomplete 無料化可能だが設計が複雑。AWS は per-request（Label $0.20/1,000）で別モデル。
```
```text
用途: 地図表示を低コストに実装したい
推奨: MapTiler（月額で読みやすい）または AWS GeoMaps（タイル $0.04/1,000）
理由: MapTiler は Flex $25 / Unlimited $295 で予算が固定でき事故りにくい。AWS はタイル単価が最安級。
注意: AWS はタイル粒度課金で「セッション数→タイル数」換算が読みづらい（§7 の前提参照）。Mapbox/Google は map load 課金。
```
```text
用途: MapLibre / OSS 寄りの構成にしたい
推奨: MapTiler（+ MapLibre GL JS / MapTiler SDK）
理由: MapTiler は MapLibre の主要スポンサーで SDK も MapLibre ベース。ベンダーロックインを避けやすい。
注意: 検索/ジオコーディングは OSM ベースで日本精度に難。POI は別途 Google 併用も検討。
```
```text
用途: AWS 環境に統合したい
推奨: Amazon Location Service
理由: IAM / CloudWatch / Budgets / VPC とネイティブ統合。請求も AWS に一本化。
注意: 日本 POI が弱い。HERE 利用時は日本データ保存禁止。タイル課金の読みづらさに注意。
```
```text
用途: 請求事故を避けて個人学習したい
推奨: MapTiler Free（ハード上限・自動停止＝課金事故ゼロ）/ Google（per-SKU 無料枠が大きく小規模は実質 $0）
理由: MapTiler Free は超過で停止し請求が発生しない。Google は小規模なら無料枠内で収まる。
注意: MapTiler Free は商用不可。従量課金の Google/Mapbox/AWS は必ず予算アラート/上限を設定。
```
```text
用途: 商用 SaaS で月間数万〜数百万件使いたい
推奨: 機能で分離 — 地図表示=MapTiler/AWS、住所=Mapbox/AWS、日本POI検索=Google
理由: 1社で全部賄うより、強い領域ごとに使い分けるのが最もコスト効率が良い。
注意: 各社の無料枠・volume discount・保存規約を月次で再確認。請求アラート必須。複数社運用は attribution 表記も各社分必要。
```

---

## 7. 実コスト試算（概算）

> **重大な前提と不確実性（必読）**
> - 単価は base tier。Google / Mapbox は **volume discount** で大口は逓減（下表は割引前または一部のみ反映、過大評価寄り）。
> - **AWS は「地図表示セッション」概念がなくタイル単位課金**。ここでは **1 セッション ≈ 20 タイルリクエスト** と仮定（パン/ズーム込み）。実値はUI次第で大きく変動 → **要確認**。
> - **Autocomplete 1 セッション ≈ 5 キーストローク + 1 詳細取得** と仮定。
> - **Google Autocomplete** は終端を Place Details Pro にして Autocomplete 無料化する設計を仮定。
> - **Google の volume 帯境界**（0–100K / 100K–500K / 500K–1M / 1M–5M）は推定。**要確認**。
> - AWS は **3か月トライアル中は概ね $0**。下表 AWS は **トライアル後（定常）**。
> - MapTiler は **月額プラン**で、超過は後払い。商用は Free 不可。

### シナリオ A: 小規模検証（地図 10,000 / Geocoding 5,000 / POI 1,000 / Autocomplete 1,000 セッション）

| Provider | 概算 / 月 | 内訳・コメント |
|---|---|---|
| **Google** | **≈ $0** | すべて per-SKU 無料枠内（Map 10,000、Geocoding 10,000、Text Search Pro 5,000、Autocomplete/Place Details 各 10,000）。小規模なら無料は強い。 |
| **Mapbox** | **≈ $0〜5** | Map/Geocoding は無料枠内。Search Box（POI+Autocomplete=2,000 sessions − 500 無料）= 1,500 × $3/1,000 ≈ **$4.5**。POI を Geocoding で代替すれば ~$0。 |
| **MapTiler** | **≈ $25**（Flex） | 地図 10,000 sessions > Free 上限 5,000 のため Flex 必須。商用なら尚更。requests は枠内。 |
| **AWS** | **トライアル中 ≈ $0 / 後 ≈ $12** | Map 10,000×20タイル=200,000×$0.04/1,000=$8 + Geocoding $2.5 + POI $0.5 + Autocomplete $1。タイル仮定が支配的。 |

### シナリオ B: 小規模商用（地図 100,000 / Geocoding 50,000 / POI 10,000 / Autocomplete 20,000 セッション）

| Provider | 概算 / 月 | 内訳・コメント |
|---|---|---|
| **Google** | **≈ $1,300** | Map (100,000−10,000)×$7=$630 + Geocoding (50,000−10,000)×$5=$200 + Text Search Pro (10,000−5,000)×$32=$160 + Autocomplete(Pro終端: Place Details Pro 20,000−5,000)×$17 ≈ $255。日本POI価値とのトレードオフ。 |
| **Mapbox** | **≈ $340** | Map (100,000−50,000)×$5=$250 + Geocoding 無料枠内=$0 + Search Box(30,000−500)×$3/1,000≈$88.5。コスト優位だが日本POI弱。 |
| **MapTiler** | **≈ $175**（Flex） | 地図 100,000 sessions: Flex $25 + 超過(100,000−25,000)×$2/1,000=$150。requests 枠内。地図主体なら最安級だが POI は非推奨。 |
| **AWS** | **トライアル後 ≈ $130** | Map 100,000×20×$0.04/1,000=$80 + Geocoding $25 + POI $5 + Autocomplete $20。安いが日本POI弱・タイル仮定依存。 |

### シナリオ C: 中規模（地図 1,000,000 / Geocoding 300,000 / POI 100,000 / Autocomplete 200,000 セッション）

| Provider | 概算 / 月 | 内訳・コメント |
|---|---|---|
| **Google** | **≈ $12,000〜13,000** | Map ≈$4,970（帯別: 90K×$7+400K×$5.6+500K×$4.2）+ Geocoding ≈$1,250 + Text Search Pro 95,000×$32≈$3,040 + Autocomplete(Pro終端)≈$2,975。**帯境界は要確認**。大口は要見積/割引交渉。 |
| **Mapbox** | **≈ $5,000〜6,000** | Map (950,000)×$5≈$4,750（割引で実際は下振れ）+ Geocoding 200,000×$0.75/1,000=$150 + Search Box 299,500×$3/1,000≈$900。 |
| **MapTiler** | **≈ $1,350**（Unlimited） | 地図 1,000,000 sessions: Unlimited $295 + 超過(700,000)×$1.5/1,000=$1,050。requests 5M 枠内。**地図主体なら圧倒的に安い**が日本POI検索には不向き。実際は Custom 契約圏。 |
| **AWS** | **トライアル後 ≈ $1,200** | Map 1,000,000×20×$0.04/1,000=$800 + Geocoding $150 + POI $50 + Autocomplete $200。最安級だが日本POIの実用性と HERE 日本保存禁止に注意。 |

**コスト試算の読み方**
- **地図表示だけ**なら MapTiler / AWS が桁違いに安い。
- **日本の POI 検索**を本気でやるなら Google 一択で、その分高い → **「地図は安い会社、POI 検索だけ Google」のハイブリッド**が現実解。
- 数字は前提に強く依存。本番採用前に各社の料金計算ツールで自社トラフィックを当てて再計算すること。

---

## 8. 選定フローチャート

```text
POI / 施設名・店舗名検索が必要か？
├── Yes（店名・施設名・ビル名を検索する）
│   ├── 日本国内の精度を最優先          → Google Places API (New)
│   ├── コスト重視（日本精度は妥協可）    → Mapbox Search Box を PoC 検証
│   ├── AWS 環境統合を重視              → Amazon Location Service（プロバイダ要選定）
│   └── 結果を DB 保存したい            → AWS (Stored) / Mapbox Permanent ／ Google は Place ID のみ
└── No（POI 検索は不要）
    ├── 住所ジオコーディング中心
    │   ├── 保存しない・安く            → Mapbox Temporary Geocoding
    │   ├── 保存する                   → Mapbox Permanent / AWS Stored / MapTiler
    │   └── 高精度・統合               → Google Geocoding
    └── 地図表示中心
        ├── OSS / MapLibre / 予算固定   → MapTiler
        ├── タイル単価最安・AWS統合      → Amazon Location (GeoMaps)
        ├── デザイン自由度・モバイルSDK  → Mapbox
        └── Google エコシステム前提      → Google Maps（Dynamic Maps）

請求事故を避けたい（個人/学習）？
└── Yes → MapTiler Free（自動停止）／ Google（小規模は無料枠内）。従量各社は予算アラート必須。
```

---

## 9. 出典（公式情報源）

### Google Maps Platform
- Pricing overview: https://developers.google.com/maps/billing-and-pricing/overview
- 2025年3月の料金改定: https://developers.google.com/maps/billing-and-pricing/march-2025
- Core services pricing（SKU/CPM）: https://developers.google.com/maps/billing-and-pricing/pricing
- Billing FAQ（$200廃止・volume discount・Legacy）: https://developers.google.com/maps/billing-and-pricing/faq
- per-product 無料枠 blog: https://mapsplatform.google.com/resources/blog/start-building-today-with-up-to-10-000-monthly-free-calls-per-product/
- Autocomplete / session pricing: https://developers.google.com/maps/documentation/places/web-service/session-pricing
- session tokens: https://developers.google.com/maps/documentation/places/web-service/using-session-tokens
- Geocoding usage & billing: https://developers.google.com/maps/documentation/geocoding/usage-and-billing
- Service Specific Terms（§3.2.3 caching）: https://cloud.google.com/maps-platform/terms/maps-service-terms
- Places Policies / Place IDs: https://developers.google.com/maps/documentation/places/web-service/policies ／ https://developers.google.com/maps/documentation/places/web-service/place-id
- Coverage details（日本フィルタ可）: https://developers.google.com/maps/coverage

### Mapbox
- Pricing: https://www.mapbox.com/pricing ／ https://docs.mapbox.com/accounts/guides/pricing/
- GL JS map loads pricing: https://docs.mapbox.com/mapbox-gl-js/guides/pricing/
- Geocoding API: https://docs.mapbox.com/api/search/geocoding/
- Temporary vs Permanent Geocoding: https://docs.mapbox.com/help/dive-deeper/understand-temporary-vs-permanent-geocoding/
- Search Box API: https://docs.mapbox.com/api/search/search-box/
- Attribution: https://docs.mapbox.com/help/dive-deeper/attribution/
- Terms of Service: https://www.mapbox.com/legal/tos
- Data sources: https://docs.mapbox.com/help/dive-deeper/mapbox-data-sources/

### MapTiler
- Cloud pricing: https://www.maptiler.com/cloud/pricing/
- Geocoding API: https://docs.maptiler.com/cloud/api/geocoding/
- Tiles API: https://docs.maptiler.com/cloud/api/tiles/
- Cloud Terms（保存可否）: https://www.maptiler.com/terms/cloud/
- Copyright / attribution: https://www.maptiler.com/copyright/
- session vs request: https://docs.maptiler.com/guides/maps-apis/maps-platform/tile-requests-and-map-sessions-compared/
- Open source / MapLibre: https://www.maptiler.com/open-source/

### AWS Amazon Location Service
- Pricing & free tier: https://aws.amazon.com/location/pricing/
- Places pricing（Label/Core/Advanced/Stored・保存規約）: https://docs.aws.amazon.com/location/latest/developerguide/places-pricing.html
- Routes pricing: https://docs.aws.amazon.com/location/latest/developerguide/routes-pricing.html
- Maps pricing: https://docs.aws.amazon.com/location/latest/developerguide/maps-pricing.html
- New standalone APIs（GeoPlaces/GeoRoutes/GeoMaps）: https://aws.amazon.com/blogs/aws/announcing-new-apis-for-amazon-location-service-routes-places-and-maps/
- FAQs: https://aws.amazon.com/location/faqs/
- Price List offer file（実数値）: https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonLocationService/current/index.json
- HERE 日本保存制約 / providers: https://docs.aws.amazon.com/location/previous/APIReference/API_CreatePlaceIndex.html

### 補足プロバイダ（参考）
- **OpenStreetMap / MapLibre**: 自前ホスティングで API 課金ゼロ化可能だが、運用・タイル生成・ジオコーダ（Nominatim/Photon）構築コストが必要。日本 POI は OSM 品質依存。
- **HERE / TomTom**: 商用地図データに強み。日本カバレッジは Google に次ぐ場合があるが、本資料の主比較外（必要なら別途調査）。

---

## 要確認サマリ（再検証すべき項目）

- Google: Enterprise tier の SKU 全リストと "Preferred" フィールド階層の正式名称、volume 帯の正確な境界値、Legacy API（Places/Directions/Distance Matrix）の廃止日。
- Mapbox: pricing ページの各 volume 帯境界と逓減ステップ、Zenrin データが日本の geocoding/POI 検索に使われるか（地図表示のみか）。
- MapTiler: batch geocoding の request カウント（1 か 50 か）、OpenAddresses 利用有無。
- AWS: Tokyo リージョン（ap-northeast-1）の単価、3か月トライアル枠と $200 クレジットモデルの併存状況、Grab プロバイダ単価。
- 全社: **日本の POI / 住所精度は料金表に現れない実務観点**。本番前にサンプルクエリで必ず実地検証すること。
