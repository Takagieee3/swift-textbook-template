# 第7章：センサーの活用

> 執筆者：高木　宏輔
> 最終更新：2026-07-30

## この章で学ぶこと

センサー連携である姿勢データのリアルタイムで取得、状態管理のビューとデータバインディングを学習し図形描画である円や直接を描くこれらを組み合わせた水平器の画面UIについて学習する。

## 模範コードの全体像

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

スマホが前後左右にどの向きでどのくらい傾くのかを測定し連動して円と気泡が表示され見た時にわかりやすい。また平らに近づくと緑色に変化する

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
init() {
    isAvailable = motionManager.isDeviceMotionAvailable
}

func startUpdates() {
    guard isAvailable else { return }
    // ...
}
motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self, let motion = motion else { return }

    self.pitch = motion.attitude.pitch
    self.roll = motion.attitude.roll
    self.yaw = motion.attitude.yaw
}
```

**何をしているか：**
センサーからのデータ受け取り頻度を1秒間に60回に指定しています。センサーの監視を開始し、データが届くたびにクロージャ内の処理を実行します。to: .mainを指定することで、この通知をメインキューで受け取り、プロパティを更新しています。

**なぜこう書くのか：**
SwiftUIの画面更新は必ずメインスレッドで行う必要があるというiOSの標準ルールを守るためです。画面の描画フレームレートにセンサーの更新頻度をぴったり合わせることで、端末を傾けたときの気泡の動きを遅れなく滑らかに見せるためです。

**もしこう書かなかったら：**
画面内の気泡の動きがカクカクしたり、傾けてから画面が反応するまでに目に見えるタイムラグが発生してしまいます。ビューが破棄されても MotionManager のインスタンスがメモリに残り続け、メモリリークが発生します。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
// MotionManager 内
var pitch: Double = 0    // 前後の傾き
var roll: Double = 0     // 左右の傾き
var yaw: Double = 0      // 水平方向の回転

// startUpdates() 内
self.pitch = motion.attitude.pitch
self.roll = motion.attitude.roll
self.yaw = motion.attitude.yaw
// LevelIndicator 内
private let maxOffset: CGFloat = 100

private var xOffset: CGFloat {
    CGFloat(roll) * maxOffset
}

private var yOffset: CGFloat {
    CGFloat(pitch) * maxOffset
}
// LevelIndicator 内
private var isLevel: Bool {
    abs(pitch) < 0.03 && abs(roll) < 0.03
}
```

**何をしているか：**
前後・左右の傾きの絶対値がともに0.03ラジアン未満であるかを判定し、色変更や「水平!」テキスト表示の条件にしています。

**なぜこう書くのか：**
角度に係数100を掛けてピクセル単位の移動量に換算し、さらに min/max で円の外に気泡が飛び出さないよう範囲制限を行っています。

**もしこう書かなかったら：**
センサーの値がクラス内に反映されず、どれだけ端末を傾けても画面上の気泡や数値が初期値（0）のまま一切動きません。気泡が1.5ピクセル程度しか動かず、見た目上ほとんど変化しなくなります。

---

### 歩数計（CMPedometer）

```swift
guard CMPedometer.isStepCountingAvailable() else { return }
pedometer.startUpdates(from: Date()) { pedometerData, error in
    guard let data = pedometerData, error == nil else { return }
    
    Task { @MainActor in
        self.stepCount = data.numberOfSteps.intValue
    }
}
```

**何をしているか：**
端末に歩数計測機能が備わっているかを確認しています。from: Date() で今からの歩数リアルタイム計測をスタートし、データが更新されるたびにメインスレッドで stepCount プロパティに保存しています。

**なぜこう書くのか：**
iPadや一部の旧型端末など、歩数計測に対応していないデバイスで処理が進むのを防ぐためです。歩数がカウントされるたびにクロージャが呼ばれるため、歩数カウントをリアルタイムに書き換えることができます。画面更新を行うため Task { @MainActor in } でメインスレッド処理を保証しています。

**もしこう書かなかったら：**
非対応端末で計測を開始しようとした際に、アプリがクラッシュしたり無駄なエラーが発生します。歩数が画面にリアルタイム反映されなかったり、バックグラウンドスレッドからの画面更新になってXcodeの警告や表示の不具合が発生します。

---

### CoreLocationとの連携

```swift
locationManager.delegate = self
locationManager.startUpdatingHeading()
func locationManager(_ manager: CLLocationManager, didUpdateHeading newHeading: CLHeading) {
    Task { @MainActor in
        self.heading = newHeading.magneticHeading // 磁北（度）を取得
    }
}
```

**何をしているか：**
iPhone内蔵の電子コンパスを起動し、端末が北から何度向いているかのリアルタイム更新を開始します。コンパスから届いた角度を受け取り、メインスレッド@MainActorでプロパティに代入して画面を再描画させています。

**なぜこう書くのか：**
startUpdatingHeading()を呼ぶことで、端末の向きが変わるたびにデリゲートメソッドへ新しい角度データが自動で送られてくるようになるためです。CoreLocationのデリゲート通知はサブスレッドで呼ばれることがあるため、画面更新を行うために Task { @MainActor in } を使って安全にメインスレッドへ切り替える必要があります。

**もしこう書かなかったら：**
方位センサーが起動せず、コンパス針の回転や「北から◯度」といった方角のリアルタイム表示ができません。バックグラウンドスレッドから直接UIを更新することになり、Xcodeで警告が出たり、画面の描画が不安定になったりクラッシュする原因になります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| `CMAttitude` | 3次元空間におけるデバイスの傾きを表すオブジェクト | `if let motion = motionManager.deviceMotion {let pitch = motion.attitude.pitch}` |
|`.animation(_:value:)`|指定した値が変化したときに、対象のビューへアニメーションを自動適用するSwiftUIモディファイア|`Circle().offset(x: xOffset, y: yOffset).animation(.spring(duration: 0.1), value: xOffset)`|
|`ContentUnavailableView`|データが存在しない場合や、シミュレータのようにセンサー機能が使えない場合のエラー・空状態画面を簡単に作成できます。|`ContentUnavailableView("センサー不可",systemImage: "iphone",description: Text("実機で実行してください"))`|

## 自分の実験メモ

**実験1：**
- やったこと：センサーの更新頻度を上げる
- 結果：動きがより滑らかになった。
- わかったこと：更新頻度を1秒に何回を指定でき見やすいぐらいがちょうどいい。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
センサーのデータ受け取り頻度に上限はありますか
   **得られた理解：**
一般的に設定できる上限は100Hz程度であり滑らかさや電力の兼ね合いでバランスが良いのは60Hzが良い。
2. **質問：**
スマホの傾きの精度はどのくらいですか

   **得られた理解：**
普段使いの壁に額縁をまっすぐ掛ける家具の傾きを見るといった用途であれば、市販のアナログ水準器と比べても遜色ないレベルの精度を持っています。

## この章のまとめ

CoreMotionによるリアルタイム姿勢取得であるCMMotionManagerを使って端末の傾きpitch,roll,yawを60fpsで滑らかに取得する手法を学びました。UIとの連携の仕方を学び自動反映させリアルタイムで気泡の位置、角度がわかるようになった。
