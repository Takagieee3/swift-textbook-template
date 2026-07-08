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
見た目が悪くなるだけでなく、データが壊れたり勝手に消えたりする危険なアプリになってしまう。

---

### @Queryによるデータ取得

```swift
// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    
    // ↓ ここが @Query によるデータ取得部分です
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
```

**何をしているか：**
@Query というマクロを配列Memoの前に付けるだけで、SwiftDataが背後にあるデータベースからすべての Memo データを自動的に引っ張ってきて、このmemos変数に格納してくれます。
引数の `(sort: \Memo.createdAt, order: .reverse) `によって、作成日時（createdAt）が新しい順（reverse＝降順)に並んだ状態でデータが取得されます。

**なぜこう書くのか：**
データベースを常に監視する役割がありデータの追加・編集・削除が行われると感知しListを自動的に最新状態
に再描画してくれるから。またコード量が減りバグが少なく済む。

**もしこう書かなかったら：**

@Queryがない場合

1.取得したデータを保持しておくための `@State private var memos: Memo = []`

2.データベースからデータを取ってくるための関数 `func fetchMemos()`（エラーハンドリングの do-catch 文付き）

3.画面が表示された瞬間にその関数を実行する `.onAppear { fetchMemos() }`

4.新しいメモを追加・削除・編集した後に、手動で再度データを読み直すための fetchMemos() の呼び出し
これらが必要になる。また並び替えの順番など後からの変更が面倒になる。

---

### @AppStorageによる設定保存

```swift
// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    
    // ↓ ここが @AppStorage による設定保存の部分
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    
    @State private var isShowingAddSheet = false
```

**何をしているか：**
この部分では、「ユーザーの名前」や「アプリの表示設定（お気に入り優先）」といった、アプリ全体の気軽な設定データをスマホ内に保存（永続化）しています。変数の中身を書き換えた瞬間に、裏側で自動的に保存処理まで完了してくれる。

**なぜこう書くのか：**
SwiftDataメモのように同じ形のデータが1件、2件、100件…と増えていくものなど値が変わったら画面をリロードするため保存するのに向いているから。

**もしこう書かなかったら：**
通常のUserDefaultsには、SwiftUIの画面を再描画させる機能がありませんなので設定を変えても画面が更新されないバグが起きる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
|`@AppStorage` | アプリの設定データを、スマホ内に超カンタンに保存・自動同期できる、SwiftUI専用のプロパティラッパー | `@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false` |
|`ModelContext` | 「追加（.insert()）」や「削除（.delete()）を、オブジェクトをそのまま放り込むだけで直感的に指示する文法。 | `@Environment(\.modelContext)` |
|`class`|SwiftUIの画面を作るViewはすべて struct（構造体)ですが、SwiftDataで定義するMemoモデルは`class`で作られています。| `final class Memo {//以下varが続く`|

## 自分の実験メモ

**実験1：**
- やったこと：既存の Memo クラスに、最終更新日を管理するための updated: Date プロパティを追加し、初期化時（init）に作成日と同じ日時が自動で入るように変更しました。
- 結果：メモのデータが「作った日時」だけでなく、「最後にいつ触ったか（updated）」も内部で正しく記憶できるようにしたかった。（失敗）
- わかったこと：データ取得はデータベース（@Query）で大まかに行い、細かい並び替えはSwiftのコード（.sorted）で行うこと

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
    構造体（Struct）ではデータがメモリ上の違う場所になるということですか
   **得られた理解：**
   structを使用するとメモリ上でコピーが作られてしまい一覧画面に戻ったが古いまま変わらないという現象になる。今回の場合Classを使用することが良い。

2. **質問：**
   @Queryを使用するときはデータベース専用に近いですか
   **得られた理解：**
    @Queryはデータベースから自動でデータを持ってくる専用パーツである
3. **質問：**
    @Queryの前は何を使っていましたか
   **得られた理解：**
   `@FetchRequest`を使っており違いとして@QueryはSwiftDataという最新用にあるもので現在では
   `@FetchRequest`を使用していない（Core Data）で必須とされていた。

## この章のまとめ
@Queryなどのデーターベースにまつわる仕様を考え、自動でやってくれる良いものであると理解できた。また@FetchRequestはCoredataに使用するもので時代によって最新のものをつけていく必要があるとわかりました。

