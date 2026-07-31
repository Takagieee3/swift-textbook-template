# 第6章：ジェスチャー操作

> 執筆者：高木　宏輔
> 最終更新：2026-07-24

## この章で学ぶこと

6章はユーザーの各種ジェスチャーの認識の方法を学習する。タップ、ロングプレス、ドラッグ操作、ピンチイン、ピンチアウト、2本指の回転など各種ジェスチャーの実装方法を学習し、ジェスチャーを組み合わせたアプリにする。

## 模範コードの全体像

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                // 複数のジェスチャーを「同時に」効かせるには
                // .gesture を重ねるのではなく .simultaneousGesture を使う
                .simultaneousGesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .simultaneousGesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
.onTapGesture {
    tapCount += 1
    backgroundColor = Color(
        hue: Double.random(in: 0...1),
        saturation: 0.7,
        brightness: 0.9
    )
}
.onLongPressGesture(minimumDuration: 1.0) {
    isPressed = true
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        isPressed = false
    }
}
```

**何をしているか：**
ユーザーが対象の円を1秒間以上押し続けたこと を検知します。
条件を満たすと円が緑色になり拡大その1秒後に元のオレンジ色・標準サイズへ戻すタイマー処理を動かしています。ビューが1回タップされたタイミングを検知し、クロージャ内の処理を実行しています。ここでは、タップされた回数を1増やし、カードの背景色をランダムな色に変更しています。

**なぜこう書くのか：**
minimumDuration: 1.0　誤操作を防ぐため、長押しと判定するまでの待ち時間を明示的に1秒と定義しています。
DispatchQueue.main.asyncAfter　状態を成功に変えた後、ユーザーが指を離しても一定時間後に自動で通常状態にリセットさせるために記述する

**もしこう書かなかったら：**
minimumDuration を指定しなかった場合デフォルトの長押し時間（通常0.5秒）が適用されます。より長時間の押し込みを要求したいUIでは、反応が早すぎてユーザーの意図とズレる原因になる。
---

### ドラッグジェスチャーとオフセット管理

```swift
// ① 状態変数の定義
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero

// ② ビューへの適用
.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)
```

**何をしているか：**
指を動かしている間、前回の確定位置に今回の指の移動量を足し合わせて、一時的な現在地をリアルタイムに更新しています。
指を離した瞬間に、その時点の移動完了位置offsetをlastOffsetに保存して次回へ引き継ぎます。

**なぜこう書くのか：**
SwiftUIのDragGestureが提供するvalue.translationは、指を画面につけた瞬間を原点(0, 0)とした移動量を返します。1回目のドラッグが終わった後、2回目のドラッグを始めると value.translation は再び (0, 0) からカウントが始まります。過去のドラッグ位置を保存しておくlastOffsetを用意し、今回の移動量と足し合わせることで、前回指を離した場所からスムーズに次のドラッグを開始できるようにする。

**もしこう書かなかったら：**
lastOffsetを使わず、以下のようにoffset = value.translationだけ書くと
```swift

.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
        }
)
```
value.translationの更新が止まるため、とりあえず右下に留まります。しかし2回目でドラッグで指を触れた瞬間value.translationが0, 0にリセットされるため、カードが画面中央へ一瞬でワープしてしまいます。

---

### 拡大縮小と回転

```swift
// ① 状態変数の定義
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0

// ② ビューへの適用
.font(.system(size: 100))
.foregroundStyle(.yellow)
.frame(width: 300, height: 300)      // タッチ領域の枠
.contentShape(Rectangle())           // 透明部分もタッチ可能に
.scaleEffect(scale)                  // 倍率を適用
.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)
```

**何をしているか：**
MagnifyGesture():2本指でつまむ・開く動作を検知します。onChanged:ピンチ操作中、前回の確定倍率に今回のピンチ倍率を掛け算して、現在の倍率をリアルタイムに更新します。

**なぜこう書くのか：**
ピンチ操作のvalue.magnificationは、指を触れた瞬間を等倍とする変化比率を返す、そのため足し算ではなく 掛け算lastScale * value.magnificationで前の状態を掛け合わせる必要があります。星のアイコンは線の内側や背景が透明です。透明な場所を指でつまんでもジェスチャーが反応しづらくなるのを防ぐため、透明な枠全体を当たり判定として定義しています。

**もしこう書かなかったら：**
1回拡大した後に一度指を離し、2回目のピンチ操作をした瞬間に、倍率が一旦元の大きさへガクッと戻ってしまったり、動きがギクシャクしたりします。contentShape(Rectangle())を書かなかったとき星の黄色い色がついている部分にピッタリ指を乗せてピンチしないと反応しなくなります。透明な背景部分をつまんでも拡大縮小できず、操作性が著しく悪化します。

---

### ジェスチャーの組み合わせとアニメーション

```swift
// 画像ビューに対して
.gesture(
    DragGesture()
        .onChanged { value in /* ドラッグ処理 */ }
        .onEnded { _ in /* 位置確定 */ }
)
// 複数のジェスチャーを「同時に」効かせるには .gesture を重ねるのではなく .simultaneousGesture を使う
.simultaneousGesture(
    MagnifyGesture()
        .onChanged { value in /* 拡大処理 */ }
        .onEnded { _ in /* 倍率確定 */ }
)
.simultaneousGesture(
    RotateGesture()
        .onChanged { value in /* 回転処理 */ }
        .onEnded { _ in /* 角度確定 */ }
)
Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
        scale = 1.0
        lastScale = 1.0
        angle = .zero
        lastAngle = .zero
    }
}
```

**何をしているか：**
ドラッグ、ピンチ、回転という3つの独立したジェスチャーを1つのビューに対して設定し、これらを同時に並行して判定・実行できるようにしています。リセットボタンが押されたときに、移動量、倍率、回転角のすべての状態変数を初期値（原点・1倍・0度）に戻す処理を withAnimation(.spring)で包んでいます。

**なぜこう書くのか：**
.simultaneousGestureを使うことで、他のジェスチャーが発生しても横取りせず、同時に認識して処理するという特別な命令を与えることができます。withAnimationブロック内でStateを変更することで、その変数の変更によって生じる画面上のレイアウト変化に対して、バネのように心地よく跳ねて収束するアニメーションを滑らかに割り込ませることができる。

**もしこう書かなかったら：**
もしwithAnimationを使わず、以下のようにそのまま変数を更新した場合は
```swift
// ❌ アニメーションなしの例
Button("リセット") {
    offset = .zero
    lastOffset = .zero
    // ...
}
```
ボタンを押した瞬間に、カードが一瞬で初期位置・初期状態に瞬間移動してしまいます。ユーザーから見ると画面の変化が唐突に感じられ、アプリ全体の自然な操作感が損なわれてしまいます。
---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `DragGesture` |ドラッグ操作を検知するジェスチャーオブジェクト| `.gesture(DragGesture().onChanged { ... })` |
| `MagnificationGesture` | ピンチジェスチャーで拡大・縮小を検知 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
|`.onTapGesture`|ビューがタップされたことを検知してクロージャ内の処理を実行するモディファイア|`.onTapGesture {tapCount += 1}`|
|`.simultaneousGesture()`|複数のジェスチャーが同時に発生した際、互いをブロックせずに並行して認識・処理させるためのモディファイア|`.gesture(DragGesture()).simultaneousGesture(MagnifyGesture())`|

## 自分の実験メモ

**実験1：**
- やったこと：拡大・縮小に制限をつける
- 結果：制限の上限まで到達するとそれ以上はいかないようになった。
- わかったこと：スマホの画面内に収めたい場合に有用である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
RotateGesture()について教えて
**得られた理解：**
小さいアイコンなどに直接適用すると、2本指を乗せた際にタッチ判定から外れやすくなる.frame(width: 300, height: 300) と .contentShape(Rectangle())を使って、透明なタッチ判定エリアをあらかじめ広げておくのが良い。

2. **質問：**
ドラッグ操作の詳細を教えて
   **得られた理解：**
DragGesture は、画面上の指の移動をリアルタイムに検知し、位置や移動量を扱えるAPI、@Stateで位置を記憶・維持 するか@GestureStateで指を離した際に自動復帰させるかで用途に応じた使い分けが可能です。

## この章のまとめ

操作の変位量を前回の確定値に蓄積し、指を離した後の位置・状態を正しく追従させる構造が重要である。またwithAnimation(.spring)や@GestureStateを組み合わせることで、リセット時や自動復帰時の動作を滑らかで直感的なUIになる
