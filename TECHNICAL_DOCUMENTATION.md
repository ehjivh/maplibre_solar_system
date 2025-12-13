# 太陽系大きさ体感マップ - 技術資料

## 1. プロジェクト概要

### 1.1 背景と目的
天文学的な距離やスケールは、その巨大さゆえに人間が直感的に理解することが困難です。本プロジェクトは、太陽系の惑星や天体の大きさを地理的なスケールに変換し、地図上に可視化することで、天文学的スケールの体感的理解を促進することを目的としています。

### 1.2 研究意義
- **教育的価値**: 天文学教育における新しい教材として活用可能
- **直感的理解**: 抽象的な天文学的距離を身近な地理空間に対応付け
- **インタラクティブ性**: ユーザーが能動的に探索できる学習環境の提供
- **スケール感の獲得**: 惑星間距離や天体サイズの相対的関係の理解促進

### 1.3 実装アプリケーション一覧
本プロジェクトでは4つの異なるスケールのマップを実装：

1. **太陽系大きさ体感マップ** (solar-system-map.html)
   - 太陽系内の惑星軌道を地図上に描画
   - カイパーベルト、オールトの雲まで対応

2. **恒星サイズ比較マップ** (stellar-comparison-map.html)
   - 様々な恒星の大きさを比較可視化
   - 主系列星から超巨星まで対応

3. **恒星間距離体感マップ** (stellar-distance-map.html)
   - 太陽系から近隣恒星までの距離を可視化
   - 28個の代表的な恒星を配置

4. **銀河系大きさ体感マップ** (galaxy-distance-map.html)
   - 銀河系内の主要天体までの距離を可視化
   - 銀河系中心、マゼラン雲、アンドロメダ銀河等を含む

---

## 2. 技術スタック

### 2.1 コア技術
- **MapLibre GL JS v5.0.0**
  - オープンソースの地図レンダリングライブラリ
  - WebGLベースの高性能レンダリング
  - ベクタータイルとラスタータイルの両対応
  
- **Turf.js v6.5.0**
  - 地理空間解析ライブラリ
  - 円の生成、距離計算、方位計算等に使用
  
- **Vanilla JavaScript**
  - フレームワーク非依存の実装
  - 軽量で高速な動作

### 2.2 使用データソース
- **OpenStreetMap**: デフォルトベースマップ
- **地理院地図**: 
  - 標準地図
  - 淡色地図
  - 白地図
  - 衛星写真
- **Geolocation API**: 現在地取得機能

### 2.3 Web技術
- HTML5
- CSS3 (Flexbox, Transitions)
- ES6+ JavaScript (Arrow Functions, Destructuring, Template Literals)

---

## 3. 実装アーキテクチャ

### 3.1 縮尺計算アルゴリズム

#### 3.1.1 基本原理
ユーザーが指定した地球の直径（cm）から、すべての天体と距離の縮尺を統一的に計算：

```javascript
// 縮尺係数の計算
const scale = earthDiameterCm / (EARTH_ACTUAL_DIAMETER_KM * 100000);

// 1天文単位の地図上での距離（メートル）
const oneAUInMeters = (AU_IN_KM * 100000 * scale) / 100;
```

#### 3.1.2 縮尺変換プロセス
1. 実際の直径（km）→ cm単位に変換
2. 縮尺係数を乗算
3. 地図用の単位（メートル）に変換

この手法により、すべての天体が同一の縮尺で表現され、相対的なサイズ関係が正確に保持されます。

### 3.2 軌道描画システム

#### 3.2.1 Turf.jsによる円生成
```javascript
const orbitCircle = turf.circle(sunCoordinates, orbitRadiusKm, {
    steps: 128,  // 円の滑らかさ
    units: 'kilometers'
});
```

- `steps: 128`: 円周を128個のポイントで近似
- 高精度な円形軌道を効率的に生成

#### 3.2.2 天体配置アルゴリズム
```javascript
// 惑星の位置を計算（軌道上の東側、方位角90度）
const planetPosition = turf.destination(
    sunCoordinates,      // 始点（太陽の位置）
    orbitRadiusKm,       // 距離
    90,                  // 方位角（東方向）
    { units: 'kilometers' }
);
```

### 3.3 描画範囲の最適化

地球の周囲距離（約40,075km）を考慮し、描画範囲を20,000km以内に制限：

```javascript
if (orbitRadiusKm > 20000) {
    // 範囲外の天体情報を記録
    window.outOfRangePlanets.push({
        name: planet.name,
        color: planet.color,
        distanceKm: orbitRadiusKm,
        timesAroundEarth: orbitRadiusKm / 40075
    });
    return;
}
```

これにより：
- パフォーマンスの維持
- 地球周回数による直感的な距離表現
- メモリ効率の向上

### 3.4 レイヤー管理システム

#### 3.4.1 レイヤー構成
各天体につき3つのレイヤーを動的生成：
1. **軌道レイヤー**: `orbit-${planet.name}`
2. **天体本体レイヤー**: `planet-${planet.name}`
3. **ラベルレイヤー**: `label-${planet.name}`

#### 3.4.2 レイヤー更新フロー
```javascript
// 既存レイヤーの削除
PLANETS.forEach(planet => {
    ['orbit', 'planet', 'label'].forEach(type => {
        const layerId = `${type}-${planet.name}`;
        if (map.getLayer(layerId)) {
            map.removeLayer(layerId);
        }
    });
});

// 新規レイヤーの追加
map.addLayer({
    id: orbitLayerId,
    type: 'line',
    source: orbitSourceId,
    paint: { /* スタイル定義 */ }
});
```

---

## 4. 主要機能の実装詳細

### 4.1 現在地取得機能

#### 4.1.1 Geolocation APIの活用
```javascript
navigator.geolocation.getCurrentPosition(
    (position) => {
        const { latitude, longitude } = position.coords;
        sunCoordinates = [longitude, latitude];
        map.flyTo({ center: sunCoordinates, zoom: 14, duration: 2000 });
        updateOrbits();
    },
    (error) => { /* エラーハンドリング */ },
    {
        enableHighAccuracy: true,  // 高精度測位
        timeout: 10000,            // 10秒タイムアウト
        maximumAge: 0              // キャッシュ無効化
    }
);
```

#### 4.1.2 ユーザーエクスペリエンスの配慮
- 取得中の視覚的フィードバック（📡アイコン）
- 詳細なエラーメッセージ
- スムーズなアニメーション（flyTo）

### 4.2 到達時間計算機能

3つの速度で到達時間を計算：

```javascript
function calculateTravelTime(distanceKm) {
    const LIGHT_SPEED_KM_S = 299792.458;  // 光速
    const SHINKANSEN_KM_H = 300;          // 新幹線
    const WALKING_KM_H = 4;               // 徒歩

    // 適切な単位で表示（秒→分→時間→日→年）
    const lightSeconds = distanceKm / LIGHT_SPEED_KM_S;
    // ... 単位変換ロジック
}
```

教育的意義：
- 光速の遅さを実感（太陽-地球間で8分）
- 人間的スケールとの対比
- 天文学的距離の実感

### 4.3 インタラクティブなポップアップ

各天体をクリックすると詳細情報を表示：
- 軌道半径（天文単位）
- 実際の距離（億km）
- 到達時間（光速/新幹線/徒歩）
- Google Mapsリンク

```javascript
map.on('click', orbitLayerId, (e) => {
    const popupContent = `
        <div style="font-family: 'Segoe UI', sans-serif;">
            <h3>${planet.name}</h3>
            <p>軌道半径: ${planet.orbitAU.toFixed(2)} AU</p>
            <!-- 詳細情報 -->
        </div>
    `;
    new maplibregl.Popup({ maxWidth: '350px' })
        .setLngLat(coordinates)
        .setHTML(popupContent)
        .addTo(map);
});
```

### 4.4 地図スタイル切り替え

#### 4.4.1 状態保持機構
地図スタイル変更時に以下を保持：
- 現在の中心座標
- ズームレベル
- 太陽マーカー位置
- 軌道データ

```javascript
function changeMapStyle() {
    // 現在の状態を保存
    const currentCenter = map.getCenter();
    const currentZoom = map.getZoom();
    const currentSunCoords = sunCoordinates;

    // スタイル変更
    map.setStyle(MAP_STYLES[selectedStyle]);

    // 状態復元
    map.once('styledata', () => {
        map.setCenter(currentCenter);
        map.setZoom(currentZoom);
        sunCoordinates = currentSunCoords;
        updateOrbits();
    });
}
```

### 4.5 ドラッグ可能な太陽マーカー

```javascript
sunMarker = new maplibregl.Marker({
    element: el,
    draggable: true
}).setLngLat(sunCoordinates).addTo(map);

sunMarker.on('dragend', () => {
    sunCoordinates = sunMarker.getLngLat().toArray();
    updateOrbits();  // 軌道を再計算・再描画
});
```

ユーザーが任意の地点を中心に設定可能にすることで、
- 自分の居住地を太陽に設定
- 特定の地理的特徴との対応付け
- 探索的学習の促進

---

## 5. パフォーマンス最適化

### 5.1 レンダリング最適化
- **WebGLレンダリング**: MapLibre GL JSのGPU活用
- **ベクトルデータ**: ラスタ画像に比べ軽量で拡大縮小に強い
- **レイヤーの動的管理**: 必要なレイヤーのみ描画

### 5.2 計算の効率化
- **範囲外天体の除外**: 描画範囲を20,000kmに制限
- **条件分岐による最適化**: 天体タイプ別の処理
- **キャッシュの活用**: 縮尺係数の再利用

### 5.3 メモリ管理
```javascript
// レイヤー・ソース削除によるメモリ解放
if (map.getLayer(layerId)) {
    map.removeLayer(layerId);
}
if (map.getSource(sourceId)) {
    map.removeSource(sourceId);
}
```

---

## 6. ユーザーインターフェース設計

### 6.1 折りたたみ可能なパネル

```javascript
function togglePanel(contentId) {
    const content = document.getElementById(contentId);
    if (content.classList.contains('collapsed')) {
        content.classList.remove('collapsed');
        content.style.maxHeight = content.scrollHeight + 'px';
    } else {
        content.classList.add('collapsed');
    }
}
```

画面領域の効率的活用とユーザー制御の両立

### 6.2 レスポンシブデザイン
- `position: absolute`による柔軟なレイアウト
- `z-index`による階層管理
- モバイル対応のビューポート設定

### 6.3 視覚的フィードバック
- ホバー効果（`:hover`）
- トランジション（`transition`）
- カーソル変更（`cursor: pointer`）
- 無効化状態の表現（`:disabled`）

---

## 7. データモデルと天体定義

### 7.1 惑星データ構造
```javascript
const PLANETS = [
    {
        name: '水星',
        orbitAU: 0.39,              // 軌道半径（天文単位）
        diameterKm: 4879,           // 直径（km）
        color: '#A0A0A0',           // 表示色
        type: 'planet'              // 天体種別
    },
    // ... 他の天体
];
```

### 7.2 天体種別による描画の差別化
- **planet**: 主要惑星（線幅4px、不透明度0.6）
- **dwarf**: 準惑星（線幅4px、不透明度0.6）
- **region**: 領域（線幅2px、不透明度0.3、破線）
- **star**: 恒星（線幅3px、不透明度0.5、破線）

### 7.3 天文学的データの精度
- 軌道半径: 平均距離を使用
- 直径: 赤道直径を採用
- データソース: NASA、IAU等の公的機関データ

---

## 8. 技術的課題と解決策

### 8.1 地球の球面性への対応

**課題**: 地球は球体であり、平面地図への投影で歪みが発生

**解決策**:
- Turf.jsの測地線計算を使用
- Webメルカトル図法の特性を考慮
- 比較的小スケール（数千km）での使用に限定

### 8.2 極端なスケール差

**課題**: 太陽系内でも数十億倍のスケール差

**解決策**:
- 範囲外天体リストによる情報提供
- 地球周回数による相対的理解の促進
- 段階的なスケール設定の推奨

### 8.3 ブラウザ互換性

**対応**:
- Geolocation APIのフォールバック
- Fullscreen APIの対応確認
- CSS Flexboxによる柔軟なレイアウト

---

## 9. 教育的応用

### 9.1 学習目標
1. **スケール感の獲得**: 天文学的距離の直感的理解
2. **相対的関係の把握**: 惑星サイズと軌道距離の比較
3. **科学的思考**: 縮尺の概念と比例関係の理解

### 9.2 授業での活用例
- **小学校理科**: 太陽系の学習導入
- **中学校理科**: 天文単位の実感的理解
- **高校物理**: ケプラーの法則の視覚化
- **大学天文学**: スケール比較の教材

### 9.3 自主学習ツールとして
- 自分の住む場所を中心に設定
- 通学路や生活圏との対応
- 友人との情報共有（Google Mapsリンク）

---

## 10. 今後の発展可能性

### 10.1 機能拡張案
1. **軌道の楕円化**: より正確な軌道形状
2. **時間変化の実装**: 惑星の公転運動の可視化
3. **3D表示**: 軌道傾斜角の表現
4. **AR対応**: 実空間への投影
5. **マルチスケール対応**: ズームレベルに応じた天体切り替え

### 10.2 データ拡張
- 小惑星帯の密度分布
- 彗星の軌道
- 太陽系外縁天体
- 人工衛星や探査機の位置

### 10.3 インタラクション強化
- タイムスライダーによる時間制御
- 複数天体の同時比較
- ユーザー定義天体の追加
- シミュレーション機能（軌道計算）

### 10.4 教育コンテンツ連携
- クイズモード
- 学習進捗管理
- 解説動画の統合
- 多言語対応

---

## 11. 関連研究・先行事例

### 11.1 類似プロジェクト
- **If the Moon Were Only 1 Pixel**: ピクセル単位での太陽系スケール表現
- **Solar System Scope**: 3D太陽系シミュレーター
- **Celestia**: デスクトップ天文シミュレーション

### 11.2 本プロジェクトの独自性
- 実際の地理空間との対応付け
- 現在地ベースのパーソナライズ
- 軽量なWebアプリケーション
- オープンソース技術の活用

---

## 12. 技術的貢献

### 12.1 オープンソースへの貢献
- MapLibre GL JSの実践的活用例
- Turf.jsの教育利用事例
- Geolocation APIのUXベストプラクティス

### 12.2 再利用可能性
- フレームワーク非依存の設計
- モジュール化された関数群
- カスタマイズ容易なデータ構造

### 12.3 ドキュメント化
- コード内コメントの充実
- 関数の明確な命名
- 処理フローの可読性

---

## 13. まとめ

### 13.1 プロジェクトの成果
本プロジェクトは、Web地図技術を活用して天文学的スケールを地理空間に投影し、直感的な理解を促進する教育ツールを実現しました。以下の点で技術的・教育的貢献があります：

1. **技術面**:
   - MapLibre GL JSとTurf.jsの効果的な組み合わせ
   - パフォーマンスとUXを両立した実装
   - モダンなWeb技術の実践的活用

2. **教育面**:
   - 抽象的な天文学的概念の具体化
   - 能動的な探索を促すインタラクティブ性
   - 段階的な学習を支援する複数マップの提供

3. **発展性**:
   - 拡張可能なアーキテクチャ
   - オープンソースによる継続的改善の可能性
   - 多様な教育現場への適用可能性

### 13.2 学会発表に向けて
本技術資料は以下の観点での発表に活用できます：

- **教育工学**: インタラクティブ教材の設計と評価
- **情報可視化**: 多次元データの地理空間投影
- **Web技術**: オープンソース地図ライブラリの応用
- **天文教育**: デジタルツールを活用した概念理解

### 13.3 期待される効果
- 天文学への興味関心の向上
- スケール感覚の育成
- 科学的思考力の促進
- デジタルリテラシーの向上

---

## 付録A: 主要定数一覧

| 定数名                   | 値                    | 単位 | 説明                         |
| ------------------------ | --------------------- | ---- | ---------------------------- |
| EARTH_ACTUAL_DIAMETER_KM | 12,742                | km   | 地球の赤道直径               |
| SUN_ACTUAL_DIAMETER_KM   | 1,392,700             | km   | 太陽の直径                   |
| AU_IN_KM                 | 149,600,000           | km   | 1天文単位                    |
| LIGHT_SPEED_KM_S         | 299,792.458           | km/s | 光速                         |
| INITIAL_CENTER           | [135.40660, 34.14466] | 度   | 初期中心座標（みさと天文台） |

## 付録B: 主要関数一覧

| 関数名                            | 機能                   | 戻り値 |
| --------------------------------- | ---------------------- | ------ |
| `init()`                          | アプリケーション初期化 | void   |
| `createSunMarker()`               | 太陽マーカー作成       | void   |
| `updateSunDiameter()`             | 太陽直径の自動計算     | void   |
| `updateOrbits()`                  | 軌道の再描画           | void   |
| `calculateTravelTime(distanceKm)` | 到達時間計算           | Object |
| `changeMapStyle()`                | 地図スタイル変更       | void   |
| `setCurrentLocation()`            | 現在地取得             | void   |
| `togglePanel(contentId)`          | パネル開閉             | void   |

## 付録C: 参考文献・リソース

### 公式ドキュメント
- MapLibre GL JS: https://maplibre.org/
- Turf.js: https://turfjs.org/
- MDN Web Docs (Geolocation API): https://developer.mozilla.org/

### 天文データソース
- NASA Solar System Exploration: https://solarsystem.nasa.gov/
- International Astronomical Union (IAU): https://www.iau.org/
- JPL Horizons System: https://ssd.jpl.nasa.gov/horizons/

### 地図データ
- OpenStreetMap: https://www.openstreetmap.org/
- 国土地理院地図: https://maps.gsi.go.jp/

---

**Document Version**: 1.0  
**Last Updated**: 2025年12月13日  
**Author**: MapLibre Solar System Project  
**License**: MIT License (想定)
