# 第1章：WebAPIの基本

> 執筆者：高木宏輔
> 
> 最終更新：2026-04-17

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

この章では、iTunes Search APIを使用した非同期通信からデータを取得して、アプリ内に表示する方法を学ぶ。取得したJSONデータを使って音楽を検索し、その結果をリストビューへ反映させ表示させるものを学習する。

## 模範コードの全体像

```swift

// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル
//構造体にCodableをつける
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?
    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        //初期状態の表示
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

iTunesのAPIを使用し音楽を検索できるアプリ。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
```

**何をしているか：**

Codableを構造体に使用することでJsonの構造体に合わせて変換できるようにし対応させている。

**なぜこう書くのか：**

・SearchResponseを使うことによってAPIの形に合わせる
・Codableを使い安全かつ簡単に変換する
・IdentifiableはUIで扱いやすくする
・不完全データに対応するためにOptionalを使用する

**もしこう書かなかったら：**

特になし

---

### API通信の処理

```swift
func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

```

**何をしているか：**

ユーザーが入力した文字列を、URLで使える安全な英数字の形式に変換し、指定したURLへネットワークリクエストを送り、サーバーからデータを受け取りiTunesAPIから返ってきた生のJSONデータを、Swiftで定義したSearchResponseや Song構造体の型に変換してsongs配列に格納している。

**なぜこう書くのか：**

URL(string:)は無効な文字列を渡されるとnilを返すため、guard let を使って安全に取り出しています。await を付けることで「通信完了を待つ間、画面描画をブロックせずに別のタスクへ処理を譲ることができます。tryは通信エラーに備えるためです。Swiftは型安全な言語なので、JSONのまま扱うより構造体に変換したほうがプロパティに安全かつ簡単にアクセスできます。

**もしこう書かなかったら：**

同期処理を行うと、データ取得が終わるまでアプリの画面が完全にフリーズし、ボタンタップやスクロールが一切効かなくなります。(URL(string: urlString)!）などをした場合、何らかの理由でURL生成に失敗するとアプリがクラッシュします。

---

### ビューの構成

```swift
NavigationStack {
    VStack {
        // ① 検索バー（入力フィールドとボタンの水平配置）
        HStack {
            TextField("アーティスト名を入力", text: $searchText)
                .textFieldStyle(.roundedBorder)

            Button("検索") {
                Task {
                    await searchMusic()
                }
            }
            .buttonStyle(.borderedProminent)
            .disabled(searchText.isEmpty) // ボタンの無効化制御
        }
        .padding(.horizontal)

        // ② 条件分岐による表示の切り替え
        if isLoading {
            ProgressView("検索中...")
                .padding()
            Spacer()
        } else if songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(songs) { song in
                SongRow(song: song)
            }
        }
    }
    .navigationTitle("Music Search")
}
```

**何をしているか：**

テキストフィールドと検索ボタンを横並びに配置し、入力文字列が空のときはボタンを押せないよう無効化しています。アプリの状態が通信中、結果が空（初期状態含む）データありを判断し、表示するビューを完全に切り替えています。

**なぜこう書くのか：**

入力欄と決定ボタンを視覚的に関連付ける一般的なUIレイアウトであり、空の状態でAPIを叩く無駄な通信を防止するため。SwiftUIの @State変数の変化に反応して、ユーザーへ今何が起きているか（ロード中なのか、何も見つからないのか、結果一覧なのか）」を明確に伝えるためです。

**もしこう書かなかったら：**

通信中も真っ白な画面のままフリーズして見えたり、結果が無い時に何も表示されずユーザーが「動いているのか壊れているのか」判断できなくなります。文字を入力していない状態で検索ボタンを押せてしまい、空のキーワードで無駄なネットワークリクエストが飛んでエラーや想定外の挙動の原因になる。


---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| `async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `Identifiable`|各データ要素が一意のID（識別子）を持っていることを示すプロトコル。 | `struct Song: Identifiable { let trackId: Int var id: Int { trackId } }`|
| `JSONDecoder` | バイト列（Data 型）のJSONデータを、Codable に準拠したSwiftの型にデコードするクラス。|`let response = try JSONDecoder().decode(SearchResponse.self, from: data)` |
|`ContentUnavailableView` |検索結果が空の場合や初期状態、エラー発生時など、画面に渡すコンテンツがない状態を統一されたUIで表示するためのSwiftUIビュー。 | `ContentUnavailableView("曲を検索してみよう",systemImage: "music.note",description: Text("キーワードを入力してください"))`|

## 自分の実験メモ

**実験1：**
- やったこと：Song構造体のtrackId数値をString文字列に変えて検索を実行する。
- 結果：typeMismatchである型の不一致というエラーが出力される
- わかったこと：Codableが厳密に型をチェックしている。

**実験2：**
- やったこと：エスケープ処理をスキップする
- 結果：URL(string:) が nil を返し、検索が行われなくなる
- わかったこと：URLエンコードの必要性がわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
不完全データ以外にOptionalを使う場面はありますか
   **得られた理解：**
検索機能、ボタンやカスタムViewのイベント通知など未実施、非該当、安全な開放の状態を安全に表現するときに使われる。

2. **質問：**
オプショナルバインディングとはなんですか
   **得られた理解：**
Swiftで値が入っているかもしれないし、nilかもしれないというオプショナル型の変数から、安全に値を取り出して別の変数・定数に代入する仕組み

## この章のまとめ

非同期通信（データの取得）データ構造（受け皿）SwiftUIの表示（画面への反映）をいかに安全かつ明確に繋ぐかが重要になる。アプリが動かない、クラッシュする、更新されないとなったらURLは正しいか、JSONと構造体の型は合っているか、UIの状態管理（@State）は更新されているかを確認するようにしたい。
