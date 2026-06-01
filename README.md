# location-api-decision-matrix

Web / モバイルアプリで地図・位置情報機能を実装する際に、主要な **地図系 API / 位置情報 API** を比較し、用途に応じて適切なプロバイダを選定するための意思決定資料です。

## 比較対象

| プロバイダ | 強み（ざっくり） |
|---|---|
| **Google Maps Platform** | 日本の POI / 店舗名・施設名・ビル名検索が最強。統合検索 UX。単価は高め。 |
| **Mapbox** | 住所ジオコーディングが安い（Temporary）。Search Box はセッション課金。デザイン自由度高。 |
| **MapTiler** | 月額プランでコストが読みやすい。MapLibre / OSS 寄り。地図表示向き。 |
| **AWS Amazon Location Service** | AWS 統合・タイル単価が安い。バケット課金（Label/Core/Advanced/Stored）。 |

補足として OpenStreetMap / MapLibre / HERE / TomTom にも軽く触れています。

## 主な比較観点

- 地図表示 API と検索 API の違い
- **Geocoding（住所⇄座標）と POI / Places Search（施設名検索）の違い**（別物）
- Autocomplete / Suggest の課金単位（per-request か session か）
- **Temporary / Permanent / Stored** の保存可否と規約差（DB 保存できるか）
- 無料枠の形の違い（SKU 別 / 月額プラン内 / 全体クレジット / 期間限定トライアル）
- 日本国内の POI 検索カバレッジの実務的な差
- 月間利用件数ごとの概算コスト（小規模検証 / 小規模商用 / 中規模）

## 結論（要約）

- **日本の POI / 店舗名検索を重視** → Google Places API (New)
- **住所ジオコーディングを安く（保存しない）** → Mapbox Geocoding (Temporary)
- **地図表示を安く / OSS 寄り** → MapTiler または AWS GeoMaps
- **Autocomplete を低コストに** → Mapbox Search Box（セッション課金）
- **検索結果を DB 保存** → AWS (Stored) / Mapbox (Permanent) ／ Google は Place ID のみ
- **請求事故を避けたい** → MapTiler Free（自動停止）／ Google（小規模は無料枠内）
- **現実解**: 「地図は安い会社、日本の POI 検索だけ Google」のハイブリッド構成

詳細は **[docs/location-api-decision-matrix.md](docs/location-api-decision-matrix.md)** を参照してください。
（カテゴリ比較表 / 階層図 / 課金モデル比較 / 保存可否解説 / 用途別おすすめ / 実コスト試算 / 選定フローチャート / 出典）

## 想定読者

- Web / モバイルアプリに地図・位置情報機能を実装するエンジニア / テックリード
- 地図 API の選定とコスト試算を行うプロダクトオーナー / 意思決定者
- 日本国内サービスで POI 検索の精度とコストを両立させたい開発者

## 注意事項

> **料金・無料枠・API 名称・保存条件は頻繁に変動します。** 本資料は 2026-06-01 時点の各社公式情報に基づく概算であり、採用判断の前に必ず各社の公式ドキュメント / 料金ページで最新情報を確認してください。コスト試算は明示した前提（地図セッション→タイル換算、Autocomplete セッションの構成、volume 帯境界など）に強く依存します。**要確認** と記載した項目は特に再検証が必要です。
