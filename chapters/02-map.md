# 第2章：地図アプリの基本

> 執筆者：高木　宏輔
> 最終更新：2026-05-20

## この章で学ぶこと

この章では、SwiftUIとMapkitを使用して地図アプリを作成する基本を学び観光スポットの情報を構造体で管理しMapKitを使用し地図上に複数のマーカーを表示する詳細情報をカード形式で表示する。

## 模範コードの全体像

```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}


```

**このアプリは何をするものか：**

SwiftUIとMapKitフレームワークを使用して、地図上に東京の観光スポットを表示するiOSアプリ

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    // Hashableへの準拠（MapKitでの選択処理に必要）
    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    // カテゴリ定義
    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        // カテゴリに応じたアイコン
        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        // カテゴリに応じたカラー
        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}
```

**何をしているか：**
ランドマークの基本情報、座標、およびカテゴリ別のスタイル（アイコンや色）を管理する役割を担っています。

**なぜこう書くのか：**

この書き方は、「MapKitの高度な機能（タップ選択）を使いつつ、将来的にカテゴリやデータが増えても壊れにくい、メンテナンス性の高い設計」を目指した結果と言えます。アプリとして実用的な機能（選択・抽出・整理）を持たせるには、この構成が最もバランスの良い正攻法です。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671), // 東京駅付近
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)       // ズーム倍率
    )
)
Map(position: $cameraPosition, selection: $selectedLandmark) {
    // ここにマーカーなどのコンテンツを配置
}
.mapStyle(.standard(elevation: .realistic)) // 地図のスタイル設定
.onMapCameraChange { context in
    // 地図の操作に応じた処理を追加できる
    // 例：現在の中心座標をログに出す、表示範囲に合わせてデータを再読み込みするなど
}
```

**何をしているか：**

まず、カメラが「どこを、どのくらいの範囲で映すか」を保持する変数、body 内で地図をレンダリングしている部分、ユーザーが地図をスクロールしたりピンチイン・アウトしたことを検知するトリガーです。

**なぜこう書くのか：**

この書き方をする最大のメリットは、「地図という複雑なUIを、ただの変数の更新として扱えるようになる」ことです。
地図の状態（データ）はどうあるべきかを記述するだけで済むようになります。

**もしこう書かなかったら：**

今の書き方をしなかったら、地図の状態を管理するための管理職（コード）が大量に必要になる

---

### マーカーの表示

```swift
ForEach(filteredLandmarks) { landmark in
    Marker(
        landmark.name,                           // マーカーの下に表示されるタイトル
        systemImage: landmark.category.iconName, // アイコン（SF Symbols）
        coordinate: landmark.coordinate          // 緯度・経度
    )
    .tint(landmark.category.color)               // マーカーの色
    .tag(landmark)                               // タップ判定用の識別子
}
```

**何をしているか：**

地図上に標準的な形のマーカーを表示します。カテゴリに合わせて「お寺（建物の柱）」「タワー（電波）」「公園（葉っぱ）」のアイコンを表示します。
マーカーの土台の色（赤、青、緑など）を指定します。このマーカーに「このデータ（landmark）ですよ」という印を付けます。

**なぜこう書くのか：**

文字だけでなくアイコンがあることで、ユーザーが直感的に「そこが何の場所か」を判別できるようにするためです。カテゴリごとに色を分けることで、視認性を高めています。Map(selection: $selectedLandmark) と連動しており、ユーザーがピンをタップした際、この .tag に設定したデータが自動的に selectedLandmark という変数に代入されます。これにより「タップされた場所の詳細を表示する」という動きが実現できます。

**もしこう書かなかったら：**

もしMarkerではなく、自分でもっと自由にデザインしたい場合（例えばスポットの写真を表示したい場合など）は、Annotation というパーツを使います。

---
```swift
// デザインを自由に変えたい場合の例
Annotation(landmark.name, coordinate: landmark.coordinate) {
    Image(systemName: "star.fill") // 星型にしたり、画像にしたりできる
        .padding(4)
        .background(Color.yellow)
        .clipShape(Circle())
}
```
---

### フィルター機能

```swift
// カテゴリのセット（重複のない集合）を保持
// 初期状態は全カテゴリ(allCases)が入っている
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)
var filteredLandmarks: [Landmark] {
    // 全データ（sampleData）をチェック
    Landmark.sampleData.filter { landmark in
        // その場所のカテゴリが、選択中のセットに含まれているものだけを残す
        selectedCategories.contains(landmark.category)
    }
}
Button {
    if selectedCategories.contains(category) {
        // すでに選択されていたら、除外する
        selectedCategories.remove(category)
    } else {
        // 選択されていなかったら、追加する
        selectedCategories.insert(category)
    }
} label: {
    // 見た目の処理（選択中なら色を付け、そうでなければグレーにする）
}
```

**何をしているか：**

まず、「今どのボタンが押されているか」を記録しておく場所です。
全データの中から、地図に表示するものだけを抽出するフィルターの本体です。
ユーザーが画面下のボタンをタップした時の処理です。

**なぜこう書くのか：**

なぜ Set か: 「寺社」と「公園」のように複数選択が可能で、かつ「すでにあるなら消す、ないなら入れる」という処理が高速に行えるためです。
元データを直接書き換えるのではなく、「今の選択状況なら、このリストが見えるはず」という計算結果をリアルタイムで導き出すためです。ボタンを押して selectedCategories が変わるたびに、この計算が走り、地図上のピンがパッと消えたり現れたりします。
ON/OFFの切り替え（トグル動作）を実現するためです。このボタン操作によって selectedCategories が更新されると、上記（2番）の filteredLandmarks が自動的に再計算され、最終的に Map の表示が更新されるという連鎖が起きます。

**もしこう書かなかったら：**

「データ」と「表示」を切り離してバラバラに命令すると、コードが複雑に絡み合い、ボタンの操作と地図の表示が一致しなくなる「状態の矛盾（バグ）」を引き起こす原因になります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `@Binding` | 親ビューが持っている@Stateへの参照を子ビューに渡す文法 | `struct FilterButton: View {@Binding var isOn: Boolvar body: some View {Toggle("フィルター", isOn: $isOn)}}` |
| `Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
|`.mapStyle` | 地図の見た目を一言で変更できるモディファイア|`Map().mapStyle(.standard(elevation: .realistic))` |
| `UIViewControllerRepresentable`| SwiftUIにまだ存在しない機能や、カメラ（UIImagePickerController）などの既存のUIKit部品をSwiftUIに持ち込むためのプロトコル| `import SwiftUI import UIKit struct CameraView: UIViewControllerRepresentable {func makeUIViewController(context: Context) -> UIImagePickerController { let picker = UIImagePickerController() picker.sourceType = .camera return picker}func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}}`|

## 自分の実験メモ

**実験1：**
- やったこと：スポットを選択した時にカメラを自動で移動させる
- 結果：タップした場所が自動的に地図の中心に来るようになった。
- わかったこと：見やすさは最初のでも良いと思うが簡単に変えられることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
landmarkの判定について教えて
   **得られた理解：**
タップ判定ではHashableに準拠したLandmarkを.tag(landmark)で渡すことで、Map(selection:)がタップされたピンを自動特定する。表示判定はselectedCategories.contains(...) の条件文で、配列から合致するデータだけを抽出する。

2. **質問：**
UUIDとはなんですか
   **得られた理解：**
世界中で絶対に他のデータと重複しないランダムなIDを自動生成する仕組み
3. **質問：**
ランドマークを追加する場合  Landmark()に追加していけば良いですか
   **得られた理解：**
Landmark.sampleDataの配列の中にLandmark(...) を追加していくだけで、地図上のマーカーやフィルター機能に自動的に反映されます。

## この章のまとめ

地図やピンを直接操作するのではなく、データを正しく管理すれば、表示はすべてSwiftUIが自動で辻褄を合わせてくれる。$をつけて Mapに渡すことで、ユーザーの操作結果がそのまま自分の変数に書き込まれる。データに一意なIDを持たせることは、SwiftUIで動的なリストや地図を扱うための絶対条件。
