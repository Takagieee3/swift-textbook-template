# 第5章：機能統合の実践

> 執筆者：高木　宏輔
> 最終更新：2026-07-08

## この章で学ぶこと

これまでに学習した写真＋地図＋データの保存を組み合わせたアプリを学習する。複数のフレームワークから得た
データを一つのPhotoRecodeに結合し、永続化する。・画像データ・位置情報、緯度、経度・テキストこれらを
保存、管理について学ぶ。

## 模範コードの全体像

```swift

// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}

```

**このアプリは何をするものか：**

写真といる場所をGPSを使いセットで記録してくれる。またライブラリから写真を選びタイトルやメモを添えて保存します。地図を開くと保存した写真のアイコンが地図上に表示される。リストで表示し日付順で表示され不要になったら簡単に削除可能である。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
１つの思い出のデータの設計図を定義している。タイトルやメモといったテキスト情報、位置情報、画像データ、作成日時を1つのオブジェクトとしてパッケージ化し、それをデバイスのフォルダに永久保存できるようにしています。

**なぜこう書くのか：**
@Modelをつけることにより、Swiftが自動でDBのテーブルを作成しデータの追加・削除・自動保存を行える。
UIImage は画面に表示するためのUI部品(オブジェクト)であり、ファイルとしてそのまま保存できません。そのため、画像ファイルをデータのかたまりであるData型に変換して保存する設計にしている。

**もしこう書かなかったら：**
@Modelを付けなかったら、SwiftDateの管理対象から外れるためIPhoneのストレージに書き込まれずデータが消えてしまう。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }
            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**

・アプリの画面下にマップと一覧を切替えるタブバーTabViewを設置している。
・各画面MapTabとListTabにアイコンとテキストのセット.tabItemを割り当て、タップで画面を切り替えられるようにしている。

**なぜこう書くのか：**

・地図で探すとリストで探すという目的の異なる2つの主要画面を、ユーザーがいつでも1タップで行き来できるようにするため。

**もしこう書かなかったら：**

1つの画面（例:地図）しか表示できなくなり、リスト一覧を見せるためにわざわざ別画面へ遷移するボタンNavigationLinkなどを押し込まなければならず、アプリの操作性が格段に悪くなる。

---

### カメラと位置情報の連携

```swift
// MARK: - 位置情報マネージャー
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?
    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - 記録追加画面（一部抜粋）
// ① PhotosPickerで写真を選ぶ
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選択", systemImage: "photo")
}

// ② 写真選択＆保存時に「現在の位置情報」と「写真」を合体させて保存
func saveRecord() {
    guard let location = locationManager.currentLocation else { return }
    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )
    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**

・バックグラウンドでスマホのGPSLocationManagerを常に動かして現在地を取得し続ける。
・ユーザーがPhotosPickerで写真を選んで保存ボタンを押した瞬間のGPS座標currentLocationと写真データを1つの記録PhotoRecordとしてセットで保存しています。

**なぜこう書くのか：**

写真追加時に手動で「ここがどこか」を入力させるのではなく、アプリが全自動で撮影地（現在地）の緯度経度を判別して記録に紐付ける設計にするためです。

**もしこう書かなかったら：**

requestWhenInUseAuthorization()やstartUpdatingLocation()がないとGPSが動作しないため、写真を追加してもどこで撮った写真かのデータが空になり、地図連携アプリとして成立しなくなります。

---

### SwiftDataでの画像保存

```swift
// ① 写真ライブラリから選択した「アイテム（PhotosPickerItem）」の変更を検知
.onChange(of: selectedItem) { _, newItem in
    Task {
        // 写真データ（Data型）を非同期でロードして保持する
        if let data = try? await newItem?.loadTransferable(type: Data.self) {
            selectedImageData = data
            if let uiImage = UIImage(data: data) {
                previewImage = Image(uiImage: uiImage)
            }
        }
    }
}

// ② 保存処理：取得した Data 型をそのまま SwiftData モデルに渡す
func saveRecord() {
    let record = PhotoRecord(
        title: title,
        // ...（中略）...
        imageData: selectedImageData // Data? 型として保存
    )
    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**

・PhotosPickerで選んだ写真を、アプリで保存できるバイト列データData型に変換し、一時変数selectedImageDataに格納しています。
・保存ボタンが押された際、そのData型をそのままPhotoRecordに渡してデータベースへ永続化しています。

**なぜこう書くのか：**

データベースに保存できる形式がData型だから。
UIImageなどの画像オブジェクトはメモリ上の表現であり、そのままデータベースに保存できません。万能なデジタルデータ形式であるData型として持ち替えることで、SwiftDataに安全に保存させることができる。

**もしこう書かなかったら：**

UIImageのまま直接SwiftDataのモデルに保存しようとするとコンパイルエラーになります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `PhotosPicker` | iOS標準の写真選択画面を呼び出すSwiftUI専用コンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images) { Text("写真を選択")}` |
| `loadTransferable` | 選択された写真から、非同期で実際の画像データを安全に読み取るAPI | `if let data = try? await selectedItem?.loadTransferable(type: Data.self) {self.selectedImageData = data `|
|`UserAnnotation` |自分の現在地をマーク |`Map(position: $cameraPosition) { UserAnnotation()` |

## 自分の実験メモ

**実験1：**
- やったこと：ピンの見た目を変更
- 結果：四角になり写真が見やすくなった
- わかったこと：systemNameの周辺を変更することにより変更ができる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
2. ・UserAnnotation()は具体的に何をするところですか
   **得られた理解：**
   ・スマホの移動に合わせてリアルタイムで追従するために必要である

4. **質問：**
5. Data型で保存する場合どのくらい保存できますか
   **得られた理解：**
   スマホの空き容量次第であるが追加し続けると容量を圧迫する

7. **質問：**
8. CLLocationCoordinate2D以外に座標データを扱えるものはありますか
   **得られた理解：**
   今回のような地点を表すならCLLocationCoordinate2Dを使用し、他にも3種類ほど使い分けができる。

## この章のまとめ

写真とGPS座標をリンクさせ登録することでマップ上にピンで表示させることができる。
