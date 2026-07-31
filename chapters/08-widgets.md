# 第8章：ウィジェット

> 執筆者：高木　宏輔
> 最終更新：2026-07-31

## この章で学ぶこと

Codable,Identifiableに準拠した構造体設計と、CalendarAPIを利用して今日の日付に応じたデータを動的に切り替えるロジックを学び、TimelineProviderを用い、読み込み中、追加時、将来の自動更新という3つの表示状態を管理する仕組みを学びます。

## 模範コードの全体像

```swift
// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}
import Foundation

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]
    //dateは日付で中身は数値
    static func todaysQuote() -> Quote {
        
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
```

**このアプリは何をするものか：**

日替わりで指定された今日の名言がアプリとホーム画面にあるウィジェット上で自動更新して表示するアプリです。またウィジェットではアプリを開かずに毎日新しい名言を楽しめます。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
func placeholder(in context: Context) -> QuoteEntry {
    QuoteEntry(
        date: Date(),
        quote: Quote(id: 0, text: "読み込み中...", author: "")
    )
}
func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
    let entry = QuoteEntry(
        date: Date(),
        quote: QuoteStore.todaysQuote()
    )
    completion(entry)
}
func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
    let currentDate = Date()
    let quote = QuoteStore.todaysQuote()
    let entry = QuoteEntry(date: currentDate, quote: quote)

    // 次の日の0時にウィジェットを更新
    let tomorrow = Calendar.current.startOfDay(
        for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
    )

    let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
    completion(timeline)
}
```

**何をしているか：**

placeholder(in:)でウィジェット追加時や読み込み中に表示する仮画面を返す。getSnapshot(in:completion:)は追加画面で表示するプレビュー用データを即座に返す。

**なぜこう書くのか：**

ウィジェットはアプリのようにバックグラウンドで常にプログラムを動かし続けることができません。そのため、次の日の0時になったら一度起こして、新しい画面を描画してねと更新時刻だけを予約しています。

**もしこう書かなかったら：**

日付が変わっても前の日の名言がそのまま残り、アプリを開くまでウィジェットが更新されなくなります。iOSのシステム制限に達して更新がブロックされ、意図したタイミングで更新されなくなります。
---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}
```

**何をしているか：**

entryで受け取ったデータを画面に描画しています。また、@Environment(\.widgetFamily) を使うことで、ユーザーが設置したウィジェットのサイズに合わせてレイアウトを切り替えています。

**なぜこう書くのか：**

entryから名言データを取り出してテキスト表示に反映させています。ホーム画面の小サイズと中サイズでは表示できる面積が大きく異なります。サイズごとにsmallWidgetとmediumWidgetを分けることで、どちらのサイズでも綺麗に見せるためです。

**もしこう書かなかったら：**

小サイズにしたときに文字がはみ出て途切れたり、逆に中サイズにしたときに余白だらけでスカスカに見えたりするなど、デザインが大きく崩れてしまいます。TimelineProviderが更新した最新データがビューに届かず、日付が変わってもウィジェットの表示が古いまま変わらなくなってしまいます。

---

### ウィジェットサイズごとのレイアウト

```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family // ① ウィジェットのサイズを取得

    var body: some View {
        // ② サイズに応じて表示するビューを分岐
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // ③ 小サイズ用レイアウト（縦並び・コンパクト）
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // ④ 中サイズ用レイアウト（横並び・ゆったり）
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}
```

**何をしているか：**

@Environment(\.widgetFamily)を使って、ユーザーがホーム画面に配置したウィジェットの現在のサイズを取得しています。
switch familyでサイズを判定し、小サイズなら smallWidget、中サイズならmediumWidgetというように、異なるSwiftUIのビューを返しています。

**なぜこう書くのか：**

・小サイズは横幅がないため、アイコン・本文・著者を上から下に縦並びにし、フォントサイズも小ぶりにして文章が途切れにくいよう工夫しています。
・中サイズは横幅に余裕があるため、左側に大きめのアイコン、右側に名言テキストを横並びで配置し、見やすくゆとりのあるレイアウトに調整しています。

**もしこう書かなかったら：**

サイズ分岐をせず1つの共通レイアウトだけで書いてしまうと、小サイズウィジェットに配置された際に文字が途中で見切れて読めなくなったり、レイアウトが極端に潰れたりします。逆に小サイズ用のレイアウトを中サイズに表示するとアンバランスな見た目になる。

---

### メインアプリとの連携

```swift
// ① 【共通ロジック】アプリとウィジェットで全く同じデータストアを参照
struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [ /* 名言データリスト */ ]
    
    // 日付（年間通算日）から「今日の名言」を1つ決定するロジック
    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// ② 【メインアプリ（ContentView）での呼び出し】
struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote() // ← 共通ロジックで取得
    @State private var allQuotes = QuoteStore.quotes // ← 共通データから一覧取得
    // ...
}

// ③ 【ウィジェット（QuoteProvider）での呼び出し】
struct QuoteProvider: TimelineProvider {
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote() // ← メインアプリと同じロジックで取得
        )
        completion(entry)
    }
    // ...
}
```

**何をしているか：**

アプリ本体とウィジェットの両方から参照できる単一のデータ管理構造を定義しています。QuoteStore.todaysQuote()という共通の関数を呼び出すことで、日付に応じた名言を1つ選ぶという判定ロジックを両者で共有しています。

**なぜこう書くのか：**
アプリとウィジェットで別々に今日の名言を計算してしまうと、コードの書き間違いや修正漏れによってアプリを開いた時の表示とホーム画面ウィジェットの表示で違う名言が出てしまう問題が起こります。

**もしこう書かなかったら：**

アプリ側とウィジェット側で別々の配列や計算ロジックを書いていると、日付切り替わりの計算が1日ズレてしまい、アプリとウィジェットで言っていることが違うという不具合が発生しやすくなります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| `TimelineEntry` | WidgetKitでいつどんなデータを表示するかを定義する標準プロトコル。dateプロパティの保持が必須 | `import WidgetKitstruct SimpleEntry: TimelineEntry {let date: Datelet message: String}` |
|`containerBackground` |iOS17から導入された、ウィジェットの背景やスマートスタック表示に合わせたバックグラウンド領域を設定するModifier。 | `struct MyWidgetEntryView: View {var entry: SimpleEntryvar body: some View {VStack {Text(entry.message)}.containerBackground(.fill.tertiary, for: .widget)}}`|

## 自分の実験メモ

**実験1：**
- やったこと：シャッフルボタンをアプリに追加
- 結果：指定された名言以外が出てくる
- わかったこと：データモデルへの関数追加の書き方。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
ウィジットはどのような時に必要ですか
**得られた理解：**
ウィジェットが必要となるのは、主にユーザーがアプリを起動する一手間を省いて、ひと目で情報を確認・操作したい時(天気、カレンダーなど)

2. **質問：**
ウィジットはいつから出ましたか
**得られた理解：**
2014に画面の上からスワイプして出す通知センター内に、初めて他社アプリ製ウィジェットが置けるようになりました。また今回使用した仕組みは2020以降に確立されたものである。
3. **質問：**
TimelineProviderの解説が欲しい。
**得られた理解：**
いつどんな画面を表示すれば良いかを予約する仕組みでバックグラウンドを動き続けないように省電力化するために用いられる。

## この章のまとめ

ウィジェットはアプリではなくOSに渡すタイムラインであり、バッテリーを保護しながら指定時刻に画面を差し替える仕組み。アプリと表示を一致させるにはロジックの共通化を徹底し、@Environment(\.widgetFamily) でサイズに合わせた最適なUIへ切り替えることが重要である。
