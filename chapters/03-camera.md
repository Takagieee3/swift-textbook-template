# 第3章：カメラの利用

> 執筆者：高木宏輔
> 最終更新：2026-05-20

## この章で学ぶこと

PhotosPickerを使用して、安全かつ簡単にフォトライブラリから写真を取得する方法を学びUIViewControllerRepresentableを介してSwiftUIだけでは直接扱えないカメラ機能を呼び出し、撮影した画像をアプリ内に取り込む選択されたデータから画像を表示可能な形式へ変換する非同期処理や、取得した画像の動的な画面表示・更新について学習する

## 模範コードの全体像


```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、画面に表示します。
// 「カメラ」ボタンで撮影もできます。
//
// 【動作環境】
//   - フォトライブラリから選択：シミュレータでも動作します。
//   - カメラ撮影：実機（iPhone / iPad）専用。シミュレータでは
//     カメラボタンが自動的に無効化されます。
//
// 【注意】実機でカメラを使う場合は Info.plist に以下を追加してください：
//   - NSCameraUsageDescription
//     値: "撮影した写真を表示するためにカメラを使用します"
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影（シミュレータには未搭載のため自動的に無効化）
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

iPhoneのカメラや写真ライブラリを使って、好きな画像をアプリ内に取り込み、画面に表示させるためのシンプルなアプリ

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
@State private var selectedItem: PhotosPickerItem? // 選択された情報を保持
@State private var selectedImage: Image?          // 実際に表示する画像を保持
// フォトライブラリから選択
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
// フォトライブラリから選択
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**

ユーザーが写真を選んだ瞬間にそのデータがアプリ内に取り込まれ、画面が更新される仕組みになっている。

**なぜこう書くのか：**

ユーザーがこれと選んだその1枚だけのデータがアプリに渡される仕組みなので、安全性が非常に高いかつawaitを使うことで、読み込みが終わるまで、この処理は裏側で待機させてね。その間、他の画面操作は止めないでねという命令をしている。

**もしこう書かなかったら：**

AIから最も手軽で安全な標準の書き方と評されている。

ifこう書かなかったら

・連動を忘れると：写真を選んでも画面が変わらない
・フィルターを忘れると：動画が選ばれてアプリが困惑する

---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
    // Task { ... } を使うことで「ここから先は非同期（裏側）でやってね」と指示する
    Task {
        await loadImage(from: newItem) // await で処理の完了を待つ
    }
}
// async がついているため、この関数全体が非同期で実行される
func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }

    do {
        // try await：データの読み込みが終わるまで、この行で「一時停止」して待つ
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            
            // 読み込みと変換がすべて完了したら、最後にメイン画面の画像（@State）を更新する
            selectedImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像の読み込みに失敗: \(error.localizedDescription)")
    }
}
```

**何をしているか：**

写真が選ばれたことを検知する(.onChange)をしTaskを使うことでバックグラウンド処理の指示、データの読み込みを「待ち合わせ」する(try await)を使い最後の selectedImage = Image(uiImage: uiImage) で変数に画像がセットされたら写真が表示される。

**なぜこう書くのか：**

・Task で重い処理を裏側に放り投げて、画面のフリーズを防ぐため。
・await で、複雑な裏側処理を「上から下へ」読みやすく書くため。
・try で、データ破損などの予期せぬエラーからアプリが落ちるのを守るため。

**もしこう書かなかったら：**



---

### UIViewControllerRepresentableによるカメラ連携

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### Coordinatorパターン

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
