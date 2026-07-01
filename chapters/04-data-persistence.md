# 第4章：データの永続化

> 執筆者: 高木　宏輔
> 最終更新：2026-06-24

## この章で学ぶこと

クラスに@Moddelをつけるだけでデータベースのテーブルとして扱え、@Queryでは自動更新など永続化を学び
、双方向性のバインディングや@AppStorageによる永続化。そしてContentUnavailableViewによる空き状態の表示などを学びます。

## 模範コードの全体像

```swift
import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

+を押しメモの新規作成を行い、タイトル、内容を打ち込む→保存。タイトル画面には作成したメモが一覧表示
されており、作成したメモをタップするとお気に入りをすることができお気に入りがメモ一覧の上部にくる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**
クラスの直前に@Modelを付けることでこのクラスがそのままデータベースのテーブル構造として定義される。
titleやcreateAtなど内部のプロパティが自動的にデータベースの列（カラム）として扱われる。

**なぜこう書くのか：**
modelContextでオートセーブが働くのでdo {try} catchなどを書かずに済み、@Queryと連動して画面が自動更新されリアルタイムにデータを操作できるから。

**もしこう書かなかったら：**
保存を確定させるために
```swift
modelContext.insert(memo)
do {
    try modelContext.save() // いちいち明示的に保存が必要だった
} catch {
    print("保存失敗")
}
```
このようなものを書かなければならず冗長になると思いました。
---

### データの追加・削除（modelContext）

```swift
ToolbarItem(placement: .confirmationAction) {
    Button("保存") {
        // 1. 入力されたタイトルと内容で、新しいMemoインスタンス（オブジェクト）を作成
        let memo = Memo(title: title, content: content)
        
        // 2. modelContextを使ってデータベースに挿入（保存）
        modelContext.insert(memo)
        
        dismiss()
    }
    .disabled(title.isEmpty)
}
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        // 1. 削除対象の行（インデックス）から、該当するMemoオブジェクトを特定
        let memo = displayedMemos[index]
        
        // 2. modelContextを使ってデータベースから削除
        modelContext.delete(memo)
    }
}
```

**何をしているか：**

ユーザーが入力したタイトルと内容をもとに、新しく作ったメモのデータ（memo）をデータベースのテーブルに新しく追加（登録）しています
modelContext.insert(memo)
ユーザーが画面上でスワイプして削除しようとした特定のメモを特定し、それをデータベースから完全に削除しています。
modelContext.delete(memo)

**なぜこう書くのか：**
不正なデータ（空のメモ）を最初から作らせない、iOS標準の快適なスワイプ削除機能に乗っかる、画面の表示順と連動させて、絶対に消し間違いを起こさない
アプリ開発において重要な安全性と使いやすさを少ないコードで実現するために選ばれた洗練された書き方だから。


**もしこう書かなかったら：**

---

### @Queryによるデータ取得

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### @AppStorageによる設定保存

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
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
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
