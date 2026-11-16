# Fusion-Asteroids と Trial2026 Fusion Host Basics の比較

## 1. 比較対象

  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  項目                                   Fusion-Asteroids---Client-Host-2.1.3                                   Trial2026_fusion-host-basics-2.1.1
  -------------------------------------- ---------------------------------------------------------------------- --------------------------------------------------------------------
  GitHub                                 https://github.com/jshimura-DAS/Fusion-Asteroids---Client-Host-2.1.3   https://github.com/jshimura-DAS/Trial2026_fusion-host-basics-2.1.1

  Unity バージョン                       6000.3.28f1                                                            6000.3.13f1

  主な目的（コードから読み取れる範囲）   Asteroids ゲームのネットワーク実装                                     Fusion の Host Mode、入力共有、予測、物理、RPC の学習サンプル

  中心となる実装                         SpaceshipController、SpaceshipWeapon、AsteroidBehaviour、GameManager   BasicSpawner、Player、NetworkInputData、Ball、PhysxBall
                                         など                                                                   

  発射物の方式                           ProjectileData を NetworkArray で同期し、見た目はローカル GameObject   NetworkObject として Ball / PhysxBall を Runner.Spawn で生成
  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

本資料では、リポジトリ内の C# スクリプト、Prefab、Fusion 設定、Docs
を確認して比較しています。ゲームの実行結果や通信状況を実機で測定したものではありません。

## 2. 全体的な設計思想の違い

### Trial2026_fusion-host-basics-2.1.1

Fusion
の基本概念を個別に理解できるよう、機能を小さなサンプルに分けています。

-   `NetworkInputData` でプレイヤーの操作意図を送る
-   `InputAuthority` と `StateAuthority` の役割を確認する
-   `NetworkCharacterController` でプレイヤーを移動させる
-   `[Networked]` プロパティと `ChangeDetector` で状態変化を検知する
-   RPC でメッセージをホスト経由で全員に伝える
-   `Ball` と `PhysxBall` で、式による移動と PhysX による移動を比較する

### Fusion-Asteroids---Client-Host-2.1.3

Fusion の機能をゲームルールに結び付けています。

-   プレイヤーの入力・移動・生存状態を管理する
-   弾の発射情報をネットワークデータとして保持する
-   Lag Compensation の Raycast / OverlapSphere で命中や接触を判定する
-   アステロイドへの命中、破壊、スコア加算を行う
-   ゲーム進行、プレイヤーの参加・退出、リスポーンなどを扱う

**要約すると、Trial2026 は Fusion
の仕組みを学ぶためのサンプル、Fusion-Asteroids
は複数の仕組みを組み合わせたゲーム実装です。**

## 3. ネットワーク入力とプレイヤー処理

  | 観点 | Trial2026 | Fusion-Asteroids |
  | --- | --- | --- |
  | 入力の収集 | `BasicSpawner.OnInput()` | `PlayerInput` が `NetworkEvents.OnInput` に登録される |
  | 入力データ | `NetworkInputData`（方向ベクトル、マウスボタン） | `SpaceshipInput`（移動入力、ボタン） |
  | 入力の取得 | `Player.FixedUpdateNetwork()` の `GetInput()` | `SpaceshipController.FixedUpdateNetwork()` の `GetInput<SpaceshipInput>()` |
  | 移動 | `NetworkCharacterController.Move()` | `_velocity` を Networked 状態として保持し、Tick ごとに位置・回転を更新 |
  | 発射処理 | `Player.FixedUpdateNetwork()` 内でホストが `Runner.Spawn()` | `SpaceshipController` から `SpaceshipWeapon.Fire(input)` を呼ぶ |

Trial2026 の `BasicSpawner` は `INetworkRunnerCallbacks`
を実装し、`OnInput()` で WASD とマウスボタンを `NetworkInputData`
に詰めています。`Player` は `GetInput()`
でそのデータを取得し、移動と発射を処理します。

Fusion-Asteroids では、`SpaceshipController`
が入力を受け取り、移動・回転を更新したうえで、武器処理を
`SpaceshipWeapon`
に委譲します。プレイヤー制御と武器制御が別クラスに分かれている点も特徴です。

## 4. 発射物の同期方式：最も大きな違い

### 4.1 Trial2026：発射物そのものをネットワーク生成する

`Player.FixedUpdateNetwork()` は、State Authority がある場合に
`Runner.Spawn()` を呼びます。

-   左クリック：`Ball` を生成
-   右クリック：`PhysxBall` を生成

`Ball` と `PhysxBall` はそれぞれ `NetworkObject` を持つ Prefab
です。つまり、発射物自体がネットワーク上のオブジェクトとして生成・管理されます。

#### Ball：式で位置を更新する方式

`Ball.cs` は `FixedUpdateNetwork()` 内で次のように位置を更新します。

``` csharp
transform.position += Speed * transform.forward * Runner.DeltaTime;
```

Prefab は `NetworkObject`、`NetworkTransform`、`Ball`
を中心に構成されています。スクリプトは移動と `TickTimer`
による寿命管理を担当します。

#### PhysxBall：物理演算を利用する方式

`PhysxBall.cs` は `Init()` で `Rigidbody.linearVelocity`
を設定し、寿命を `TickTimer` で管理します。移動そのものは PhysX
が行います。

Prefab には
`NetworkObject`、`NetworkRigidbody`、`Rigidbody`、`SphereCollider`、`PhysxBall`
が配置されています。`NetworkRigidbody` は Fusion Physics
アドオンのコンポーネントです。

`BasicSpawner.StartGame()` では `RunnerSimulatePhysics`
を追加しています。また、`RunnerSimulatePhysics3D`
の追加例はコメントアウトされています。実際の物理シミュレーションの挙動は、アドオン設定と実行環境での確認が必要です。

### 4.2 Fusion-Asteroids：弾のデータを同期し、見た目はローカル生成する

`SpaceshipWeapon.cs`
のコメントには、弾はデータバッファとして同期し、見た目はローカル
GameObject として扱う旨が記載されています。

`ProjectileData` は `INetworkStruct` で、次の情報を持ちます。

-   `FireTick`：発射した Tick
-   `FirePosition`：発射位置
-   `FireVelocity`：弾の速度
-   `HitPosition`：命中位置

これらを容量 16 の `NetworkArray<ProjectileData>` に保存します。弾ごとの
`NetworkObject` を `Runner.Spawn()` する方式ではありません。

`Render()` では `_bulletDummyPrefab` をローカルに Instantiate
し、ネットワークデータから算出した位置に表示します。したがって、弾の見た目と実際の命中判定が分離されています。

### 4.3 方式の比較

  ---------------------------------------------------------------------------------------------
  項目                    Trial2026 の Ball /          Fusion-Asteroids の弾
                          PhysxBall                    
  ----------------------- ---------------------------- ----------------------------------------
  ネットワーク生成        `Runner.Spawn()`             発射物ごとの NetworkObject は生成しない
                          で発射物を生成               

  同期対象                発射物のネットワーク状態     発射
                                                       Tick・位置・速度・命中位置などのデータ

  移動                    Ball は式、PhysxBall は      Tick と速度から位置を計算
                          PhysX                        

  見た目                  ネットワークオブジェクトの   ローカルの Dummy GameObject
                          Transform / 補間             

  命中判定                サンプルコードでは Lag       `Runner.LagCompensation.Raycast()`
                          Compensation の Raycast      
                          は使用していない             

  寿命・終了              `TickTimer` の期限で         画面外で `FireTick` を無効化する処理など
                          `Runner.Despawn()`           
  ---------------------------------------------------------------------------------------------

この違いは、ネットワーク上で「何を同期するか」という設計の違いです。Trial2026
は発射物オブジェクトを同期する構成、Fusion-Asteroids
は弾の状態を小さなデータとして同期し、各ピアで見た目を再構成する構成です。

## 5. 物理衝突と Lag Compensation の違い

ここは混同しやすいポイントです。

### Trial2026：物理シミュレーションの学習

Trial2026 の `PhysxBall` は `Rigidbody` と `SphereCollider`
を持ち、Fusion Physics アドオンの `NetworkRigidbody` を利用します。Docs
では、ホストが物理結果を確定し、クライアント側の予測・再シミュレーションや補間について説明しています。

これは、物理演算の結果（位置・速度など）をネットワークで扱うための構成です。

### Fusion-Asteroids：過去の状態を考慮した当たり判定

Fusion-Asteroids の `SpaceshipWeapon.UpdateProjectile()` は、前 Tick
と現在 Tick の弾位置から判定線分を作り、次の API を呼びます。

``` csharp
Runner.LagCompensation.Raycast(
    previousPosition,
    direction,
    distance,
    Object.InputAuthority,
    out var hit,
    _hitMask,
    HitOptions.IgnoreInputAuthority
)
```

さらに `SpaceshipController.HasHitAsteroid()` では
`Runner.LagCompensation.OverlapSphere()`
を呼び、宇宙船とアステロイドの接触を判定します。

Fusion-Asteroids の `NetworkProjectConfig.fusion` では Lag Compensation
が有効で、次の値が設定されています。

  設定                                値
  ----------------------------- --------
  `LagCompensation.Enabled`       `true`
  `HitboxBufferLengthInMs`         `300`
  `HitboxDefaultCapacity`          `512`
  `CachedStaticCollidersSize`       `64`

また、`NetworkRunner.prefab` には `HitboxManager`
が配置されています。これらの構成とゲームコード内の Lag Compensation API
呼び出しが対応しています。

### 5.1 役割の違い

  ---------------------------------------------------------------------------------------------------------------------------------------------------
  比較項目                Fusion Physics / NetworkRigidbody                  Lag Compensation / HitboxManager
  ----------------------- -------------------------------------------------- ------------------------------------------------------------------------
  主な目的                物理シミュレーションの結果をネットワーク同期する   過去のネットワーク状態を考慮した命中・接触判定を行う

  主な対象                Rigidbody を持つ物理オブジェクト                   Fusion の Hitbox を使った判定対象

  代表的な構成・API       `NetworkRigidbody`、`RunnerSimulatePhysics`        `HitboxManager`、`Runner.LagCompensation.Raycast()`、`OverlapSphere()`

  この比較対象での例      Trial2026 の `PhysxBall`                           Fusion-Asteroids の弾と宇宙船の当たり判定
  ---------------------------------------------------------------------------------------------------------------------------------------------------

**NetworkRigidbody と HitboxManager
は同じ役割のコンポーネントではありません。**
前者は物理状態の同期、後者は Lag Compensation の判定管理に関係します。

Trial2026
のゲームスクリプトを検索した範囲では、`Runner.LagCompensation.Raycast()`、`OverlapSphere()`、`HitboxManager`
の明示的な利用は確認できませんでした。そのため、Trial2026 を Lag
Compensation の実装例として扱うことはできません。

## 6. 状態同期・イベント通知

Trial2026 の Docs では、状態を共有する `[Networked]`
プロパティと、出来事を通知する RPC の使い分けが詳しく説明されています。

### Trial2026 の例

-   `Player.SpawnedProjectile`：`[Networked]`
    プロパティ。`ChangeDetector`
    で変化を検知し、プレイヤーの色を変える。
-   `TickTimer`：発射間隔や Ball の寿命を Tick 単位で管理する。
-   `RPC_SendMessage()`：Input Authority から State Authority
    へメッセージを送り、`RPC_RelayMessage()` で全員へ中継する。

### Fusion-Asteroids の例

-   `SpaceshipController` の
    `IsAlive`、`Lives`、`Score`、`ShieldTimer`、`Nickname` などを
    `[Networked]` として保持する。
-   `SpaceshipWeapon` は
    `FireCount`、`_buttonsPrevious`、`_shootCooldown`、`_projectileData`
    をネットワーク状態として扱う。
-   ニックネーム設定などには RPC を使用する。

両方のプロジェクトで `[Networked]`、`TickTimer`、RPC
が使われていますが、Trial2026
は概念を説明する小さな実例、Fusion-Asteroids
はゲーム進行に必要な状態管理として使っています。

## 7. クラス構成の比較

  | 役割 | Trial2026 | Fusion-Asteroids |
  | --- | --- | --- |
  | Runner・起動・プレイヤー生成 | `BasicSpawner` | `GameManager`、`SpaceshipSpawner` など |
  | 入力取得 | `BasicSpawner.OnInput()` | `PlayerInput` |
  | プレイヤー制御 | `Player` | `SpaceshipController` |
  | 発射物 | `Ball`、`PhysxBall` | `SpaceshipWeapon` 内の `ProjectileData` とローカル弾表示 |
  | ゲーム対象 | サンプル用 Ball / PhysxBall | `AsteroidBehaviour`、`AsteroidSpawner` |
  | UI | `OnGUI` / TextMeshPro の簡易表示 | `GameUI`、`PlayerUI`、`StartMenu` |
  | ネットワーク物理 | `RunnerSimulatePhysics`、`NetworkRigidbody` | 今回比較した弾の命中処理は Lag Compensation API を利用 |

## 8. 学習の順番としての比較・関連付け

2つのリポジトリは、次の順番で見ると理解しやすいです。

1.  **Trial2026 / `BasicSpawner.cs`**\
    Runner の起動、Host / Client の開始、`OnInput()`
    による入力収集を理解する。

2.  **Trial2026 / `Player.cs` と `NetworkInputData.cs`**\
    `GetInput()`、`InputAuthority`、`StateAuthority`、`FixedUpdateNetwork()`
    の関係を理解する。

3.  **Trial2026 / `Ball.cs` と `PhysxBall.cs`**\
    ネットワークオブジェクトとして発射物を生成する方法と、`NetworkTransform`
    / `NetworkRigidbody` の違いを理解する。

4.  **Trial2026 / Docs**\
    Tick、予測、再シミュレーション、`[Networked]`、ChangeDetector、RPC
    の考え方を整理する。

5.  **Fusion-Asteroids / `SpaceshipController.cs`**\
    入力を受け取り、移動・接触判定・武器処理を組み合わせる構造を確認する。

6.  **Fusion-Asteroids / `SpaceshipWeapon.cs`**\
    NetworkArray に弾データを保存し、Tick 間の移動線分を
    `Runner.LagCompensation.Raycast()` に渡す仕組みを追う。

7.  **Fusion-Asteroids / `NetworkRunner.prefab` と
    `NetworkProjectConfig.fusion`**\
    HitboxManager と Lag Compensation の設定がゲームコードの判定 API
    を支えることを確認する。

## 9. 結論

-   Trial2026 は、Fusion の Host Mode
    における入力、権限、状態同期、RPC、予測、物理同期を学ぶための基礎サンプルです。
-   Fusion-Asteroids は、それらの基本概念をゲーム全体に適用し、さらに
    Lag Compensation を使った命中・接触判定を組み込んだ応用例です。
-   発射物の扱いは大きく異なります。Trial2026 は `Runner.Spawn()` で
    NetworkObject を生成しますが、Fusion-Asteroids は弾の情報を
    NetworkArray で同期し、見た目をローカル生成します。
-   Trial2026 の `NetworkRigidbody`
    は物理状態を同期する仕組みです。一方、Fusion-Asteroids の
    `HitboxManager` と `Runner.LagCompensation.Raycast()` は Lag
    Compensation の当たり判定に関係します。
-   したがって、**Trial2026 で Fusion
    の基本的なネットワークシミュレーションを理解し、その知識を使って
    Fusion-Asteroids のデータ同期方式と Lag Compensation
    の実装を読む**という順番が有効です。

## 10. 参照した主なファイル

### Trial2026_fusion-host-basics-2.1.1

-   `Assets/Trial_fusion-host-basics-2.1.1/Scripts/BasicSpawner.cs`
-   `Assets/Trial_fusion-host-basics-2.1.1/Scripts/Player.cs`
-   `Assets/Trial_fusion-host-basics-2.1.1/Scripts/NetworkInputData.cs`
-   `Assets/Trial_fusion-host-basics-2.1.1/Scripts/Ball.cs`
-   `Assets/Trial_fusion-host-basics-2.1.1/Scripts/PhysxBall.cs`
-   `Assets/Trial_fusion-host-basics-2.1.1/Prefab/Ball.prefab`
-   `Assets/Trial_fusion-host-basics-2.1.1/Prefab/PhysxBall.prefab`
-   `Assets/Trial_fusion-host-basics-2.1.1/Prefab/PlayerPrefab.prefab`
-   `Docs/# ホストモードにおける入力共有と予測.md`
-   `Docs/ホストモードにおける入力共有と予測（続）Ball・物理・NetworkInputData.md`
-   `Docs/プロパティ変更とRPC.md`

### Fusion-Asteroids---Client-Host-2.1.3

-   `Assets/Scripts/Player/SpaceshipController.cs`
-   `Assets/Scripts/Player/SpaceshipWeapon.cs`
-   `Assets/Prefabs/NetworkRunner.prefab`
-   `Assets/Photon/Fusion/Resources/NetworkProjectConfig.fusion`
-   `Assets/Scripts/GameManager.cs`
